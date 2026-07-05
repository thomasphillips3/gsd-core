# Loop Engineering

Principles for outer loops that re-invoke an agent with fresh context per iteration, grinding
through file-based plans until done. Backs `/gsd:ralph`; applies to any unattended-execution
design in GSD. Distilled from Geoff Huntley's Ralph technique (ghuntley.com/ralph) and Anthropic's
long-running-agent harness guidance.

---

## Core principles

1. **Fresh context per iteration.** Long-lived contexts rot — output quality degrades well before
   the window is full. Each iteration is a new session that reconstructs state from disk. Never
   carry conversation memory across iterations.
2. **One task per iteration.** The cardinal rule. A single scoped task keeps every iteration in
   the context "smart zone" and makes failure recovery meaningful (one bad iteration = one bad
   commit, not a tangled session).
3. **State lives in files and git, not the model.** Plan files, state files, learnings files,
   commit history. In GSD terms: ROADMAP.md, phase PLAN.md files, STATE.md are already the exact
   artifacts a loop needs — the loop just needs to read, act, update, commit.
4. **Backpressure over trust.** Deterministic gates (tests, typecheck, lint, build) must reject
   invalid work before it commits. A loop without backpressure drifts and overcooks. The gates are
   the steering wheel; the prompt is just the throttle.
5. **Termination is layered, never trust-based.** Text saying "done" is not a signal. Use all
   three: an exact-match completion sentinel, a hard iteration cap (the primary safety mechanism),
   and stall detection (N consecutive iterations without a commit = stop).
6. **Plans are disposable.** When a plan accumulates cruft or drifts from the code, regenerate it
   (plan-mode loop) rather than patching around it.
7. **Separate planning from building.** Two prompts: a plan prompt does gap analysis only (specs
   vs code → repaired task list, no implementation); a build prompt executes one task. Never both
   in one iteration.
8. **Capture learnings, not status.** A learnings file carries operational knowledge (commands,
   gotchas) across iterations. Status lives in STATE.md/plan checkboxes; mixing the two poisons
   both.
9. **Sit on the loop, not in it.** The operator's job is observing failure patterns and tuning the
   prompt/guardrails between runs — not babysitting individual iterations.

## Known failure modes → mitigations

| Failure | Mitigation |
|---------|------------|
| Reward hacking (marking done without doing, weakening tests) | "Never remove or weaken tests" guardrail; acceptance-criteria-derived validation; plan-mode audits of checked tasks |
| Duplicate implementation (fresh context assumes "not implemented") | "Search before assuming unimplemented" instruction; subagent-powered codebase search in the orient step |
| Infinite loop (sentinel never fires) | Hard iteration cap; stall detection; BLOCKED sentinel as an explicit escape hatch |
| Sentinel ambiguity (can't tell success from stuck) | Two distinct sentinels: COMPLETE and BLOCKED, with distinct exit codes |
| Context rot mid-iteration | One task per iteration; fan out searches/reads to subagents; short prompts (~100 focused words beat 1,500) |
| Broken repo on wake-up | Atomic commit every iteration; `git reset --hard` as the recovery move |
| Overcooking (invented features, gratuitous refactors) | Scoped plans; "plan order is priority order — don't reorder"; iteration caps |

## Ralph loop vs `/gsd:autonomous`

| | `/gsd:autonomous` | `/gsd:ralph` harness |
|---|---|---|
| Loop location | Inside one session (workflow orchestration) | Outside any session (bash re-invocation) |
| Context | Orchestrator context accumulates across phases | Fresh window every iteration |
| Granularity | Phase per loop step | One PLAN.md task per iteration |
| Human involvement | Pause points (grey areas, verification, blockers) | None until exit (complete/blocked/cap/stall) |
| Quality machinery | Full GSD pipeline (discuss, plan-checker, verifier, code review) | Backpressure commands + plan acceptance criteria |
| Survives crash / context exhaustion | No — session-bound | Yes — next iteration is a new process |
| Best for | Milestones needing judgment calls along the way | Well-specified grinding: plan burn-down, migrations, test backfill, mechanical work with crisp validation |

Rule of thumb: if a phase's tasks can be auto-validated, Ralph can grind them; if they need
judgment mid-flight, use autonomous mode. The two compose — autonomous mode for phases needing
decisions, a Ralph run for the mechanical stretches.

## Safety

- The generated loop runs `claude -p --dangerously-skip-permissions`. Run it only from a git
  worktree, container, or checkout you can discard.
- Always pass an iteration cap. Budget roughly: iterations × (one focused session's cost).
- Review the diff before merging a Ralph branch — unattended commits get the same review bar as
  any other PR.
