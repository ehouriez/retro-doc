# Diataxis partition matrix

Diataxis splits documentation by reader need. retro-doc defaults to the two
quadrants that describe *existing* code faithfully; the other two describe
*intended learning paths* and are produced only on explicit request.

| Quadrant | Reader need | Produced by retro-doc |
|---|---|---|
| **Reference** | Information — what it is, exactly | Yes (default) |
| **Explanation** | Understanding — why it is this way | Yes (default) |
| **How-to** | A specific task, step by step | On request only |
| **Tutorial** | Learning from zero to a first result | On request only |

## Per-entity decision

For each entity in scope, decide which quadrants apply:

| Entity type | Reference | Explanation | How-to | Tutorial |
|---|---|---|---|---|
| Public module / service | yes | yes | on request | on request |
| CLI command or flag | yes | no | on request | no |
| Internal module / architecture | yes | yes | no | no |
| Config option | yes | no | on request | no |
| API endpoint | yes | no | on request | on request |
| Design pattern / philosophy | no | yes | no | no |
| Multi-step workflow | no | on request | on request | on request |

## Why retro-doc defaults to Reference + Explanation

Retro-documentation describes a system that already exists. Reference captures
*what is there* (the public surface, traceable to code) and Explanation captures
*why it is there* (design rationale, trade-offs). Tutorials and How-tos encode a
chosen learning or task path — a product decision that usually has not been made
yet for legacy code. Generating them blind produces plausible-looking but
invented guidance, which is worse than no guide. So they stay opt-in.
