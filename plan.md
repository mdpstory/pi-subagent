# plan — merge pi-subagent into pi-workflow

Goal: one package. Move `pi-subagent`'s `subagent` tool (and its agent
personas/prompts) into `~/Notes/.pi/extensions/pi-workflow/`, then retire this
repo. Not implemented yet — this is the plan only.

Scope discipline: this is a **move**, not a refactor. pi-workflow's existing
`index.ts` is not touched except to add one import line.

## Why merge (confirmed by reading both trees)

- `pi-workflow/index.ts` already has its **own** `wf_write_artifact` (role +
  CLR gated, routes shared artifacts to `.workflow/shared/`) — strictly
  better than `pi-subagent`'s copy (whitelist-only, no CLR gate, no shared-
  artifact routing). The comment at `pi-workflow/index.ts:1189` even flags
  this exact divergence ("registered by some installed copy but absent from
  this canonical index.ts") — `pi-subagent` is that installed copy.
- `pi-workflow/NOTE-subagent-env.md` documents `env` passthrough as a hard
  blocker for pi-workflow's role model. `pi-subagent/index.ts` already
  implements it (`buildChildEnv`, `RESERVED_ENV_KEYS`, depth ceiling) — this
  is the missing piece pi-workflow needs and doesn't have yet.
- `pi-subagent/agents/*.md` (architect/director/documenter/engineer/planner/
  qa/reviewer/scout/worker, with `workflowRole:` frontmatter) are symlinked
  individually from `~/.pi/agent/agents/*.md` (director.md deliberately not
  linked — director isn't dispatched as a subagent). These are load-bearing:
  `wf_stage_start`'s DELEGATE hint tells the director to call
  `subagent({ agent: "<role>", ... })`, which only resolves if these files
  are discoverable. Must survive the move with symlinks repointed.
- Two separate installed packages today (`git:github.com/mdpstory/pi-subagent`
  and `git:github.com/mdpstory/pi-workflow` in `~/.pi/settings.json`) doing
  one job.

## Target layout — `~/Notes/.pi/extensions/pi-workflow/`

```
index.ts                  # unchanged, plus one import of subagent/tool.ts
subagent/
  agents.ts               # = pi-subagent/agents.ts, unchanged (discovery)
  format.ts               # formatTokens/Usage/ToolCall, langForPath,
                          # simpleDiffLines, extractPartialStringField,
                          # renderToolCallDetail/LiveBlock
  run.ts                  # buildChildEnv, RESERVED_ENV_KEYS, runSingleAgent,
                          # getPiInvocation, writePromptToTempFile,
                          # mapWithConcurrencyLimit, types (SingleResult,
                          # UsageStats, LiveToolCall, SubagentDetails)
  tool.ts                 # registerSubagentTool(pi): execute/renderCall/
                          # renderResult, calls run.ts + format.ts
agents/                   # moved from pi-subagent/agents/*.md as-is
prompts/                  # moved from pi-subagent/prompts/*.md as-is
skills/                   # already exists (wf-*), unchanged
lib/, tools-*.ts          # NOT created. see "deferred".
package.json              # add "prompts": "./prompts" to "pi" field
README.md                 # append subagent section
```

Only edit to `index.ts`:
```ts
import installSubagentTool from "./subagent/tool.ts";
// ... inside the existing default export, one added line:
installSubagentTool(pi);
```

The `subagent/` split is free — it is genuinely separate code arriving from a
separate repo, so it lands as modules rather than being pasted inline.

## Steps

1. **Extract `pi-subagent` internals into `subagent/{agents,format,run,tool}.ts`**
   under the pi-workflow tree, preserving all documented behavior/comments
   (identity/env security notes, live-streaming throttling, output caps,
   depth ceiling). Drop `pi-subagent`'s `wf_write_artifact` registration
   entirely — pi-workflow's stays canonical.
2. **Wire it**: add the single import + `installSubagentTool(pi)` call to
   `pi-workflow/index.ts`. No other change to that file.
3. **Move `agents/*.md` and `prompts/*.md`** from `pi-subagent/` into
   `pi-workflow/agents/` and `pi-workflow/prompts/`, unchanged. Add a
   one-line comment at the top of `agents/director.md` explaining why it is
   present but deliberately not symlinked (director is not dispatched as a
   subagent) — otherwise it reads as an oversight.
4. **Repoint symlinks**: `~/.pi/agent/agents/{architect,documenter,engineer,
   planner,qa,reviewer,scout,worker}.md` currently point into
   `pi-subagent/agents/`; recreate them pointing into
   `pi-workflow/agents/`. First check whether pi's `getAgentDir()` discovery
   accepts a single symlinked *directory* — if so, use one dir symlink
   instead of eight file symlinks (but then `director.md` becomes visible, so
   only do this if director being discoverable is harmless; otherwise keep
   the eight explicit links).
5. **Update `pi-workflow/package.json`**: add `"prompts": "./prompts"` to
   the `"pi"` field (agents dir has no manifest field — same as today,
   discovery is via `getAgentDir()`/symlink only).
6. **Verify single-sourced `wf_write_artifact`**: confirm whether pi *errors*
   on duplicate tool registration or silently last-wins. If last-wins, a
   passing smoke test proves nothing — assert explicitly that the live
   `wf_write_artifact` is pi-workflow's (has the CLR gate and
   `.workflow/shared/` routing).
7. **Smoke test**: run pi in this repo, confirm `subagent` (single, parallel,
   chain modes; `env` passthrough; depth ceiling), all `wf_*` tools, and the
   DELEGATE hint (`wf_stage_start`) still work; confirm agent persona
   symlinks resolve to every role name the DELEGATE hint can emit.
8. **Retire `pi-subagent`**: remove its `git:github.com/mdpstory/pi-subagent`
   entry from `~/.pi/settings.json` `packages`. Leave the repo itself as an
   archived pointer (README note: "merged into pi-workflow, see
   ~/Notes/.pi/extensions/pi-workflow/") rather than deleting the git repo
   outright — but no code stays wired into the running package.
9. **README merge** (last, non-blocking): keep pi-workflow's architecture
   notes, append subagent tool usage/modes docs from pi-subagent's README.

## Deferred — splitting `pi-workflow/index.ts`

An earlier draft of this plan also split pi-workflow's 1439-line `index.ts`
into `lib/{state,hooks,tools-workflow,tools-artifact,tools-knowledge,tools-bus}.ts`.
Deliberately dropped from this plan:

- It bundles two risky changes into one. If the smoke test fails, there is no
  way to tell whether the merge or the split caused it.
- There is no test suite. The only verification is manual poking, which is
  weak cover for a verbatim move of gating / CLR / retry logic.
- A single 1439-line file is not currently a problem: greppable, no
  cross-module imports, no shared-state wiring. Splitting it *creates* the
  `state.ts` circular-import problem that does not exist today.

If `index.ts` size actually becomes painful, split it later as its own commit,
on top of an already-proven merge.

## Explicit non-goals

- No behavior changes to gating logic, retry/CLR semantics, or the subagent
  spawn/env security model during the move. Any bug fixes found along the way
  get called out separately, not silently folded in.
- No restructuring of pi-workflow's existing code (see "deferred").
- Not deleting the `pi-subagent` git repo/history — just deregistering it as
  an active package.
