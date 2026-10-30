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
  version: "1.0.0"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---

# retro-doc — Diataxis Retro-Documentation Generator

You produce **high-quality, structured retro-documentation** for an existing
codebase: a feature, a module, or a whole project. You **research the code
thoroughly before writing a single line of documentation**.

This is a **prompt-only** skill: it has no scripts and no runtime dependency. It
uses only your own tools (Bash, Read, Write, Edit, Grep, Glob, AskUserQuestion).

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
   write outside it except the single README link (Step 7).

---

## Step 3 — Codebase archaeology (research phase)

The quality of the documentation depends on how well you understand the code, so
this step carries the most weight.

1. **Map the project structure:**

   ```bash
   find . -type f -not -path "./.git/*" -not -path "./node_modules/*" -not -path "./dist/*" -not -path "./build/*" -not -path "./.next/*" -not -path "./vendor/*" | head -200
   ```

2. **Read the entry points:**
   - `README*`, `ARCHITECTURE*`, `CONTRIBUTING*`, `CLAUDE.md` / `AGENTS.md`.
   - Project manifest: `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod`
     / `pom.xml` / `composer.json` (understand the project type and entry points).
   - Main entry files, configuration files and examples.

3. **Read the source for each target entity** (not just signatures):
   - Implementation files end to end.
   - Tests — they reveal intended behavior, edge cases, and real usage.
   - Related modules the target depends on or is depended upon by.
   - Existing inline comments, especially `NOTE:`, `DESIGN:`, `WHY:`.

4. **Build a concept map** (internal, not written to disk) for each target:

   ```
   Target: <name>
   Purpose: <one sentence — what problem it solves>
   Key concepts: <3-5 concepts a reader must understand>
   Public surface: <commands, functions, config options, API endpoints>
   Dependencies: <what it needs> / Dependents: <what relies on it>
   Edge cases: <from tests and code>
   Design decisions: <non-obvious "why" choices, with file:line evidence>
   ```

5. State what you found: "Researched N files, K public-surface items, M concepts,
   J design decisions." Then proceed to partition.

---

## Step 4 — Diataxis partition

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

## Step 5 — Write Reference documentation first

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

---

## Step 6 — Write Explanation documentation

Explanation docs answer "why does this work this way?" — the design rationale.
Use the Explanation template in `references/templates.md`.

Rules:
- Lead with the problem, not the solution.
- Use ASCII diagrams for architecture (grep-able, diff-friendly, render anywhere).
- Name trade-offs explicitly. "We chose X over Y because Z" is the gold standard.
- Pull "alternatives considered" from code comments, ADRs, or git history when
  discoverable; never invent them.
- Do not repeat reference material — link to it. File name `explanation-<slug>.md`.

### Tutorial / How-to (on request only)

If the user explicitly asked for Tutorial or How-to guides, use the corresponding
templates in `references/templates.md`. Otherwise skip them.

---

## Step 7 — INDEX & discoverability

After writing the docs:

1. **Write `<base>/retro/INDEX.md`** listing every generated document with its
   Diataxis quadrant and a one-line description (use the INDEX template).
2. **Cross-link quadrants.** Each Reference links to its Explanation and back.
3. **Link from the project README.** Add (or update) a single line pointing to
   `<base>/retro/INDEX.md`, under a "Documentation" heading. This is the **only**
   write outside `<base>/retro/`. If the user declines README edits, say so and
   skip this.
4. **Check for broken links.** Grep the generated files for `](` targets and
   confirm each referenced file exists:

   ```bash
   grep -rhoE "\]\(([^)]+\.md)\)" <base>/retro/ | sort -u
   ```

---

## Step 8 — Quality self-review

Before reporting done, review each document:

**Accuracy gate**
- [ ] Every API description matches the actual code signature.
- [ ] Every command/example would work if copy-pasted.
- [ ] No stale references to renamed or removed entities.

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

## Step 9 — Secret scan, then STOP (no commit, no push)

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

---

## Hard rules

- **Research before writing.** Step 3 is not optional.
- **Accuracy over everything.** Traceable to code or not stated.
- **Reference ≠ Explanation ≠ How-to ≠ Tutorial.** Do not mix quadrants.
- **Default is Reference + Explanation + INDEX.** Tutorial/How-to on request only.
- **Dedicated output dir.** Generated files stay under `<base>/retro/`.
- **One language per project, asked once, persisted.**
- **Never commit or push.** Write files, scan for secrets, report, stop.
