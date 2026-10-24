---
title: "HTTP API"
description: "The askld HTTP API — running queries, the response envelope, result limits, and the pagination endpoints that fetch what the caps withheld"
weight: 210
---

`askld` serves one query endpoint and three pagination endpoints. Everything a
client needs is JSON over plain HTTP; the examples below run against a
[local deployment](/docs/local-deployment), where the API listens on
`localhost:3002`.

## POST /query {#query}

Runs an Askl query. The request body is the query text, sent as
`text/plain`:

```sh
curl -X POST 'http://localhost:3002/query' \
  --data-binary 'mod("amdgpu") { func { "drm_dev_enter" } }'
```

### Query parameters {#query-parameters}

| Parameter | Default | Meaning |
|---|---|---|
| `limit` | server `--max-result-symbols` (100) | Max distinct symbols **per command**. Every command gets this budget for itself — a dedicated selector's results are never squeezed out by another command's fan-out. Overridden per command by [`limits(symbols=)`](/docs/syntax#result-limits). `0` = unlimited. |
| `detail` | `full` | `skeleton` returns the graph's structure only — nodes, edges, containment — and leaves instances and call sites to the pagination endpoints; `full` inlines them. The UI asks for `skeleton`; one-shot consumers such as the MCP server take `full`. Any other value is a 400. |
| `projects` | all indexed projects | Comma-separated project names to make visible; an unknown name is a 400. See [narrowing the request](/docs/syntax#project-project-filter). |
| `format` | `json` | `json` or `markdown` (a rendered report instead of the graph). |
| `projection` | `signature` | Markdown only: `names`, `signature`, or `body` — how much source rides along. |
| `listing` | server `--listing-cap` (100) | Max rows in any single per-node listing: instances per node and distinct out-edges per node. `0` = uncapped. Overridden per command by the [`limits()`](/docs/syntax#result-limits) verb. |

`listing` is an *assembly* cap: it bounds what one response carries, never
which symbols the query finds. Anything it trims is reported as truncated and
can be fetched with the [references](#references) and [instances](#instances)
endpoints. Setting `limit` and `listing` to `0` restores the full payload —
at the risk of hitting the 1 MB response ceiling on hub-heavy queries.

### Response envelope, `detail=full` {#response}

```text
{
  "nodes": [
    {
      "id": "12890223412",            // symbol id, string-encoded
      "label": "amdgpu",
      "symbol_instances": [ ... ],     // up to `listing` entries
      "query_statements": [ ... ],     // which statements selected it
      "instances_truncated": true,     // instance list was capped
      "total_instances": 800           // exact count before capping
    }, ...
  ],
  "edges": [                           // one entry per (from, to) pair
    {
      "from": "12890223412",
      "to": "12885652740",
      "occurrence_count": 4,           // call sites of this pair in the result
      "from_objects": ["206420"]       // files the call sites live in
    }, ...
  ],
  "has_edges": [ ... ],                // containment, one container per child
  "objects": [ ... ],                  // only files the payload references
  "warnings": [ ... ],                 // e.g. "this part of the graph is
                                       // incomplete" (see below)
  "truncated": true,                   // some command hit its `limit`
  "total_symbols": 906,                // distinct symbols before capping
  "edges_truncated": ["12890223412"]   // node ids whose out-edge listing was
                                       // capped — fetch the rest per node
}
```

Field notes:

- `truncated` / `total_symbols` — the symbol `limit` applies per
  command; `truncated` is true when **any** command overflowed its
  budget. Within a command, the kept symbols are chosen
  deterministically by `(path, offset, id)`.
- `edges` — an edge is a relationship between two *symbols*; its call
  sites are detail nobody reads until the edge is opened, so `/query`
  ships one entry per pair with a count, and the
  [edge occurrences endpoint](#edge-occurrences) pages the sites.
- `edges_truncated` — symbols whose *distinct out-edge* listing was
  capped by `listing`. The [references endpoint](#references) pages every
  edge. Call sites are never capped.
- `instances_truncated` / `total_instances` — per node. The kept
  instances are ordered so real code instances (definitions,
  declarations) come before directory-like ones (sentinel, containment);
  a truncated list never misrepresents what kind of symbol a node is.
  The [instances endpoint](#instances) pages the rest.
- `warnings` — two kinds. A statement whose *discovery* read hit the
  safety cap carries `too many results: this part of the graph is
  incomplete`: the symbol set itself may be missing entries, and the fix
  is a narrower query, not pagination. A statement whose leaf read
  stopped at the result budget instead (eight times its symbol limit)
  carries `results were bounded to N rows by the server result budget and
  may be incomplete — narrow the query or raise the request limit`. See
  [Result limits and truncation](/docs/syntax#result-limits).

### The skeleton, `detail=skeleton` {#skeleton}

The graph a client lays out is which symbols are in the result and how they
relate; everything else is detail it fetches when a node or edge is opened.
The skeleton ships exactly that:

```text
{
  "nodes": [
    {
      "id": ...,                 // one node per containment context (below)
      "symbol_ids": [ ... ],     // the raw symbol id(s), for the detail endpoints
      "label": ...,
      "symbol_type": ...,        // "function", "module", ...
      "is_directory": ...,       // from the symbol type: module, directory, file
      "instance_count": ...,     // instances in THIS result — not a total
      "parent_symbol": ...,      // the containment context, or null
      "query_statements": [ ... ]
    }, ...
  ],
  "edges": [ { "from", "to", "from_symbol", "to_symbol", "occurrence_count" }, ... ],
  "has_edges": [ { "parent", "child" }, ... ],
  "warnings": [ ... ], "truncated": ..., "total_symbols": ..., "edges_truncated": [ ... ]
}
```

- There are no instances, no `objects` map and no `total_instances`: the
  [instances endpoint](#instances) returns the exact total when a node is
  opened, and every detail row carries its own path. On
  `mod("amdgpu") { derive(type="has") func }` this took the response
  from 160 KB to 50 KB.
- A symbol under several containers renders as one node per containment
  context. The server does that split: such node ids are
  `<symbol>\0<context>`, `symbol_ids` keeps the raw symbol, and
  `parent_symbol` names the context — pass it as `?parent=` to page that
  node's instances alone. Containment edges are rebuilt between the
  splits; reference edges route to the split whose files hold their call
  sites.

## GET /v1/symbol/{id}/references {#references}

Pages the references **out of** a symbol — what a node listed in
`edges_truncated` calls for. Rows come in the engine's own
`(from_instance, ref)` unit, so fetched rows are exactly the edges the
cap withheld.

```sh
curl 'http://localhost:3002/v1/symbol/12890223412/references?limit=2'
```

```text
{
  "kind": "refs",
  "rows": [
    {
      "to_symbol": "12885652740",
      "to_symbol_name": "cgs_device",
      "to_instance_id": "28021338",
      "from_instance_id": "28021318",
      "object_id": "206420",
      "path": "/linux/drivers/gpu/drm/amd/acp/acp_hw.c",
      "project_id": "linux",
      "start_offset": 1399,
      "end_offset": 1409
    }, ...
  ],
  "next": "28021318:43631875",
  "count": 1000,
  "count_capped": true
}
```

| Parameter | Default | Meaning |
|---|---|---|
| `kind` | `refs` | `refs` — references out of the symbol; `has` — instances it contains. |
| `after` | — | The previous page's `next` value, passed back verbatim (an opaque keyset cursor). |
| `limit` | 50 | Rows per page (max 500). |
| `projects` | all | Same narrowing as `/query`. |

- `next` is `null` when the listing is exhausted. Cursors are stable:
  pages are disjoint and every row appears exactly once.
- `count` is **capped** at 1000: `count_capped: true` reads as
  "at least `count`". An exact out-edge total for a hub node would cost
  an unbounded scan, so it is never computed.
- References written inside a macro's body are included like any other;
  a macro use is itself a reference, not an instance.
- A page may occasionally carry fewer than `limit` rows: a
  self-reference (the referenced symbol's only instance *is* the caller)
  is not an edge the engine draws, so it is dropped from the rows while
  still advancing the cursor.

## GET /v1/symbol/{id}/instances {#instances}

Pages a symbol's instances — what a node with `instances_truncated`
calls for.

```sh
curl 'http://localhost:3002/v1/symbol/12890223412/instances?limit=5'
```

```text
{
  "rows": [
    {
      "id": "28021318",
      "instance_type": "containment",
      "object_id": "206420",
      "path": "/linux/drivers/gpu/drm/amd/acp/acp_hw.c",
      "project_id": "linux",
      "start_offset": 0,
      "end_offset": 1724
    }, ...
  ],
  "next": "206420:1724:28021318",
  "total": 931
}
```

| Parameter | Default | Meaning |
|---|---|---|
| `after` | — | The previous page's `next`, passed back verbatim (an opaque `object:offset:id` cursor). |
| `limit` | 50 | Rows per page (max 500). |
| `parent` | — | A containing symbol's id: list only the instances inside it — the node a skeleton split under that `parent_symbol`. Absent, every instance of the symbol. |
| `projects` | all | Same narrowing as `/query`. |

- `total` is **exact** — unlike edge counts, a per-symbol instance count
  is bounded by the symbol's own size.
- Pages come in display order — `(file, offset, id)`, the order the UI
  shows — so each page appends where the previous one ended. With
  `parent`, the pages of all contexts together are the symbol's whole
  instance set, with no overlap.

## GET /v1/edge/{from}/{to}/occurrences {#edge-occurrences}

Pages one edge's call sites — what the UI opens for an edge whose
`occurrence_count` is above one.

```sh
curl 'http://localhost:3002/v1/edge/12890223412/12885652740/occurrences?limit=2'
```

```text
{
  "rows": [
    {
      "ref_id": "43631875",
      "from_instance_id": "28021318",
      "object_id": "206420",
      "path": "/linux/drivers/gpu/drm/amd/acp/acp_hw.c",
      "project_id": "linux",
      "start_offset": 1399,
      "end_offset": 1409
    }, ...
  ],
  "next": "206420:1399:43631875"
}
```

| Parameter | Default | Meaning |
|---|---|---|
| `after` | — | The previous page's `next`, passed back verbatim. |
| `limit` | 50 | Rows per page (max 500). |
| `projects` | all | Same narrowing as `/query`. |

- Rows come in display order `(file, offset, ref id)`, so pages append
  contiguously. There is no count: the edge's `occurrence_count` in the
  `/query` response already is one.

## Server configuration {#configuration}

Each cap has a CLI flag and an environment variable; per-request query
parameters override the server default. Set the env vars in
`compose.local.yaml` the same way as `ASKL_MAX_UPLOAD_BYTES` (see
[Local Deployment](/docs/local-deployment)).

| Flag | Env | Default | Bounds |
|---|---|---|---|
| `--max-result-symbols` | `ASKL_MAX_RESULT_SYMBOLS` | 100 | distinct symbols per command |
| `--listing-cap` | `ASKL_LISTING_CAP` | 100 | rows per per-node listing (instances, distinct out-edges) |
| `--probe-cap` | `ASKL_PROBE_CAP` | 1000 | discovery-read safety cap (see [Planning](/docs/design/planning#configuration)) |

`0` disables the corresponding cap. Per command, the
[`limits(symbols=N, listing=M)`](/docs/syntax#result-limits) verb overrides
these defaults and inherits into nested scopes. The design rationale — why
discovery caps warn while assembly caps paginate — is in
[Bounded Results](/docs/design/bounded-results).
