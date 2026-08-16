---
title: "Layer Keys and Hashing"
description: "How a command's verbs, filters, and context become the cache keys of the layer forest"
weight: 145
---

Every node in the [layer forest](/docs/design/shards) is
content-addressed: its cache key is a hash, and a cache hit means "this node
already exists". This page explains how those keys are built, bottom-up. One
rule governs everything on this page:

> **Whatever a populate *reads*, its key must *name* — and nothing more.**

Name too little and two different results collide on one key (a correctness
bug). Name too much and identical results get distinct keys (a cache
fragmented for no reason). Each subsection below adds one ingredient.

## 1. The filter predicate F {#filter-predicate}

Start with a single command. Besides its content verbs it
carries **filters** — the units of filter composition: a name pattern, a
type constraint, `project("linux")`, `ignore(...)`. (A *verb* is the
generic execution unit — see
[Queries and their Meaning](/docs/design/semantics); most filters
arrive as verbs, but not all — a bare name string is a filter without
being a verb.) For keying and
evaluation the filters are combined into **one predicate** \(F_c\) — the
conjunction of all of them — represented as a tree: predicate leaves,
combined by And/Or/Not nodes.

Two details, each worth its own sentence:

- **Inheritance.** "All of them" means all filters *on the
  command* — which includes filters inherited from enclosing
  substatements (inheritable filters are copied into child commands at
  parse time). A nested command's \(F_c\) therefore already reflects its
  ancestors' constraints; nothing later needs to walk the enclosing
  substatements.
- **Uniformity.** \(F_c\) applies to every content verb of the command
  alike — physically at read time, where every read of the command's layers
  conjoins the same \(F_c\) over their rows; a populate consults at most
  \(F_c\)'s object-narrowing part to restrict what it scans
  ([semantics §5](/docs/design/semantics/#content-map)).

## 2. The filter hash {#filter-hash}

\(\mathcal{H}(F_c)\) is computed recursively over the predicate tree:

- each **leaf** hashes its own semantics: which predicate it is, and its
  arguments in canonical encoding;
- each **inner node** hashes an operator tag (And, Or, Not) followed by its
  children's hashes.

Two filters hash equally exactly when they are the same tree
([search()](/docs/design/search/#cache-key-composition) gives the
byte layout). This is the *only* filter-awareness mechanism in the whole
cache — no filter type is special-cased anywhere.

## 3. Per-verb input hashes {#verb-input-hashes}

Each content verb \(i\) of command \(c\) gets its own hash:

$$H(c,i) \;=\; \mathcal{H}\bigl(\, \mathrm{dom}(i) \,\Vert\, \mathrm{inputs}(i) \,\Vert\, \mathcal{H}(F_c) \,\bigr)$$

- \(\mathrm{dom}(i)\) — the verb discriminator (`"search"`, `"loc"`, …): a
  domain-separation tag, so different verbs with coincidentally equal
  argument bytes cannot collide;
- \(\mathrm{inputs}(i)\) — the verb's own arguments, canonically encoded and
  length-prefixed (for `search`: query bytes, case flag, whole-word flag,
  limit);
- \(\mathcal{H}(F_c)\) — present exactly when the populate *reads*
  \(F_c\), per the governing rule. `search` reads it: \(F_c\)'s object-level
  part narrows the scan's input corpus (a `project(...)` decides which
  projects' content is scanned at all), so the key must name it — and it
  names the whole tree via the shared §2 hash rather than extracting the
  object part, since that part is derived from the full tree. A verb
  whose populate reads nothing of \(F_c\) folds no \(\mathcal{H}(F_c)\):
  `loc`'s path and `project=` arguments already fix what it reads, and
  \(F_c\) reaches its rows only at read time.

## 4. Combining verbs: the command hash {#command-hash}

A command may carry several content verbs (`search("a") search("b")`),
all contributing to one node group. They are required to be **mutually
independent** — each is a self-contained populate, none reads another's
output, results combine by plain union. This is a keying requirement: no
\(H(c,i)\) folds any \(H(c,j)\), so a cross-verb dependency would mean a key
that fails to name one of its inputs.

The command hash is then \(H(c) = H(c,1)\) for a single verb (its key is
unchanged by the composition machinery — single-verb commands stay
cache-warm). For several verbs the per-verb hashes fold in **source
order**:

$$H(c) \;=\; \mathcal{H}\bigl(\, \mathrm{dom}_{\mathrm{composite}} \,\Vert\, H(c,1) \,\Vert\, \dots \,\Vert\, H(c,n) \,\bigr)$$

Each part is a fixed 32 bytes, so the concatenation needs no delimiters.
Source order does mean `search("a") search("b")` and its reverse key
different layers despite denoting the same union — harmless for correctness
(key soundness only needs same-key \(\Rightarrow\) same-content), at worst
one redundant materialisation.

## 5. Scope fusion {#scope-fusion}

\(F_c\) is not the only command context a verb may fold. A verb that
restricts its populate to the enclosing container — `search` narrowing its
scan to the parent substatement's byte ranges — reads that container's
condition, so by the governing rule its \(H(c,i)\) must name it: the fused
scope's condition (or its resolved instance ids) joins the verb's inputs.
A verb that does not fuse (e.g. `loc`, single-file by construction) folds
nothing extra.

## 6. Acyclicity {#acyclicity}

The definitions may look mutually recursive — \(H(c,i)\) folds
\(\mathcal{H}(F_c)\), and \(F_c\) is built from the same command.
There is no cycle because the two roles are **disjoint**: \(F_c\) is
assembled only from *filters*, and content-producing verbs
contribute nothing to it. The hash flow is a strictly layered DAG:

```mermaid
graph TD
    L1["filter leaf<br/>project(&quot;linux&quot;)"] --> F["ℋ(F_c)<br/>filter-tree hash"]
    L2["filter leaf<br/>type = func"] --> F
    F --> H1["H(c,1)<br/>search(&quot;a&quot;)"]
    F --> H2["H(c,2)<br/>search(&quot;b&quot;)"]
    I1["inputs: &quot;a&quot;, case,<br/>whole-word, limit"] --> H1
    I2["inputs: &quot;b&quot;, case,<br/>whole-word, limit"] --> H2
    PS["container scope<br/>(fused, when present)"] -.-> H1
    PS -.-> H2
    H1 --> HT["H(c)<br/>composite-input-v1 ‖ H(c,1) ‖ H(c,2)"]
    H2 --> HT
    HR["h(R)<br/>root identity"] --> K["κ_root<br/>root-shard key"]
    HT --> K

    classDef filt fill:#f6efe2,stroke:#c9a35a;
    classDef verb fill:#e8f1fb,stroke:#4a90d9;
    classDef key fill:#efe8f7,stroke:#8e6bbf,stroke-width:2px;
    class L1,L2,F filt;
    class I1,I2,H1,H2,HT,PS verb;
    class HR,K key;
```

Filter leaves at the bottom; the filter hash above them; per-verb hashes
above that, each also folding its own arguments (and a fused scope when
present); the command hash at the top.

## 7. Node keys {#node-keys}

The command hash \(H(c)\) is the "command inputs" ingredient of every node
key in the [shard partition](/docs/design/shards), but only the root
shard folds it directly. The root shard folds \((h(R), H(c))\),
yielding \(\kappa_{\mathrm{root}}\); a layer shard then folds
\((\mathrm{id}(\ell), \kappa_{\mathrm{root}})\) and the selection shard
folds \((\mathrm{id}(\mathrm{parent}), \kappa_{\mathrm{root}},
\mathrm{extra})\) — parent *ids*, not parent keys. Each is taken under
its own domain tag — root shard `root-shard-v1`, layer shard
`layer-shard-v1`, selection shard `selection-shard-v1` — so the three
key families are disjoint even over equal payloads, and the two
non-root families inherit the root shard's project scoping through
\(\kappa_{\mathrm{root}}\). The composite selection shard's `extra`
folds each part's extra, length-prefixed, in source order.

\(\mathrm{extra}\) is exactly the governing rule applied to a node that reads
**more than the one input its parent names** (the code's `selection_extra`). An
input shard needs none: its contents are a function of the command's inputs and
the rows of the single layer it is over. A `layer { … }` block's ops, by
contrast, may name specific ephemeral ids from earlier statements, so those
resolved ids join the key — and a block cannot collide with one that shares
everything else but its references. For content populates (`search`, `loc`)
\(\mathrm{extra}\) is empty.

Those three key shapes and the ingredients above are the complete key
system; [Partitioning a Materialisation](/docs/design/shards) shows
what it buys — losslessness, trustworthy names, and cross-query
reuse.
