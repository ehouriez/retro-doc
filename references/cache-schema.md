# `.retro-doc.yml` cache schema

The skill persists one file per project — `<base>/retro/.retro-doc.yml` — holding
the resolved output language (v1) plus, from v2 on, the incremental-regeneration
cache: the last successful run's Git SHA and a provenance-tagged entity map.

One file keeps the v1 behaviour intact (the `language:` key is never touched) and
gives incremental runs everything they need in a single read/write.

## Versions

| `schema_version` | Content | Regeneration behaviour |
|---|---|---|
| absent or `1` | `language:` only (v1 cache) | Full run forced, then rewritten as v2 |
| `2` | `language:` + `source_sha` + `entities[]` | Incremental by default (see Step 3) |

A missing or `1` value is the retro-compatibility signal: a project documented by
v1 upgrades transparently on its next run.

## Fields

| Field | Type | Meaning |
|---|---|---|
| `language` | string | Resolved output language (v1 — unchanged). |
| `schema_version` | integer | Cache format version. `2` for this schema. |
| `source_sha` | string | Git SHA of the last successful run (full or incremental). Diff base for the next incremental run. |
| `entities` | list | One entry per documented entity (see below). |

### `entities[]` entry

| Field | Type | Meaning |
|---|---|---|
| `id` | string | Stable kebab-case entity id, reused across runs. |
| `quadrants` | list | Diataxis quadrants produced for this entity (`reference`, `explanation`, …). |
| `docs` | map | Quadrant → generated file name under `<base>/retro/`. |
| `sources` | list | `file:line` or `file:start-end` anchors that feed this entity. Drives invalidation (a changed source invalidates the entity) and the targeted re-read of low-confidence claims. |
| `deps` | list | Ids of entities this one depends on. Drives one-level invalidation propagation. |
| `confidence` | `high` \| `low` | Aggregate confidence of the entity's claims. |

## Example

```yaml
# <base>/retro/.retro-doc.yml
language: fr                 # v1 — unchanged
schema_version: 2
source_sha: a1b2c3d          # SHA of the last successful run
entities:
  - id: metering-service
    quadrants: [reference, explanation]
    docs:
      reference: reference-metering-service.md
      explanation: explanation-metering-service.md
    sources:
      - server.js:127
      - metering/usage.js:1-88
    deps: [auth-middleware]
    confidence: high
```

## Invariants

- The `language:` key and any other v1 key are never clobbered on write.
- `sources` and `deps` anchors come from the concept map returned by the research
  sub-agent (see `delegation.md`): `sources` from each claim's `file:line`
  provenance, `deps` from the entity's `dependencies`.
- Entity ids are stable across runs so incremental diffs map a changed source
  back to the entity that owns it.
