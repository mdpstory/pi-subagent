# Summary — S0 identity propagation implemented

Scope: `plan.md` Milestone **S0 — identity propagation** only (S0-1 through
S0-4, plus the agent-frontmatter part of S0-2). Did **not** touch S1/S2/S3,
and did **not** proceed into the companion `pi-workflow` plan
(`~/Notes/.pi/extensions/pi-workflow/plan.md`) — stopping here per instruction.

## Files changed

- `index.ts`
- `agents.ts`
- `agents/architect.md`, `agents/director.md`, `agents/documenter.md`,
  `agents/engineer.md`, `agents/planner.md`, `agents/qa.md`,
  `agents/reviewer.md`, `agents/scout.md`

Same changes were also copied into the actually-loaded install at
`~/.pi/agent/git/github.com/mdpstory/pi-subagent/` (index.ts, agents.ts, and
the 8 agent `.md` files), since that's the copy `pi` resolves the `subagent`
tool from, not this working repo.

## What was done

**S0-1 — `env` passthrough on spawn**
- `TaskItem`, `ChainItem`, `SubagentParams` each gained an optional
  `env: Record<string,string>` field (new `AgentEnvSchema`).
- `runSingleAgent` gained an `extraEnv` parameter, threaded from all three
  call sites (chain, parallel, single) in the same position as `model`.
- `spawn(...)` now passes `env: childEnv` instead of implicitly inheriting
  `process.env` only.

**S0-2 — declared workflow role per agent**
- `AgentConfig` gained optional `workflowRole?: string`.
- `agents.ts` frontmatter loader now reads `workflowRole:` from agent `.md`
  frontmatter (generic key, no schema change needed there).
- Added `workflowRole: <name>` frontmatter to the 8 role agents (architect,
  director, documenter, engineer, planner, qa, reviewer, scout). `worker.md`
  deliberately left unset — opt-in only, not derived from `name`.
- Resolution order implemented in new `buildChildEnv()`: explicit
  `params.env.PI_WORKFLOW_ROLE` (if caller passes it) overrides the
  agent-declared `workflowRole`, which overrides nothing (no other source).

**S0-3 — workflow id propagation**
- `buildChildEnv()` resolves `PI_WORKFLOW_ID` from (in order) the parent
  process's own env, then `.workflow/.active-id` under the spawn's `defaultCwd`
  (new `readActiveWorkflowId()` helper), and injects it into the child unless
  the caller's explicit `env` already sets it.

**S0-4 — recursion depth guard**
- New env vars threaded through every spawn: `PI_SUBAGENT_DEPTH`,
  `PI_SUBAGENT_PARENT_AGENT`, `PI_SUBAGENT_SELF_AGENT`, `PI_SUBAGENT_CHAIN`.
- Default max depth 2 (`DEFAULT_MAX_SUBAGENT_DEPTH`), overridable via
  `PI_SUBAGENT_MAX_DEPTH` on the root process.
- Depth check runs *before* spawning; a violation returns a normal failed
  `SingleResult` (no process spawned) with an error naming the full chain,
  e.g. `Refused: subagent recursion depth exceeded (max 2). Chain: director > engineer > worker`.

**Precedence, all layers combined (low → high):**
`process.env` (inherited) → depth/chain bookkeeping → agent-declared
`workflowRole` → auto-resolved `PI_WORKFLOW_ID` → caller-supplied `env`
(always wins, can override anything above).

## Verified

- `node --experimental-strip-types --check` clean on `index.ts` and
  `agents.ts` (both copies).
- `tsc --noEmit` was attempted but the repo has no local `@types/node` /
  module resolution config — all reported errors are pre-existing
  environment-setup noise (`Cannot find name 'process'`, etc.), unrelated to
  this change; not fixed, out of scope.
- No other file in the repo calls `runSingleAgent`, `TaskItem`, or
  `ChainItem` directly, so the positional-argument signature change is
  contained.

## Not done (explicitly out of scope this session)

- **S1** — moving `wf_write_artifact` / deleting the duplicated
  `ROLE_ARTIFACT_ALLOW` table out of this repo. Still present, still
  duplicated, still the S1-a..S1-e defects described in `plan.md`.
- **S2** — usage-line surfacing, exit-reason distinction, `--no-session`
  README note.
- **S3** — live-output abort-kills-child-tree verification.
- The companion `pi-workflow` repo's `plan.md` (P0 etc.) — not started.

## Suggested next step

Runtime smoke test: spawn `engineer` via `subagent({agent:"engineer", task:"echo $PI_WORKFLOW_ROLE $PI_WORKFLOW_ID $PI_SUBAGENT_DEPTH"})`
from a live pi session and confirm the child reports `engineer`, a resolved
workflow id (or empty if none active), and `1`. Not run in this session — no
live session context to invoke the `subagent` tool through the actual `pi`
binary was available from inside this task.
