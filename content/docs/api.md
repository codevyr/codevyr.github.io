---
title: "HTTP API"
description: "The askld HTTP API — running queries, the response envelope, result limits, and the pagination endpoints that fetch what the caps withheld"
weight: 210
---

`askld` serves one query endpoint and two pagination endpoints. Everything a
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
| `projects` | all indexed projects | Comma-separated project names to make visible; an unknown name is a 400. See [narrowing the request](/docs/syntax#project-project-filter). |
| `format` | `json` | `json` or `markdown` (a rendered report instead of the graph). |
| `projection` | `signature` | Markdown only: `names`, `signature`, or `body` — how much source rides along. |
| `listing` | server `--listing-cap` (100) | Max rows in any single per-node listing: instances per node, distinct out-edges per node, and call-sites per `(from, to)` pair — all bounded by this one number. `0` = uncapped. Overridden per command by the [`limits()`](/docs/syntax#result-limits) verb. |

`listing` is an *assembly* cap: it bounds what one response carries, never
which symbols the query finds. Anything it trims is reported as truncated and
can be fetched with the [references](#references) and [instances](#instances)
endpoints. Setting `limit` and `listing` to `0` restores the full payload —
at the risk of hitting the 1 MB response ceiling on hub-heavy queries.

### Response envelope {#response}

```text
{
  "nodes": [
    {
      "id": "12890223412",            // symbol id, string-encoded
      "label": "amdgpu",
      "symbol_instances": [ ... ],     // up to instance_cap entries
      "query_statements": [ ... ],     // which statements selected it
      "instances_truncated": true,     // instance list was capped
      "total_instances": 800           // exact count before capping
    }, ...
  ],
  "edges": [ ... ],                    // one entry per call site, capped
                                       // per (from, to) pair
  "has_edges": [ ... ],                // containment, one container per child
  "objects": [ ... ],                  // only files the payload references
  "warnings": [ ... ],                 // e.g. "this part of the graph is
                                       // incomplete" (see below)
  "truncated": true,                   // some statement hit its `limit`
  "total_symbols": 906,                // distinct symbols before capping
  "edges_truncated": ["12890223412"]   // node ids whose edge listing was
                                       // capped — fetch the rest per node
}
```

Field notes:

- `truncated` / `total_symbols` — the symbol `limit` applies per
  command; `truncated` is true when **any** command overflowed its
  budget. Within a command, the kept symbols are chosen
  deterministically by `(path, offset, id)`.
- `edges_truncated` — symbols whose out-edge listing was capped by
  `listing` (distinct edges, or call sites per pair). The
  [references endpoint](#references) pages every occurrence.
- `instances_truncated` / `total_instances` — per node. The kept
  instances are ordered so real code instances (definitions,
  declarations) come before directory-like ones (sentinel, containment);
  a truncated list never misrepresents what kind of symbol a node is.
  The [instances endpoint](#instances) pages the rest.
- `warnings` — a statement whose *discovery* read hit the safety cap
  carries `too many results: this part of the graph is incomplete`.
  That warning is different in kind from the assembly caps above: it
  means the symbol set itself may be missing entries, and the fix is a
  narrower query, not pagination. See
  [Result limits and truncation](/docs/syntax#result-limits).

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
- References made from macro expansion sites are included — an expansion
  is an ordinary instance of its symbol.
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
  "next": 28055528,
  "total": 931
}
```

| Parameter | Default | Meaning |
|---|---|---|
| `after` | — | The previous page's `next` (an instance id). |
| `limit` | 50 | Rows per page (max 500). |
| `projects` | all | Same narrowing as `/query`. |

- `total` is **exact** — unlike edge counts, a per-symbol instance count
  is bounded by the symbol's own size.
- Pages are ordered by instance id (stable and duplicate-free). Display
  ordering is the client's concern; the UI re-sorts by path and offset.

## Server configuration {#configuration}

Each cap has a CLI flag and an environment variable; per-request query
parameters override the server default. Set the env vars in
`compose.local.yaml` the same way as `ASKL_MAX_UPLOAD_BYTES` (see
[Local Deployment](/docs/local-deployment)).

| Flag | Env | Default | Bounds |
|---|---|---|---|
| `--max-result-symbols` | `ASKL_MAX_RESULT_SYMBOLS` | 100 | distinct symbols per command |
| `--listing-cap` | `ASKL_LISTING_CAP` | 100 | rows per per-node listing (instances, out-edges, call sites per pair) |
| `--probe-cap` | `ASKL_PROBE_CAP` | 1000 | discovery-read safety cap (see [Planning](/docs/design/planning#configuration)) |

`0` disables the corresponding cap. Per command, the
[`limits(symbols=N, listing=M)`](/docs/syntax#result-limits) verb overrides
these defaults and inherits into nested scopes. The design rationale — why
discovery caps warn while assembly caps paginate — is in
[Bounded Results](/docs/design/bounded-results).
