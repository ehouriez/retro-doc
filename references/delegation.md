# Research delegation & concept-map contract

Code archaeology — reading the whole source surface — is the dominant token cost
of a retro-doc run. The skill delegates it to a fresh `general-purpose` sub-agent
(the **researcher**) running on a cheaper model, which returns a compact,
provenance-tagged **concept map**. The calling model (the **writer**) keeps only
the judgement work: Diataxis partition, low-confidence arbitration, Explanation
prose, and final writing.

## The two roles

| Role | Who runs it | Tier | Responsibility |
|---|---|---|---|
| **Writer** | The model running `/retro-doc` (the caller, already active) | `reasoning` recommended | Diataxis partition, arbitration of `confidence: low` claims, Explanation prose (trade-offs, rationale), final writing. |
| **Researcher** | A `general-purpose` sub-agent launched via the `Agent` tool (overridden model) | `standard` | Cartography + heavy code reading, producing the provenance-tagged concept map. |

### Tier → model

| Tier | `Agent` `model` value |
|---|---|
| `standard` | `sonnet` |
| `reasoning` | `opus` |

### Non-blocking recommendation

The skill **cannot** force a model switch (an IDE action). It **recommends** the
writer run on a `reasoning` model and stays functional if the writer is
`standard` — quality just degrades on the arbitration of doubtful claims. The
heavy reading is already off the writer (delegated to the `standard` researcher),
so keeping the writer on `reasoning` costs only on output (the minor cost post),
precisely where judgement pays. State the recommendation once, do not block.

## Delegation protocol

1. After the base + language are resolved, run a **light cartography**
   (`glob`/`grep`) to estimate the entities in scope. This is cheap (`find`/`grep`),
   so the writer does it inline — no separate agent.
2. **Delegation is mandatory — the writer never reads source for archaeology,
   whatever the scope.** A small, simple or "trivial-looking" module is not a
   reason to read inline. Adaptive parallelism sets only the **number** of
   researchers (always ≥ 1), never **whether** to delegate:
   - ≤ ~3 entities → **one** researcher sub-agent for the whole scope (minimum one).
   - more → **one sub-agent per coherent group** of entities, launched in parallel.
   - This avoids paying N start-up costs on a small project — but one start-up
     cost is always paid, because the heavy reading always leaves the writer.
3. Launch each researcher with the `Agent` tool:
   - `subagent_type: "general-purpose"`
   - `model: "sonnet"` (the resolved `standard` tier)
   - `prompt`: archaeology instructions (same depth as the inline v1 research:
     implementation end to end, tests, dependencies, inline `NOTE:`/`DESIGN:`/`WHY:`
     comments) **plus the output contract below** — every non-trivial claim must
     carry a `file:line` `source` and a `confidence`.
   - **Search hygiene (mandatory in the prompt).** Instruct the researcher that
     every recursive search **must exclude vendored/build trees**, otherwise a
     `grep -rn` over a project with dependencies installed scans tens of thousands
     of files (often on a slow mount) and hangs. Require
     `grep -rn --exclude-dir={node_modules,dist,build,.next,vendor,.git} …` (or
     `rg`, which honours `.gitignore` by default). Same exclusion set as the
     cartography `find` in `SKILL.md` Step 4.
   - **Never use `fork`.** A fork inherits the parent context (which ruins the
     token economy) and ignores the `model` override.
4. **Fallback — tool genuinely unavailable ONLY.** The writer does the archaeology
   inline (identical to the v1 Step 3) **only** when the `Agent` tool cannot be
   used: not callable in this IDE, erroring, or unable to honour the `model`
   override. Scope size or perceived triviality is **never** a valid reason to
   fall back — delegate instead (item 2). Inline is the degraded path that keeps
   the skill prompt-only and multi-IDE safe, not an optimisation choice.

## Concept-map output contract

For **each entity in scope**, the researcher returns a parsable structured object
(alongside any prose):

```
id                   stable kebab-case id
purpose              one sentence — what problem it solves
key_concepts[]       3-5 concepts a reader must understand
public_surface[]     each item: { text, source "file:line", confidence: high|low }
dependencies[]       entity ids this one depends on   → feeds entities[].deps
dependents[]         entity ids that rely on this one
edge_cases[]         each item: { text, source "file:line" }
design_decisions[]   each item: { text, source "file:line", confidence: high|low }
suggested_quadrants  reference / explanation (per references/diataxis.md)
```

**Rule: every non-trivial claim carries `source` (`file:line`) and `confidence`.**
The `confidence: low` claims are exactly the ones the writer re-reads before
writing (targeted re-read of the cited lines, not the whole file). The provenance
serves a double purpose: fine-grained invalidation and targeted re-verification,
extending the `file:line` accuracy already validated in v1.

This contract maps directly onto the persisted cache (`references/cache-schema.md`):
each claim's `source` feeds `entities[].sources`, `dependencies` feed
`entities[].deps`, and the aggregate confidence feeds `entities[].confidence`.
