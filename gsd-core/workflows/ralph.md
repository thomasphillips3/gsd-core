<purpose>
Generate a Ralph-style outer-loop harness under `.planning/ralph/`: two prompt files (build, plan),
a loop script with sentinel/stall/cap exits, and a learnings file. The harness re-invokes a fresh
`claude -p` session per iteration — one PLAN.md task per iteration — with all cross-iteration state
on disk and in git. This workflow generates and commits the harness; it does not run the loop.
</purpose>

<required_reading>
Read all files referenced by the invoking prompt's execution_context before starting.
@~/.claude/gsd-core/references/loop-engineering.md
@~/.claude/gsd-core/references/git-planning-commit.md
</required_reading>

<process>

<step name="validate_planning" priority="first">

## Validate planning artifacts

The harness is fueled by GSD planning artifacts. Verify they exist:

```bash
[ -f .planning/ROADMAP.md ] && echo "roadmap: yes" || echo "roadmap: no"
PLAN_COUNT=$(find .planning/phases -name "*PLAN.md" 2>/dev/null | wc -l | tr -d ' ')
echo "plans: $PLAN_COUNT"
```

- **No ROADMAP.md:** Stop. Tell the user: "Ralph needs GSD planning artifacts to grind on. Run
  `/gsd:new-project` (new project) or `/gsd:new-milestone` first."
- **PLAN_COUNT is 0:** Warn but continue: "No PLAN.md files found — build mode will have nothing
  to do. Either run `/gsd:plan-phase` first, or start the loop in plan mode
  (`loop.sh plan 3`) to generate gap-analysis tasks."
</step>

<step name="parse_arguments">

## Parse arguments

From `$ARGUMENTS`:
- `--phase N` → `PHASE_SCOPE=N` (build loop restricted to phase N). Default: unset (whole roadmap).
- `--regenerate` → overwrite existing `PROMPT-*.md` and `loop.sh`.

If `.planning/ralph/PROMPT-build.md` already exists and `--regenerate` was NOT passed: stop and
tell the user the harness exists; rerun with `--regenerate` to overwrite (LEARNINGS.md and logs
are preserved either way).
</step>

<step name="detect_backpressure">

## Detect backpressure commands

Backpressure — deterministic validation that rejects bad work — is the steering wheel of the loop.
Detect the project's validation commands:

| Signal | Candidate commands |
|--------|-------------------|
| `package.json` scripts | `npm test`, `npm run lint`, `npm run typecheck`, `npm run build` (only scripts that exist) |
| `Cargo.toml` | `cargo test`, `cargo clippy -- -D warnings` |
| `go.mod` | `go test ./...`, `go vet ./...` |
| `pyproject.toml` / `setup.py` | `pytest`, `ruff check .` (if configured) |
| `CMakeLists.txt` | `cmake --build build && ctest --test-dir build` |
| `Makefile` with `test:` target | `make test` |
| `.planning/config.json` `commands` block | use verbatim if present |

Confirm with the user via AskUserQuestion (multiSelect: true):
- header: "Backpressure"
- question: "These validation commands will gate every loop iteration. Which apply?"
- options: each detected command (pre-listing all detected), user can add via Other

**Text mode (`workflow.text_mode: true` in config or `--text` flag):** replace AskUserQuestion
with a plain-text numbered list and ask the user to type choice numbers.

If nothing is detected and the user supplies nothing: proceed, but embed a warning comment in the
prompt and tell the user plainly — a Ralph loop without backpressure drifts and overcooks. The
task acceptance criteria in PLAN.md files become the only gate.

Store the confirmed commands as `BACKPRESSURE_COMMANDS` (one per line).
</step>

<step name="generate_prompts">

## Generate iteration prompts

```bash
mkdir -p .planning/ralph/logs
```

Write `.planning/ralph/PROMPT-build.md` with these substitutions:
- `{BACKPRESSURE_COMMANDS}` → the confirmed command list (or `# WARNING: no backpressure commands configured — acceptance criteria are the only gate`)
- `{SCOPE_CLAUSE}` → if `--phase N`: `Work ONLY within phase N. Ignore other phases. "All phases complete" below means "phase N complete".` — otherwise omit the line.

~~~markdown
# GSD Ralph — Build Iteration

You are one iteration of an unattended loop. Your context is fresh; the filesystem and git history
are your only memory. Do exactly ONE task this iteration, then exit. Another fresh iteration
follows you.

## 0. Orient (read, don't assume)

- Read `.planning/STATE.md` and `.planning/ROADMAP.md`.
- Current phase = the first phase not marked complete. {SCOPE_CLAUSE}
- Read the current phase's PLAN.md file(s) under `.planning/phases/`.
- Read `.planning/ralph/LEARNINGS.md` — operational knowledge from prior iterations.
- Run `git log --oneline -15` to see what recent iterations did.
- Before implementing anything, SEARCH the codebase for existing implementations. Do not assume a
  task is unimplemented because the plan says so — a prior iteration may have partially done it.
  Use subagents for broad searches to keep your own context lean.

## 1. Select ONE task

- Pick the first unchecked task (`- [ ]`) in the current phase's plan, in plan order.
- If every task in the current phase is checked: your one action this iteration is phase
  completion — mark the phase complete in `.planning/ROADMAP.md`, update `.planning/STATE.md`,
  commit, exit.
- If every phase in scope is complete: output the completion promise (see Completion) and exit.

## 2. Implement completely

- No placeholders, no stubs, no TODO comments, no "simplified for now". Implement the task fully.
- Satisfy the task's acceptance criteria. Write or extend tests when the criteria demand them.
- If you notice unrelated failing tests, fix them as part of this iteration — a green baseline is
  part of every increment.

## 3. Validate (backpressure)

Run:

```
{BACKPRESSURE_COMMANDS}
```

Failures mean your work is rejected — fix before committing. NEVER delete, skip, or weaken a test
or acceptance criterion to make validation pass. That is the one unforgivable move.

## 4. Record and commit

- Check the task's checkbox in PLAN.md. Update `.planning/STATE.md` position.
- Append operational learnings to `.planning/ralph/LEARNINGS.md` — build/test commands discovered,
  gotchas, conventions. Operational knowledge only; no status updates, no progress narration.
  Keep the file under ~60 lines; prune stale entries when you add new ones.
- Make ONE atomic commit in conventional format: `{type}({scope}): {task summary}`.

## Guardrails

99. ONE task per iteration. Do not start a second task, however small.
999. Never remove or weaken tests or acceptance criteria to make validation pass.
9999. If blocked (missing dependency, contradictory spec, unrecoverable error): write the blocker
to `.planning/ralph/BLOCKED.md` (what, why, what a human should decide), commit it, then output
exactly this line and exit:

<promise>GSD_RALPH_BLOCKED</promise>

## Completion

When every phase in scope is marked complete in ROADMAP.md, output exactly this line and exit:

<promise>GSD_RALPH_COMPLETE</promise>

Never output either promise in any other circumstance.
~~~

Write `.planning/ralph/PROMPT-plan.md` (no substitutions except `{SCOPE_CLAUSE}`):

~~~markdown
# GSD Ralph — Planning Iteration

You are one iteration of an unattended planning loop. Fresh context; disk and git are your memory.
Plan only — do NOT implement anything. Do NOT refactor anything.

## 0. Study

- Read `.planning/REQUIREMENTS.md` (if present), `.planning/ROADMAP.md`, `.planning/STATE.md`.
- Current phase = the first phase not marked complete. {SCOPE_CLAUSE}
- Read the current phase's PLAN.md file(s) and `.planning/ralph/LEARNINGS.md`.
- Study the source tree. Use subagents for searches to keep your context lean.

## 1. Gap analysis

Find divergence between what the plans claim and what the code does:
- Requirements or acceptance criteria with no implementation.
- `TODO`, `FIXME`, `unimplemented`, placeholder, and stub markers in source.
- Tasks checked off (`- [x]`) that are not actually done — verify a sample against the code.

Before declaring anything unimplemented, SEARCH for it. Grep-negative is not proof of absence.

## 2. Repair the plan

- Append unchecked tasks (with concrete acceptance criteria) to the current phase's PLAN.md for
  each real gap found.
- Un-check tasks falsely marked done, with a one-line note of what's missing.
- Do not remove or reorder existing unchecked tasks — plan order is priority order.

## 3. Commit

One commit: `chore({phase-slug}): ralph plan gap analysis`.

If you found NO gaps and changed nothing, output exactly this line and exit:

<promise>GSD_RALPH_COMPLETE</promise>
~~~
</step>

<step name="generate_loop_script">

## Generate the loop script

Write `.planning/ralph/loop.sh` and make it executable (`chmod +x`):

~~~bash
#!/usr/bin/env bash
# GSD Ralph loop — generated by /gsd:ralph. Regenerate with: /gsd:ralph --regenerate
#
# Usage: ./loop.sh [build|plan] [max_iterations]
#   build (default) — one PLAN.md task per iteration until roadmap complete
#   plan            — gap analysis: repair PLAN.md files against the code (default cap: 3)
#
# Env overrides: CLAUDE_BIN (default: claude), RALPH_MODEL (default: session default)
#
# Exit codes: 0 complete · 1 max iterations reached · 2 blocked (see BLOCKED.md) · 3 stalled
#
# SAFETY: runs claude with --dangerously-skip-permissions. Run this from a git worktree,
# container, or otherwise sandboxed checkout — never from a checkout you can't hard-reset.
set -uo pipefail

MODE="${1:-build}"
DEFAULT_MAX=25; [ "$MODE" = "plan" ] && DEFAULT_MAX=3
MAX="${2:-$DEFAULT_MAX}"
DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT="$(cd "$DIR/../.." && pwd)"
PROMPT_FILE="$DIR/PROMPT-$MODE.md"
LOG_DIR="$DIR/logs"
CLAUDE_BIN="${CLAUDE_BIN:-claude}"
COMPLETE='<promise>GSD_RALPH_COMPLETE</promise>'
BLOCKED='<promise>GSD_RALPH_BLOCKED</promise>'
STALL_LIMIT=2

[ -f "$PROMPT_FILE" ] || { echo "[ralph] no prompt for mode '$MODE' ($PROMPT_FILE)"; exit 1; }
mkdir -p "$LOG_DIR"
cd "$ROOT"

stalls=0
for ((i=1; i<=MAX; i++)); do
  ts="$(date +%Y%m%d-%H%M%S)"
  log="$LOG_DIR/$MODE-$i-$ts.log"
  head_before="$(git rev-parse HEAD 2>/dev/null || echo none)"
  echo "[ralph] $MODE iteration $i/$MAX -> $log"

  "$CLAUDE_BIN" -p - \
    --dangerously-skip-permissions \
    ${RALPH_MODEL:+--model "$RALPH_MODEL"} \
    < "$PROMPT_FILE" 2>&1 | tee "$log"

  if grep -qF "$BLOCKED" "$log"; then
    echo "[ralph] BLOCKED after $i iteration(s) — see .planning/ralph/BLOCKED.md"
    exit 2
  fi
  if grep -qF "$COMPLETE" "$log"; then
    echo "[ralph] COMPLETE after $i iteration(s)"
    exit 0
  fi

  head_after="$(git rev-parse HEAD 2>/dev/null || echo none)"
  if [ "$head_before" = "$head_after" ]; then
    stalls=$((stalls+1))
    echo "[ralph] no commit this iteration (stall $stalls/$STALL_LIMIT)"
    if [ "$stalls" -ge "$STALL_LIMIT" ]; then
      echo "[ralph] stalled — $STALL_LIMIT consecutive iterations without a commit. Stopping."
      exit 3
    fi
  else
    stalls=0
  fi
done

echo "[ralph] max iterations ($MAX) reached without completion"
exit 1
~~~
</step>

<step name="seed_learnings">

## Seed the learnings file

Only if `.planning/ralph/LEARNINGS.md` does not exist (never overwrite — it accumulates value
across runs):

```markdown
# Ralph Learnings — operational knowledge only

Rules: commands, gotchas, conventions. No status updates. Keep under ~60 lines; prune when adding.

## Validation commands

{BACKPRESSURE_COMMANDS as a bullet list}
```
</step>

<step name="commit_harness">

## Commit the harness

Follow git-planning-commit conventions (skip if planning commits are disabled in config):

```bash
git add .planning/ralph/
git commit -m "chore(ralph): generate ralph loop harness"
```
</step>

<step name="report" priority="last">

## Report

Show the user:

```
Ralph harness generated in .planning/ralph/

  PROMPT-build.md   one task per iteration, backpressure-gated
  PROMPT-plan.md    gap analysis (plan repair, no implementation)
  loop.sh           sentinel + stall detection + iteration cap
  LEARNINGS.md      cross-iteration operational memory

Run it:

  bash .planning/ralph/loop.sh build 25    # grind the roadmap, max 25 iterations
  bash .planning/ralph/loop.sh plan 3      # repair plans against the code first

Before you start:
  - Run from a git worktree or sandbox — the loop uses --dangerously-skip-permissions.
  - Always set an iteration cap. The cap, not the sentinel, is your primary safety mechanism.
  - Sit on the loop, not in it: watch logs/, and when the loop misbehaves, fix the prompt or
    LEARNINGS.md and restart. git reset --hard is the recovery move.

Exit codes: 0 complete · 1 cap reached · 2 blocked (read .planning/ralph/BLOCKED.md) · 3 stalled
```
</step>

</process>

<success_criteria>
- [ ] `.planning/ralph/` contains PROMPT-build.md, PROMPT-plan.md, executable loop.sh, LEARNINGS.md, and logs/
- [ ] Build prompt embeds the confirmed backpressure commands and any `--phase N` scope clause
- [ ] loop.sh exits distinctly on: completion sentinel (0), iteration cap (1), blocked sentinel (2), stall (3)
- [ ] LEARNINGS.md is never overwritten on regeneration
- [ ] Harness committed to git (unless planning commits disabled)
- [ ] User told how to run it and warned to sandbox + cap iterations
</success_criteria>
