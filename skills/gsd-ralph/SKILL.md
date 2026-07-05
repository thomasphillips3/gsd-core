---
name: gsd-ralph
description: "Generate a Ralph loop harness — fresh-context outer loop that grinds phase plans unattended"
argument-hint: "[--phase N] [--regenerate]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
Generate a Ralph-style outer-loop harness for this project: a bash loop that re-invokes a fresh
Claude session with the same prompt every iteration until the roadmap is complete, blocked, or an
iteration cap hits. Each iteration does exactly one PLAN.md task. All cross-iteration state lives
in `.planning/` and git history — no conversation memory.

This is the loop-engineering counterpart to GSD's autonomous mode. Autonomous mode orchestrates phases
inside one session (accumulating orchestrator context, pausing for user decisions). The Ralph
harness runs *outside* any session: each iteration is a brand-new process with a clean context
window, so it survives crashes and context exhaustion, and runs fully unattended.

**Creates/Updates:**
- `.planning/ralph/PROMPT-build.md` — one-task-per-iteration build prompt
- `.planning/ralph/PROMPT-plan.md` — gap-analysis prompt (spec-vs-code drift repair, no implementation)
- `.planning/ralph/loop.sh` — the loop: completion/blocked sentinels, stall detection, iteration cap, per-iteration logs
- `.planning/ralph/LEARNINGS.md` — operational learnings carried across iterations (seeded once, never overwritten)

**After:** Run `bash .planning/ralph/loop.sh build 25` from a worktree or sandbox to grind the milestone.
</objective>

<execution_context>
@~/.claude/gsd-core/workflows/ralph.md
@~/.claude/gsd-core/references/loop-engineering.md
</execution_context>

<context>
Optional flags:
- `--phase N` — scope the build loop to phase N only (default: whole roadmap).
- `--regenerate` — overwrite existing PROMPT-*.md and loop.sh. LEARNINGS.md is never touched.

The harness requires existing GSD planning artifacts (ROADMAP.md, phase PLAN.md files) — they are
the loop's fuel. If no plans exist yet, run `/gsd-plan-phase` first or start the loop in plan mode.
</context>

<process>
Execute end-to-end. Preserve all workflow gates (planning validation, backpressure detection,
harness generation, safety report).
</process>
