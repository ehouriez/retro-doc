# retro-doc

A reusable, multi-project **retro-documentation generator** for Claude Code,
driven by the [Diataxis](https://diataxis.fr/) method. It researches an existing
codebase, then writes **Reference + Explanation** docs and a linked **INDEX**
(Tutorial/How-to on request).

- **Prompt-only** — no scripts, no runtime dependency (no Python, Node, or bun).
- **Per-project output language** — auto-detected, confirmed once, persisted.
- **Dedicated, regenerable output** under `<doc|docs>/retro/`.
- **Never commits or pushes** — it writes files and stops; git stays yours.

## Install

```bash
claude plugin marketplace add ehouriez/retro-doc
claude plugin install retro-doc@retro-doc
```

## Usage

Invoke the skill, optionally with a target:

```
/retro-doc
/retro-doc document the auth module
/retro-doc documente tout le projet en français
```

The skill will:

1. Confirm the scope and default deliverable (Reference + Explanation + INDEX).
2. Resolve and persist the output language.
3. Research the codebase (structure, entry points, source, tests).
4. Partition each entity across Diataxis quadrants.
5. Write Reference docs, then Explanation docs, then the INDEX.
6. Link the INDEX from the project README.
7. Scan the generated files for leaked secrets.
8. Report a summary and stop — you review, commit, and push.

Ask explicitly if you also want **Tutorial** or **How-to** guides; they are not
produced by default (see "Why" below).

## Output layout

```
<doc|docs>/retro/
├── INDEX.md
├── .retro-doc.yml            # persisted output language
├── reference-<slug>.md
└── explanation-<slug>.md
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
