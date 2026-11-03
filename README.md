# retro-doc

A reusable, multi-project **retro-documentation generator** for Claude Code,
driven by the [Diataxis](https://diataxis.fr/) method. It researches an existing
codebase, then writes **Reference + Explanation** docs and a linked **INDEX**
(Tutorial/How-to on request).

- **Prompt-only** — no scripts, no runtime dependency (no Python, Node, or bun).
- **Catalogue-first by default** — a plain run inventories the codebase cheaply;
  use `--expand` to generate full docs for all or specific entities.
- **Incremental on re-runs** — only entities whose sources changed are
  re-researched and rewritten; unchanged entities are reused from cache.
- **Per-project output language** — auto-detected, confirmed once, persisted.
- **Dedicated, regenerable output** under `<doc|docs>/retro/`.
- **Never commits or pushes** — it writes files and stops; git stays yours.

## Install

```bash
claude plugin marketplace add ehouriez/retro-doc
claude plugin install retro-doc@retro-doc
```

## Usage

Invoke the skill, optionally with a target and `--expand` flag:

```
/retro-doc                                      → catalogue only (cheap inventory, no docs)
/retro-doc --expand all                         → catalogue + full docs for every entity
/retro-doc --expand auth-service,billing        → catalogue + docs for named entities only
/retro-doc document the auth module --expand all
/retro-doc documente tout le projet en français --expand all
```

### Catalogue run (default — no `--expand`)

1. Confirm scope and output language (auto-detected, persisted).
2. Light cartography (`glob`/`grep`) — one entity per subsystem with a one-line description.
3. Write the INDEX listing every entity with a ⬜ (catalogued) marker.
4. Stop — no deep research, no Reference or Explanation files written.

### Expand run (`--expand <id>[,<id>…]` or `--expand all`)

Picks up where the catalogue left off:

1. Delegate deep research to a researcher sub-agent (cheaper `standard` model).
2. Partition each entity across Diataxis quadrants.
3. Write Reference docs, then Explanation docs.
4. Update the INDEX — expanded entities flip from ⬜ to ✅.
5. Link the INDEX from the project README.
6. Scan generated files for leaked secrets.
7. Report a summary and stop — you review, commit, and push.

On subsequent runs the skill is **incremental by default**: only entities whose
sources changed since the last run are re-researched and rewritten.

Ask explicitly if you also want **Tutorial** or **How-to** guides; they are not
produced by default (see "Why" below).

## Output layout

```
<doc|docs>/retro/
├── INDEX.md                  # all entities: ✅ documented | ⬜ catalogued
├── .retro-doc.yml            # persisted language + incremental cache (v2)
├── reference-<slug>.md       # written by --expand only
└── explanation-<slug>.md     # written by --expand only
```

The `retro/` subtree is the frontier between generated (regenerable) docs and
hand-written ones. Re-running the skill may overwrite files there; it never
touches hand-written docs outside it, except one link added to the README.

## Why Reference + Explanation by default

Retro-documentation describes code that already exists. Reference captures *what
is there* and Explanation captures *why*. Tutorials and How-tos encode a chosen
learning or task path — a decision that usually has not been made for legacy
code, so generating them blind yields invented guidance. They stay opt-in.

## Structure

```
.
├── .claude-plugin/
│   ├── marketplace.json
│   └── plugin.json
├── SKILL.md                  # the workflow (prompt-only)
├── references/
│   ├── diataxis.md           # partition matrix
│   └── templates.md          # Reference / Explanation / INDEX / How-to / Tutorial
├── doc/roadmap/              # this skill's own development plans (roadmap-tracking)
├── LICENSE
└── README.md
```

## License

MIT — see [LICENSE](LICENSE).
