---
name: retro-doc
description: >
  Generate targeted retro-documentation from an existing codebase using the
  Diataxis method. Researches the code first, then writes Reference +
  Explanation docs and a linked INDEX (Tutorial/How-to on demand). Prompt-only,
  no runtime dependency. Output language is auto-detected per project and
  persisted. Never commits or pushes. Use when asked to document, retro-document,
  or write docs for an existing project, module, or service.
license: MIT
metadata:
  author: Emmanuel Houriez
  version: "2.2.2"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - Agent
  - Task
---

# retro-doc — Diataxis Retro-Documentation Generator

You produce **high-quality, structured retro-documentation** for an existing
codebase: a feature, a module, or a whole project. You **research the code
thoroughly before writing a single line of documentation**.

This is a **prompt-only** skill: it has no scripts and no runtime dependency. It
uses only your own tools (Bash, Read, Write, Edit, Grep, Glob, AskUserQuestion,
and the `Agent` tool for delegated research when available).

## Core principles

- **Research the whole, then write the parts.** Survey the full codebase surface
  before writing any document. This prevents documentation that describes half
  the system.
- **Accuracy is non-negotiable.** Every claim must be traceable to the code. If
  unsure about a detail, read the source again. Do not guess.
- **Targeted by default.** The default deliverable is **Reference + Explanation +
  INDEX** — what retro-documentation of existing code actually needs. Tutorial
  and How-to are produced **only on explicit request** (they describe intended
  learning paths, which rarely exist yet in a retro context).
- **Regenerate only what changed.** Repeated runs are incremental: an entity
  whose sources did not change since the last run is neither re-researched nor
  rewritten (see Step 3).
- **Never commit or push.** You write files and stop. The user owns all git
  operations. Do not run `git add`, `git commit`, `git push`, or create PRs.

## Diataxis in one paragraph

Diataxis splits documentation into four reader needs:
**Reference** (information — complete, accurate technical description),
**Explanation** (understanding — why things work the way they do),
**How-to** (task — steps to accomplish a specific goal),
**Tutorial** (learning — a newcomer from zero to a working example).
Retro-doc defaults to Reference + Explanation. See `references/diataxis.md` for
the partition matrix.

---

## Step 0 — Scope & intent

1. Determine what to document:
   - A specific target named by the user (feature, module, file, service) → scope is that target.
   - No target given → scope is the whole project.
2. Confirm scope in one sentence before researching, and state the default
   deliverable: "I will produce Reference + Explanation docs and an INDEX for
   `<scope>`. Say so if you also want Tutorial or How-to guides."
3. Do not ask more than this unless the scope is genuinely ambiguous.

---

## Step 1 — Output language (detect, confirm once, persist)

The output language is **per project** and **persisted** so it is asked at most once.

1. **Resolve the output base** first (see Step 2) so you know where the config lives.
2. **Read the persisted choice** if present:

   ```bash
   cat <base>/retro/.retro-doc.yml 2>/dev/null
   ```

   If it contains a `language:` value, use it without asking.
3. **Otherwise auto-detect** the language of the existing docs to propose a default:

   ```bash
   cat README* 2>/dev/null | head -40
   ```

   Judge the dominant natural language of the README and any existing `doc/`,
   `docs/` content (French, English, …). Prose only — ignore code, identifiers,
   and command output.
4. **Confirm once** via `AskUserQuestion`: "Output language for the generated
   documentation?" with the detected language as the recommended option, plus the
   other plausible option and a free-text escape.
5. **Persist** the answer (create the file if absent, do not clobber other keys):

   ```
   # <base>/retro/.retro-doc.yml
   language: fr
   ```

   This file is also the incremental-regeneration cache from v2 on
   (`schema_version`, `source_sha`, `entities[]`). The full format is in
   `references/cache-schema.md`; never clobber keys you do not own.

6. **Override**: if the user names a language in their request ("documente en
   anglais"), that wins for this run and is written back to the config.

All generated prose is written in the resolved language. Headings, templates and
structure stay identical across languages; only the prose is translated. Code,
identifiers, commands and file paths are never translated.

---

## Step 2 — Output location (detect base, dedicated `retro/`)

Generated docs go into a **dedicated** directory so regeneration never clobbers
hand-written documentation.

1. **Detect the documentation base** of the target project:
   - `doc/` exists → base is `doc/`.
   - else `docs/` exists → base is `docs/`.
   - else neither → base is `docs/` (create it).
   - If a doc framework is detected (Docusaurus, MkDocs, VitePress, Nextra: a
     `mkdocs.yml`, `docusaurus.config.*`, `.vitepress/`, or `nextra` dep), follow
     its conventions for file placement and sidebar/nav, but still isolate
     generated files under a `retro/` subtree.
2. **Write under `<base>/retro/`.** All generated files live there. This is the
   frontier between generated (regenerable) and authored content.
3. On regeneration, you may overwrite files under `<base>/retro/` freely; never
   write outside it except the single README link (Step 8).

---

## Step 3 — Run mode (catalogue-first, then full vs incremental)

Decide **what this run produces** before any archaeology. Two decisions, in order:
first catalogue vs expand (the `--expand` flag), then — only inside expand — full
vs incremental. This is what makes retro-doc cheap: the default run only inventories
the code, and even an expand re-researches only what changed.

### Catalogue vs expand (decided first)

A run is **catalogue-first by default**. The `--expand` flag decides whether this
run only inventories the codebase or also writes full docs:

| Prompt | Mode | What the run does |
|---|---|---|
| no `--expand` | **CATALOGUE** (default) | Inventory only — no docs written. |
| `--expand` (bare) or `--expand all` | **EXPAND all** | Generate docs for every `catalogued` entity. |
| `--expand <id>[,<id>…]` | **EXPAND targeted** | Generate docs only for the named entities. |

**CATALOGUE mode** — the cheap inventory path. It never delegates deep research and
never writes a Reference or Explanation file:

1. Run the light cartography of Step 4 item 1 (`glob`/`grep` + entry points) only.
2. Derive **one entity per coherent subsystem**: a stable kebab-case `id` and a
   one-line `description`. No public-surface listing, no archaeology.
3. Persist every entity with `status: catalogued` in the v2 cache (Step 9), leaving
   any already-`generated` entity untouched.
4. (Re)write the INDEX (Step 8), produce the **catalogue report** below, then
   **STOP** — skip Steps 4–7 and Step 11 (no generated doc files to scan).

   ```
   retro-doc catalogue:
     Scope:     <scope>
     Mode:      catalogue — N entity(ies) inventoried, 0 docs written
     Language:  <lang>  (persisted in <base>/retro/.retro-doc.yml)
     Output:    <base>/retro/INDEX.md

   Entities catalogued:
     ⬜ <id-1>   — <one-line description>
     ⬜ <id-2>   — <one-line description>
     …

   Next: generate full docs with --expand
     All entities:   /retro-doc --expand all
     One entity:     /retro-doc --expand <id-1>
     Several:        /retro-doc --expand <id-1>,<id-2>
   ```

**EXPAND mode** — generate full docs for the targeted entities. Resolve each `<id>`
against the `entities[]` of `.retro-doc.yml`; a bare `--expand` or `--expand all`
targets every entity currently `status: catalogued`. For each targeted entity the
full pipeline runs (Steps 4–9): delegate research, write Reference + Explanation,
flip its cache entry to `status: generated`, and refresh its INDEX entry. The full
vs incremental decision below then applies to this expand set.

> **Interaction with the incremental logic.** Full vs incremental only concerns
> already-`generated` entities: a `catalogued` entity being expanded has nothing
> written yet to invalidate, so it is always a fresh generation. The `git diff`
> invalidation below applies to `generated` entities on a later run.
>
> **Unknown `--expand <id>`.** If a named id matches no entity in the cache, report
> it and run the catalogue first so the ids exist — never invent an entity.

### Decide full vs incremental (within EXPAND mode)

Read the cache resolved in Step 1 (`<base>/retro/.retro-doc.yml`):

```
IF cache absent OR schema_version ∈ {absent, 1}   → FULL   (v1 cache / first run)
ELSE IF the prompt contains "full"                 → FULL   (explicit override)
ELSE (cache is v2; default, or the prompt says "incrémental"/"incremental") → INCREMENTAL
```

- A missing `schema_version` (or `1`) is the v1 retro-compat signal: force a FULL
  run, then Step 9 rewrites the cache as v2 — transparent, no breakage on a
  project already documented by v1.
- `full` in the prompt always forces a full run even when a v2 cache exists.
- With a v2 cache and no keyword, **incremental is the default** — no interruption,
  still steerable by keyword.

A **FULL** run documents every entity in scope: skip the rest of this step and go
to Step 4 with the full scope.

### Incremental run — compute the invalidated entities

1. **Diff the sources against the last run** (committed history only):

   ```bash
   git diff --name-only <source_sha>..HEAD
   ```

   `<source_sha>` is the `source_sha` value from the cache.

2. **If this is not a git repo, or `<source_sha>` is unknown** (missing commit,
   shallow clone, rewritten history) → **fall back to a FULL run and say so.** Do
   not guess a diff.

3. **Invalidate by changed source, then propagate one level** down the `deps`
   graph persisted in the cache:

   ```
   invalidated = { e ∈ entities | some file in e.sources appears in `changed` }
   invalidated += { e ∈ entities | e.deps ∩ invalidated ≠ ∅ }   # one level only
   ```

   - A source matches an entity when the changed path equals the **file part** of
     any `file:line` / `file:start-end` anchor in `e.sources`.
   - **One level only** is deliberate: a full transitive closure would re-invalidate
     almost everything and cancel the saving (see the plan's "Invalidation"
     decision). A dependent of a dependent is not invalidated.

4. **Scope the rest of the run to `invalidated` only:** Step 4 researches only these
   entities; Steps 6–8 rewrite only their docs and INDEX entries; every
   non-invalidated entity keeps its existing doc files and its cache entry
   unchanged.

5. **Announce the mode** before researching, e.g.:
   "Incremental run — 2 entity(ies) invalidated by the diff (`metering-service`
   directly, `billing` via deps); researching and rewriting only those, N others
   left intact."

> **Limit — committed changes only.** The diff base is `source_sha..HEAD`, so
> uncommitted working-tree edits are not detected. Commit before an incremental
> run, or pass `full` to force a complete regeneration.

---

## Step 4 — Codebase archaeology (delegated research phase)

The quality of the documentation depends on how well the code is understood, so
this step carries the most weight. Reading the whole source surface is also the
dominant token cost of a run — so the **heavy reading is delegated** to a fresh
`general-purpose` sub-agent (the **researcher**) on a cheaper `standard` model,
which returns a provenance-tagged **concept map**. The calling model (the
**writer**) keeps the judgement work (partition, Explanation prose, arbitration
of doubtful claims, final writing). See `references/delegation.md` for the two
roles, the tier → model mapping, and the full output contract.

> **Research scope.** This step runs in **EXPAND mode only** — a CATALOGUE run
> (Step 3) skips it entirely. Within expand: on a **full** run research every
> expanded entity; on an **incremental** run research **only the invalidated
> entities**; reuse every non-invalidated entity's map entry unchanged from the cache.

> **Writer model recommendation (non-blocking).** The heavy reading is off the
> writer, so keeping the writer on a `reasoning` model costs only on output —
> where judgement pays. Recommend it once; never block. The skill stays
> functional if the writer is `standard`.

1. **Light cartography (inline — `glob`/`grep`, cheap).** Estimate the entities
   in scope and the project shape:

   ```bash
   find . -type f -not -path "./.git/*" -not -path "./node_modules/*" -not -path "./dist/*" -not -path "./build/*" -not -path "./.next/*" -not -path "./vendor/*" | head -200
   ```

   Read the entry points inline: `README*`, `ARCHITECTURE*`, `CONTRIBUTING*`,
   `CLAUDE.md` / `AGENTS.md`, and the project manifest (`package.json` /
   `pyproject.toml` / `Cargo.toml` / `go.mod` / `pom.xml` / `composer.json`).
   This is **cartography to size the scope, not the archaeology**: the module's
   implementation, tests and dependencies are read by the researcher (item 2),
   never by the writer here.

2. **Delegate the deep research — always, regardless of scope size.** The heavy
   source reading is **never** done by the writer, not even for a single small
   entity or a scope that looks trivial: the writer orchestrates and writes, the
   researcher reads. A "small" or "simple" module is **not** a reason to read
   inline — delegate anyway. Launch the researcher(s) via the `Agent` tool
   (`subagent_type: "general-purpose"`, `model: "sonnet"` — the `standard` tier).
   **Adaptive parallelism controls the _number_ of researchers, never _whether_
   to delegate:** ≤ ~3 entities → **one** sub-agent for the whole scope (minimum
   one, always); more → one sub-agent per coherent group of entities, launched in
   parallel. Each researcher reads the source end to end — implementation, tests
   (intended behavior, edge cases, real usage), dependencies, and inline
   `NOTE:`/`DESIGN:`/`WHY:` comments — and returns the **concept-map output
   contract** (`references/delegation.md`): for each entity, `purpose`,
   `key_concepts`, `public_surface`, `dependencies`/`dependents`, `edge_cases`,
   `design_decisions`, `suggested_quadrants`. **Every non-trivial claim carries a
   `file:line` `source` and a `confidence`.** **Instruct the researcher that every
   recursive search must exclude vendored/build trees** —
   `grep -rn --exclude-dir={node_modules,dist,build,.next,vendor,.git} …` or `rg`
   (same exclusion set as the cartography `find` above): without it a `grep -rn`
   over a project with dependencies installed scans tens of thousands of files and
   hangs. **Self-check before returning (mandatory in the prompt).** Instruct
   the researcher to verify before handing back its concept map: every entity
   in the announced scope is covered (none omitted); every non-trivial claim
   carries `source` (`file:line`) and `confidence`; no entity, function,
   relation or endpoint was invented (everything traceable to the code); all
   recursive searches excluded vendored/build trees. **Never use `fork`** — it
   inherits the parent context (ruining the token economy) and ignores the
   `model` override.

3. **Fallback — inline archaeology (tool genuinely unavailable ONLY).** Only when
   the `Agent` tool cannot be used — not callable in this IDE, erroring, or unable
   to honour the `model` override — does the writer do the research itself
   (identical to the v1 Step 3: read implementation end to end, tests,
   dependencies, inline comments; build the same concept map, with the same
   `source`/`confidence` on every claim). **A small, simple, or "trivial-looking"
   scope is never a reason to fall back** — that is a judgment call the skill
   forbids; delegate (item 2) instead. The same search hygiene applies — exclude
   `node_modules,dist,build,.next,vendor,.git` from recursive greps. Inline is the
   degraded path that keeps the skill prompt-only and multi-IDE safe, not an
   optimisation choice. The rest of the pipeline is identical either way.

4. **Consolidate the map.** The returned concept map (or the inline one) is the
   single input to partition and writing. Each claim's `source`, the
   `dependencies`, and the confidence feed the persisted cache
   (`references/cache-schema.md`).

5. **Targeted re-read of low-confidence claims (quality guard).** Before partition
   and writing, the **writer itself** — not a gate, not a sub-agent — scans the
   concept map and, for every claim marked `confidence: low`, re-reads the few
   cited lines via `Read`/`Grep` on its `file:line` `source`: **the cited lines
   only, never the whole file.** Correct or confirm the claim in the map. This is
   internal to the run, silent, with **no user interaction, no `/model`, no
   `AskUserQuestion`**. It extends the `file:line` accuracy validated in v1 and
   keeps quality ≥ v1 even when the researcher ran on a `standard` model.

6. State what you found: "Researched N files, K public-surface items, M concepts,
   J design decisions; re-read P low-confidence claims." Then proceed to partition.

---

## Step 5 — Diataxis partition

For each target entity, decide which quadrants to produce. Default retro-doc set
is **Reference + Explanation + INDEX**. Add Tutorial/How-to **only if the user
asked**. Use the matrix in `references/diataxis.md`.

Output the partition plan as a short table before writing:

```
Documentation plan (<scope>, language <lang>, output <base>/retro/):
  Entity                Reference   Explanation   (Tutorial/How-to)
  Widget system         yes         yes           on request
  Config loader         yes         no            -
```

If the plan has more than ~8 documents, confirm the list with the user before
writing. For smaller scopes, proceed.

---

## Step 6 — Write Reference documentation first

Reference docs are the foundation: factual, complete, derived directly from code.
Write them before Explanation because they establish the vocabulary. Use the
Reference template in `references/templates.md`.

Rules:
- Accuracy over elegance. Every claim traceable to code.
- Include types, defaults, and constraints. "Accepts a string" is insufficient —
  "Accepts a string (max 256 chars, must match `^[a-z-]+$`)" is reference-grade.
- Show real examples that would work if copy-pasted.
- Do not explain *why* here — that belongs in Explanation.
- One Reference doc per coherent entity; file name `reference-<slug>.md`.

> **Incremental runs:** write only the Reference docs of the invalidated entities
> (Step 3); leave every other `reference-*.md` untouched.

---

## Step 7 — Write Explanation documentation

Explanation docs answer "why does this work this way?" — the design rationale.
Use the Explanation template in `references/templates.md`.

Rules:
- Lead with the problem, not the solution.
- Use ASCII diagrams for architecture (grep-able, diff-friendly, render anywhere).
- Name trade-offs explicitly. "We chose X over Y because Z" is the gold standard.
- Pull "alternatives considered" from code comments, ADRs, or git history when
  discoverable; never invent them.
- Do not repeat reference material — link to it. File name `explanation-<slug>.md`.

> **Incremental runs:** write only the Explanation docs of the invalidated entities
> (Step 3); leave every other `explanation-*.md` untouched.

### Tutorial / How-to (on request only)

If the user explicitly asked for Tutorial or How-to guides, use the corresponding
templates in `references/templates.md`. Otherwise skip them.

---

## Step 8 — INDEX & discoverability

After writing the docs:

1. **Write `<base>/retro/INDEX.md`** — list **every entity** (generated and
   catalogued) from the v2 cache, with a status marker and a description. Use the
   INDEX template in `references/templates.md`. ✅ for `generated` entities (docs
   linked inline); ⬜ for `catalogued` entities (docs column shows `—`). The INDEX
   is **always fully regenerated** on every run — catalogue runs as well as expand.
2. **Cross-link quadrants.** Each Reference links to its Explanation and back
   (applies to `generated` entities only — `catalogued` entities have no docs yet).
3. **Link from the project README.** Add (or update) a single line pointing to
   `<base>/retro/INDEX.md`, under a "Documentation" heading. This is the **only**
   write outside `<base>/retro/`. If the user declines README edits, say so and
   skip this.
4. **Check for broken links.** Grep the generated files for `](` targets and
   confirm each referenced file exists:

   ```bash
   grep -rhoE "\]\(([^)]+\.md)\)" <base>/retro/ | sort -u
   ```

> **All run types (catalogue, expand full, expand incremental):** always
> regenerate the full INDEX — entity statuses change across runs (new
> `catalogued` entries, `catalogued` → `generated` flips) and a partial
> regeneration would leave the dashboard stale.

---

## Step 9 — Cache persistence (write `.retro-doc.yml` v2)

After writing all documentation and the INDEX, persist the run results so future
incremental runs can reuse the concept map and skip unchanged entities.

1. **Resolve HEAD SHA** (skip silently if not a git repo):

   ```bash
   git rev-parse HEAD 2>/dev/null || true
   ```

2. **Build the `entities[]` array** from the concept map assembled in Step 4 and
   the partition and files written in Steps 5–8. For each entity:

   - `id` — kebab-case entity id from the concept map.
   - `status` — `generated` for an entity whose docs were written this run;
     `catalogued` for an inventory-only entity (catalogue mode, no docs). Preserve
     the existing `status` of any entity left untouched this run.
   - `description` — one-line description (the concept map `purpose`, condensed).
     Always written, including for `catalogued` entities.
   - `quadrants` — the quadrants produced (from Step 5 partition).
   - `docs` — map of quadrant → filename written under `<base>/retro/`.
   - `sources` — `file:line` provenance anchors from the concept map's claims.
   - `deps` — entity ids from the concept map's `dependencies` field.
   - `confidence` — aggregate: `high` if all claims are `high`, `low` if any is `low`.

3. **Write `.retro-doc.yml` v2** — merge, never clobber `language:` or other
   pre-existing keys:

   ```yaml
   language: <resolved in Step 1>
   schema_version: 2
   source_sha: <HEAD SHA, or omit if not a git repo>
   entities:
     - id: <entity-id>
       status: generated           # or `catalogued` (inventory only, no docs)
       description: <one-line>
       quadrants: [reference, explanation]
       docs:
         reference: reference-<slug>.md
         explanation: explanation-<slug>.md
       sources:
         - <file>:<line>
       deps: [<dep-entity-id>]
       confidence: high | low
   ```

   Full field reference: `references/cache-schema.md`.

   **On a catalogue run** (no `--expand`, Step 3): write one `status: catalogued`
   entry per inventoried entity (`id` + `description` only), preserving any existing
   `status: generated` entry unchanged.
   **On an expand full run:** write all expanded entities with `status: generated`.
   **On an expand incremental run:** rewrite only the invalidated entities' entries;
   copy the non-invalidated entries unchanged from the previous cache. `source_sha`
   is set to the current HEAD in all cases.

4. **Coherence with Step 8 (INDEX):** the entities written here must match exactly
   the entities whose doc files are present under `<base>/retro/`.

5. **Coherence with Step 11 (secret scan):** pass the set of files just written
   (from `docs` entries) to the scan. On a full run this is equivalent to all
   files under `<base>/retro/`; on an incremental run only the regenerated subset.

---

## Step 10 — Quality self-review

Before reporting done, review each document:

**Accuracy gate**
- [ ] Every API description matches the actual code signature.
- [ ] Every command/example would work if copy-pasted.
- [ ] No stale references to renamed or removed entities.
- [ ] Every `confidence: low` claim was re-read against its `file:line` source (Step 4).

**Completeness gate**
- [ ] Reference docs cover 100% of the public surface in scope.
- [ ] Explanation docs name trade-offs, not just choices.
- [ ] INDEX lists every generated file; README links the INDEX.

**Voice gate**
- [ ] Written for a smart reader who has not seen the code.
- [ ] Prose is in the resolved output language; code/identifiers untouched.
- [ ] Active voice, concrete nouns, short sentences.

Fix any failure before reporting.

---

## Step 11 — Secret scan, then STOP (no commit, no push)

> **Catalogue runs skip this step entirely** — no doc files were generated, so
> there is nothing to scan. The run report was already produced in Step 3, item 4.

> **Scan scope.** Target only the files written or rewritten during this run —
> the filenames from the `docs` entries of Step 9's entities, plus
> `.retro-doc.yml` itself. On a full run this equals all files under
> `<base>/retro/`; on an incremental run it covers only the regenerated subset.

1. **Scan the generated files for live-format secrets** before handing back.
   This replaces a binary redaction tool with a dependency-free grep. Flag, do
   not auto-edit:

   ```bash
   grep -rniE "(aws_secret_access_key|-----BEGIN [A-Z ]*PRIVATE KEY-----|xox[baprs]-[0-9A-Za-z-]+|gh[pousr]_[0-9A-Za-z]{20,}|AIza[0-9A-Za-z_-]{30,}|eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,})" <base>/retro/
   ```

   Also scan for obvious password/token assignments:

   ```bash
   grep -rniE "(password|passwd|secret|api[_-]?key|token)[\"' ]*[:=][\"' ]*[^\"'[:space:]]{6,}" <base>/retro/
   ```

   If anything matches and it is not a clearly fake placeholder
   (`AKIAIOSFODNN7EXAMPLE`, `your-token-here`, `xxx`), **remove or redact the
   secret in the generated doc and tell the user which file held it.** A leaked
   secret in committed docs is a real leak.

2. **Report and STOP.** Produce the summary below. **Do not** stage, commit,
   push, or open a PR — the user does that.

```
retro-doc generated:
  Scope:     <what was documented>
  Mode:      <catalogue (N inventoried) | expand full | expand incremental (K invalidated, N reused)>
  Language:  <lang>  (persisted in <base>/retro/.retro-doc.yml)
  Output:    <base>/retro/  (+ 1 line added to README)
  Files:     <N> new, <M> updated
    Reference:    <count> (<list>)
    Explanation:  <count> (<list>)
    Tutorial:     <count or - >
    How-to:       <count or - >
    INDEX:        INDEX.md
  Secret scan:  <clean | N finding(s) redacted in: files>
  Quality:      accuracy / completeness / voice gates passed
Next (yours): review the diff, then commit and push.
```

   If any entities remain `status: catalogued` after this run, append:

   ```
   Still to document (⬜ catalogued, docs not yet generated):
     ⬜ <id-1>   — <one-line description>
     ⬜ <id-2>   — <one-line description>
     …

   Next: generate their docs with --expand
     All remaining:  /retro-doc --expand all
     Targeted:       /retro-doc --expand <id-1>,<id-2>
   ```

---

## Hard rules

- **Research before writing.** Step 4 is not optional.
- **Always delegate the heavy reading.** The writer never does the archaeology
  itself whatever the scope size; inline archaeology is the tool-unavailable
  fallback only, never a scope-size or triviality judgment.
- **Accuracy over everything.** Traceable to code or not stated.
- **Reference ≠ Explanation ≠ How-to ≠ Tutorial.** Do not mix quadrants.
- **Default is Reference + Explanation + INDEX.** Tutorial/How-to on request only.
- **Catalogue-first.** A run with no `--expand` only inventories the code
  (`status: catalogued`, no docs). `--expand <id>|all` generates full docs and
  flips the targeted entities to `status: generated`.
- **Incremental by default on a v2 cache.** Within expand, re-research and rewrite
  only what the diff invalidated; `full` in the prompt forces a complete run.
- **Dedicated output dir.** Generated files stay under `<base>/retro/`.
- **One language per project, asked once, persisted.**
- **Never commit or push.** Write files, scan for secrets, report, stop.
