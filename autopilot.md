Execute the specified plan or task with minimal user interaction. Work autonomously until complete or blocked.

**Requires the `/save` command.** `/autopilot` checkpoints and commits exclusively through `/save` (Step B); it does not commit or checkpoint on its own.

## Non-negotiable principles

**Accuracy over speed, every time.** Autonomy is not permission to cut corners. The reason `/autopilot` exists is to clear backlog while the operator is away, but a wrong deliverable left unattended is worse than no deliverable. If a step needs more time to verify, take the time. Never declare something "done" without verifying it works end to end. Never call a test "flaky" without root-cause investigation. Never skip a phase audit checkpoint. When the operator returns, every completed item must be genuinely complete.

**Test-Driven Development is the default.** For any code change that has testable behavior (new functions, bug fixes, API endpoints, business logic, parsing, validation):

1. **Write the failing test first.** Capture the expected behavior in a test before writing the implementation. Run it; confirm it fails for the right reason (not a compile error, not a missing import).
2. **Write the minimum implementation to make the test pass.** No speculative extras.
3. **Run the full relevant test suite.** Confirm the new test passes and no existing tests regressed.
4. **Refactor with the test as the safety net.** Only after green.

TDD exceptions (be honest with yourself; these are narrow):

- Pure config / infra / docs changes with no executable behavior.
- Exploratory spikes explicitly scoped as throwaway (delete the spike before committing real code).
- UI-only visual tweaks where a Playwright/E2E test would be more cost than value; still verify in a running browser per CLAUDE.md.

If you skip TDD, state why in the plan-status update for that phase. "I wrote the implementation first because…": the reason has to be defensible.

**Locally complete vs DONE: autopilot ships locally complete, not DONE.** The full lifecycle is `OPEN → IN PROGRESS → COMMITTED → CI GREEN → DEV E2E GREEN → STAGE E2E GREEN → DONE` (or the project's equivalent in CLAUDE.md). Autopilot's job is to take items from `OPEN` to `COMMITTED`. Promotion to `CI GREEN` and beyond is the operator's responsibility: they push, CI runs, they grade. Autopilot NEVER marks anything DONE.

Before marking any phase COMMITTED (the deepest autopilot status):

- Every file referenced in the plan exists and contains real implementation (no stubs, no `TODO: implement`, no `throw new Error("not implemented")`).
- Every new endpoint is registered in the router and reachable.
- Every new migration is numbered correctly and applies cleanly.
- Every new dependency is in the lockfile.
- Every new test passes locally with the same command CI will run.
- Lint passes on every changed file.
- Build passes.
- `openapi.yaml` (or equivalent spec) is updated if any REST/API surface changed.
- ADA/a11y obligations are met if UI changed (vitest-axe or equivalent ran clean).
- Docs reflect the change (CLAUDE.md, README sections, runbooks).

A half-finished phase blocks every downstream phase. Finish it locally before committing. But "locally finished" is not the same as "deployed and verified": the gap between them is the operator's promotion path, not autopilot's claim of done-ness.

## Instructions

0. **Verify the `/save` dependency, then set the autopilot-running flag.** As the very first action of `/autopilot`, confirm `/save` exists (`test -f ~/.claude/commands/save.md`); if it is missing, ABORT (the loop's checkpoint and commit both run through `/save`). Then write the current session ID to the autopilot flag file. **Location:** `~/.claude/projects/<slug>/autopilot.state`, where `<slug>` is the current working directory with `/` replaced by `-` and a leading `-` prepended (on Windows, replace `\` and `:` with `-` as well). This is the SAME `<slug>` directory that holds `memory/MEMORY.md`; the flag file is a sibling of the `memory/` subdirectory. The file content is exactly the session ID, no trailing newline, no JSON wrapping. Find the session ID the same way `/save` Section 8 does (UUIDs in `/tmp/claude/...` paths or `output-file` fields of task notifications). This flag lives OUTSIDE the project tree on purpose: it's Claude-internal runtime state, not project content, and doesn't need a `.gitignore` entry. `/save` Section 9 reads it to decide whether to continue autopilot after each checkpoint.

   **What counts as `/autopilot` invocation for flag-setting purposes:** the literal `/autopilot ...` slash command (always), AND operator phrases that clearly re-arm a previously-stopped autopilot loop ("resume autopilot", "start autopilot again", "back to autopilot", "keep autopilot going" after a confirmed stop). Setting the flag in those cases is the same Step 0 write: current session ID, same file path. If unsure whether a phrase re-arms, ask once.

1. **Parse the argument**: The text after `/autopilot` is your task. It may be a plan document path, a backlog item reference, or a free-form description. **If no argument is provided**, check the project's CLAUDE.md and memory for the backlog record it designates (a tracking doc like `TODO.md` or a deferred-items file, a backlog CSV, or an issue tracker / GitHub Project). Read it and pick the highest-priority item that has status **NOT STARTED** or **OPEN** (highest tier/priority first). Announce which item you're starting before beginning work. If no backlog document exists, ask what to work on.

   **Plan first, ALWAYS.** Before writing any code for the picked item: (a) if the backlog row links to an existing plan doc, read it; (b) if no plan exists, write a short inline plan with five required parts: **scope, files to touch, tests to add, audit checkpoints, estimated effort**. The audit-checkpoints field is REQUIRED; every plan must specify runnable verification commands (`go build`, `go test`, `pnpm test`, `curl /healthz`, `grep -n "..."`, etc.) that Step 4(g) will execute later. A plan without audit checkpoints is not an actionable plan; either define them or treat the absence as a skip-gate. Then evaluate the plan: if it surfaces an operator-decision question, a design-ambiguity gap, a blocked dependency, or any other gate, **SKIP this item: do not attempt with best-judgment scope, do not stop autopilot.** Log one line in the autopilot session log (`Documents/autopilot_session.md` or equivalent; see Step 4(k) for the fallback path): `skipped <ID>: plan blocked on <reason>`. Then go pick the NEXT-priority item. Repeat until you land on an item whose plan is actionable, OR you exhaust the whole backlog (in which case Step 8's graceful-exit clause fires). See Step 8 for the full skip-don't-stop discipline.

2. **Load context**: Read the relevant plan document, CLAUDE.md, the project's backlog record, and any referenced docs. Check the project's runbook directory if one exists (`Documents/runbook/`, `docs/runbooks/`, `runbooks/`, or a runbook section in CLAUDE.md) for any runbooks related to the task (infra rebuild, deployment, Vault recovery, etc.). If the plan references infrastructure changes (RDS, ECS, Terraform, VPC), load the matching runbook before starting. Understand the full scope before writing code.

3. **Sprint integration**: If the project uses sprints (check for sprint board, Jira, Linear, GitHub Projects, or sprint tracking in CLAUDE.md/memory), set the work item to highest priority in the current sprint and assign it to the current operator before starting. If no sprint system is in use, skip this step.

4. **Work autonomously**: Execute the plan phase by phase (or step by step). For each phase, follow this strict order:
   - **(a) Write the failing test FIRST**: per the TDD principle above. Capture the expected behavior in a test; run it; confirm it fails for the right reason. If TDD does not apply to this phase (config/infra/docs only, or a narrow exception), state why in the plan status update before proceeding.
   - **(b) Write the implementation**: the minimum code/migrations/config changes to make the test pass. No speculative extras, no surrounding cleanup that wasn't asked for.
   - **(c) Lint changed files**: run the appropriate linter on every file changed in this phase. Match the linter to the file type using the project's configured tooling (e.g., commonly ESLint for .ts/.tsx/.js, golangci-lint for .go, markdownlint for .md, and whatever CSS linter the project uses; projects vary on whether they run stylelint, biome, or fold CSS lint into ESLint). Fix all lint errors before proceeding.
   - **(d) Run the project's build**: check CLAUDE.md or package.json/Makefile for the correct command.
   - **(e) Run the full relevant test suite**: not just the new test. Confirm the new test passes AND no existing tests regressed. Per CLAUDE.md: full project test suite after each feature.
   - **(f) Fix all errors and warnings**: including ones that appear unrelated to your changes. Zero errors, zero warnings is the bar. Do not suppress, do not skip, do not `// eslint-disable` your way past a real signal. Investigate root cause.
   - **(g) Run all audit checkpoints defined in the plan**: these are the runnable commands (`go build`, `go test`, `npm run build`, `curl`, `grep`) that prove the phase is complete. (Step 1's plan-first pass already required these to be defined; this sub-step is where they get executed.)
   - **(h) Follow any referenced runbook step by step.**
   - **(i) Audit before advancing**: verify the current phase is locally complete against the "Locally complete" checklist below. Read back the plan's deliverables and confirm each one exists, compiles, and passes tests. Don't advance on "it should work"; run the audit checkpoint commands and check the output. If anything is missing or failing, fix it before proceeding. Never advance with a known gap. This MUST happen before sub-steps (j) through (l).
   - **(j) Update plan status to COMMITTED, NOT DONE**: only after sub-steps (a)–(i) all pass, mark the phase `COMMITTED <date>, awaiting CI + E2E` in the plan doc (use the project's lifecycle vocabulary if it defines one; CLAUDE.md typically specifies `OPEN → IN PROGRESS → COMMITTED → CI GREEN → DEV E2E GREEN → STAGE E2E GREEN → DONE`). **DONE is reserved for "verified on the highest target environment for the change's scope": it is the OPERATOR's promotion, not autopilot's.** Autopilot never marks anything DONE. Marking DONE here would silently misrepresent CI status; treat it as a hard error. Also refresh `.claude/session_state.md` with current progress (for crash recovery).
   - **(k) Maintain session summary**: keep a running summary in the project's `Documents/autopilot_session.md` (or equivalent project-tracking location if `Documents/` doesn't exist, fall back to repo root if needed). Within a single `/autopilot` run, APPEND entries: completed phases get a timestamp + what was done + tests added + issues; skipped items (per Step 1 / Step 8) get one line `skipped <ID>: <reason>`. A fresh `/autopilot` invocation overwrites the file: it's a live per-run log, not a cross-run history.
   - **(l) Leave the commit to `/save`**: after the phase passes its audit checkpoint and is marked COMMITTED, do NOT commit here. `/save` (Step B below) is the single commit point: it commits the deliverable's work together with its own checkpoint outputs as one logical commit per deliverable. (`/save` Section 6 covers commit-vs-push policy; never push.)

5. **Stop and ask ONLY when (mid-deliverable, after the item has been planned and started):**
   - Destructive infrastructure operations need confirmation (RDS recreation, VPC destroy, etc.)
   - A test failure that can't be diagnosed after 2 attempts (do NOT mark the test as "flaky" and move on; investigate root cause per CLAUDE.md, and if you genuinely cannot resolve it, stop and ask)
   - The plan reveals a gap or contradiction AFTER work has begun (a phase's TDD test would require operator clarification on expected behavior that wasn't visible during planning, etc.)
   - A decision is needed mid-implementation that wasn't anticipated and isn't covered by existing conventions

   **NOT this list (skip instead per Step 8):**
   - The Step 1 planning pass surfaces an operator-decision question: **skip the item**, pick next, don't stop autopilot.
   - The Step 1 planning pass surfaces design ambiguity: **skip the item**, pick next.
   - The Step 1 planning pass shows the item is blocked on missing infra / unmerged dependency: **skip the item**, pick next.
   - Pre-flight discovery of any gate before code is written = skip, not stop.

   The distinction: gates discovered DURING PLANNING route to skip-don't-stop. Gates discovered AFTER work has begun route to stop-and-ask (because rollback of partial work has a cost).

6. **Do NOT stop to ask about:**
   - Code style, naming, or patterns: follow existing conventions
   - File organization: follow existing project structure
   - Whether to write a test first: always write it first when TDD applies (see principle above)
   - Whether to run tests: always run them, full suite, after every phase
   - Whether to update docs: always update them
   - Whether to update `openapi.yaml` (or equivalent spec): always update when REST/API endpoints change

7. **On completion of each deliverable (MANDATORY):**

   **Step A: Start the auto-resume timer.** Launch a background Bash command with `run_in_background: true` running `sleep 300 && echo "5-min autopilot timer elapsed. Resume next backlog item."` (5-minute default). This is the enforced wake-up signal; without it the auto-resume loop does not work (Claude defaults to stopping at deliverable boundaries). Do NOT use `ScheduleWakeup`; that tool is scoped to `/loop` dynamic mode only.

   **Step B: Run `/save`.** Checkpoint the session via the save skill immediately after the timer starts. `/save` Section 9 owns the rest of the cycle: it reads the autopilot flag, waits for the still-in-flight timer if needed, then dispatches into Step D below. **You do NOT wait for the timer yourself**: that's `/save` Section 9's job. Include a summary of what was done in this deliverable in the `/save` arguments. **`.claude/*` writes are deferred under autopilot:** any update targeting a `.claude/`-path file (global skills `~/.claude/skills/*`, global CLAUDE.md, command files, AND project-tree `<cwd>/.claude/skills/*`) may permission-prompt and stall the loop, so `/save` Section 3 routes those to `<cwd>/Documents/pending_skill_updates.md` instead, for the operator to apply on return.

   **Step D: Pick next backlog item.** Invoked by `/save` Section 9 once the timer has fired (case (b)) OR immediately if no timer is in flight (case (c)). Scan the project's backlog record (per CLAUDE.md) for the highest-priority NOT STARTED item. **Apply the Step 1 plan-first pass to that item.** If planning reveals a gate, skip the item per Step 8 and immediately repeat Step D with the next-priority item. Repeat the skip-and-continue loop within Step D until an actionable plan lands or the whole backlog is exhausted (in which case Step 8's graceful-exit clause fires). Once an actionable item is found, execute Step 4 for it, then return to this cycle (Step A again).

   **There is no Step C.** The previous version of this skill had a "wait for the timer" sub-step, but `/save` Section 9 now does the waiting internally; keeping a Step C here was redundant documentation that misrepresented the actual control flow.

8. **Flag lifecycle: clearing must be RARE.** The flag MUST reflect reality at every moment of the session. The bar for clearing is high, the bar for keeping the flag set is low. When in doubt, leave the flag set: one extra cycle is cheap to undo; missing a continuation requires the operator to re-invoke `/autopilot` from scratch.

   **Plan-first, skip-don't-stop:** In autopilot, every Step D pick goes through a planning pass BEFORE writing code. Use the item's existing plan document if one exists (linked from the backlog row). If no plan exists, write one inline: scope, files to touch, tests to add, audit checkpoints, estimated effort. **If the plan reveals operator-decision questions, design ambiguity, missing infra, or any other gate**: SKIP the item, log one line in the autopilot session log (Step 4(k) path), and pick the next-highest-priority item. The flag STAYS SET. Skip-and-continue is the default for every gated item; graceful exit only triggers when every item in the backlog has been planned AND skipped this pass. (Some projects also encode this rule in a project-scoped `feedback_*.md` memory file; honor those if present.)

   **Claude clears the flag ONLY when:**

   - **Graceful exit: every backlog item planned and skipped THIS PASS.** Step D has scanned the whole backlog and every remaining item, after a planning pass, has come back gated (operator-decision, design ambiguity, blocked infra, customer-signal-required). Clear the flag, announce which items were skipped and why ("backlog has N items all gated: X needs DSL design, Y blocked on demo-form backend, Z waiting on customer signal"), then stop. **Do NOT graceful-exit because an item LOOKS hard or unclear**: plan it first, and if the plan is fine, do the work. The bar is "no item has an actionable plan," not "no item looks easy."
   - **Explicit operator stop intent.** The operator's message contains unambiguous stop language: "stop autopilot", "cancel autopilot", "end autopilot", "that's enough autopilot", "pause autopilot", "exit autopilot", "stop", "halt", or equivalent. Clear the flag immediately and acknowledge. Ambiguous messages ("hmm", "wait", "hold on for a sec to check X", a quick course correction, a clarification, a feedback rule, an unrelated question) do NOT count as stop intent: the flag STAYS SET and autopilot resumes after the interruption is addressed.

   **Claude does NOT clear the flag when:**

   - One item is gated → skip it, pick the next, flag stays set.
   - Operator asks a one-off question that doesn't address autopilot itself.
   - Operator corrects a specific deliverable's approach but doesn't say to stop.
   - Operator pauses to verify something Claude just did.
   - A deliverable hits an unexpected error: fix the error and continue; clearing is for "no more work to do," not "this work is hard."
   - You're uncertain whether the operator meant stop. Ask once if needed; meanwhile keep the flag set.

   **Resuming after an interrupt that didn't clear the flag:** as soon as the operator's interruption is addressed, return to the autopilot cycle. If a timer was already running and fired during the interruption, treat that as case (c) in `/save` Section 9: proceed directly to Step D. If no timer is running because none was started yet (interrupted mid-deliverable), continue the deliverable, then resume the normal 4-step cycle.

   **Re-arming after a stop:** the operator types `/autopilot` (or "resume autopilot" / "start autopilot again") to set the flag back. Re-invocation is the only mechanism; there is no implicit "stop is reversible by default."

   Once cleared, `/save` Section 9 will correctly skip the continuation step and not invent more work. If the flag is left set in error, the next `/save` will continue autopilot one more cycle, and the operator can stop it then. That asymmetry is intentional: a wrongly-still-running flag is a small annoyance; a wrongly-cleared flag silently ends a long-running task.

9. **Notifications.** Send the operator a toast notification (native OS toast on macOS and Windows; Linux via `notify-send`) when the autopilot run ENDS (graceful exit / flag cleared per Step 8) or when it is blocked and needs the operator. A per-item toast is usually noise; what the operator wants to know is that the loop has stopped.

## Example usage

```
/autopilot path/to/Implementation_Plan.md Phase 1
/autopilot the next open backlog item
/autopilot fix the 3 remaining test-coverage gaps (TEST-008, 009, 010)
```

$ARGUMENTS
