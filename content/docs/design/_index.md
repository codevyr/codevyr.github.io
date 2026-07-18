---
title: "Design"
description: "Architecture and correctness proofs behind the askl query engine and its verbs"
weight: 300
---

The pages under this section are long-form technical documents about *how* askl works under the hood — algorithms, data models, correctness invariants, and the design trade-offs that shape them.

They're aimed at contributors and at users who want a full picture of what happens when they run a query. The [Askl Syntax Reference](/docs/syntax) is the right first stop for using the language; these pages explain the machinery beneath it.

## Pages

- **[Execution Engine](/docs/design/execution-engine)** — how the query engine evaluates a nested statement tree using a monotone worklist propagation loop.
- **[Design: search()](/docs/design/search)** — how the `search()` verb turns a literal-string query into byte-range matches over the whole indexed corpus, and how the ephemeral-layer cache keeps repeat queries cheap while staying correct across filter compositions.
