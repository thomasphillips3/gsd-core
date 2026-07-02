---
type: Added
pr: 1
---
**`/gsd-ralph` — Ralph loop harness generator** — new command that generates an external fresh-context outer loop under `.planning/ralph/`: a build prompt (one PLAN.md task per iteration, backpressure-gated), a plan-repair prompt (gap analysis, no implementation), and a `loop.sh` with completion/blocked sentinels, stall detection, and a hard iteration cap. Complements `/gsd-autonomous` (in-session, phase-driven, pause points) with unattended task-level grinding that survives crashes and context exhaustion. Includes a new `references/loop-engineering.md` codifying the principles.
