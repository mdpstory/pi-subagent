# plan — pi-subagent rework

Companion to `~/Notes/.pi/extensions/pi-workflow/plan.md`. That plan's whole P0
milestone is blocked on this repo. Scope here: make `subagent` a correct
process-spawner for a role-based workflow — identity propagation, recursion
safety, and removal of the duplicated permission logic that currently
contradicts pi-workflow.

Current state: fork of the upstream `subagent` example, plus live TUI streaming
(`SUBAGENT_LIVE_DESIGN.md`) and a `wf_write_artifact` tool bolted on.

---

## Milestone S0 — identity propagation (blocks all of pi-workflow P0)

### S0-1 — `env` passthrough on spawn
`index.ts:488` spawns with no `env` option, so children inherit the director's
environment verbatim. Passing `PI_WORKFLOW_ROLE=engineer` in the task *text* is
prose to the LLM, not an env var. Every subagent therefore runs as the director.

- `TaskItem` (`index.ts:625`), `ChainItem` (`:632`), `SubagentParams` (`:644`)
  each gain:
  ```ts
  env: Type.Optional(Type.Record(Type.String(), Type.String(), {
    description: "Extra environment variables for the spawned agent process"
  }))
  ```
- `runSingleAgent` (`:403`) gains an `env` parameter, threaded from all three
  call sites (single `:884`, parallel `:841`, chain `:768`) exactly like `cwd`
  and `model` already are.
- Spawn becomes:
  ```ts
  spawn(invocation.command, invocation.args, {
      cwd: cwd ?? defaultCwd,
      shell: false,
      stdio: ["ignore", "pipe", "pipe"],
      env: { ...process.env, ...(env ?? {}) },
  });
  ```
- Precedence: explicit `env` > inherited. Never the reverse.
- AC: agent whose task is `echo $PI_WORKFLOW_ROLE` invoked with
  `env: { PI_WORKFLOW_ROLE: "engineer" }` reports `engineer`, while the parent
  session's own env is unchanged.

### S0-2 — declared workflow role per agent
Agent names in `agents/` already mirror workflow roles (`planner`, `scout`,
`architect`, `engineer`, `reviewer`, `qa`, `documenter`, `director`). Relying on
the caller to remember `env: { PI_WORKFLOW_ROLE: ... }` every dispatch is a
convention, and conventions are exactly what failed here.

- Agent frontmatter gains an optional `workflowRole:` field (`agents.ts` parser).
  Set it on the eight role agents; leave unset on `worker`.
- Resolution when spawning: explicit `params.env.PI_WORKFLOW_ROLE` wins;
  otherwise if the agent declares `workflowRole`, inject it.
- Deliberately **not** derived from `agent.name` implicitly — an agent must opt
  in, so a future non-workflow agent named `qa` doesn't silently acquire QA
  write permissions.
- AC: `subagent({ agent: "engineer", task })` with no `env` still yields
  `PI_WORKFLOW_ROLE=engineer` in the child.

### S0-3 — workflow id propagation
- When the parent has `PI_WORKFLOW_ID` set, or `.workflow/.active-id` resolves,
  inject it into the child unless the caller overrides.
- This is what makes parallel directors safe (pi-workflow P0-3): the child no
  longer depends on a single-pointer marker file that a second director can
  clobber between spawn and first tool call.
- AC: two directors with distinct `PI_WORKFLOW_ID`s spawn engineers
  concurrently; each child reports its own parent's id.

### S0-4 — recursion depth guard
Once env passes through, a subagent inherits everything — including whatever
made it a subagent. An agent with the `subagent` tool can now spawn agents that
spawn agents, with no natural termination and no cost ceiling.

- Inject `PI_SUBAGENT_DEPTH = (parent depth ?? 0) + 1`.
- Refuse to spawn above a configurable max (default 2) with a clear error naming
  the chain.
- Also inject `PI_SUBAGENT_PARENT_AGENT` for the error message and for the
  transcript.
- AC: depth-3 spawn is rejected, error names the full agent chain.

---

## Milestone S1 — kill the duplicated permission model

`wf_write_artifact` (`index.ts:1285`) re-implements pi-workflow's role rules in a
second table (`ROLE_ARTIFACT_ALLOW`, `:1274`) with a "keep in sync" comment. They
are already out of sync, in ways that break the workflow today:

| # | defect | effect |
|---|---|---|
| S1-a | `director: new Set(ARTIFACT_MDS)` — **full access** | Director can write `plan.md`, `research.md`, `review.md`, `test-report.md`, `changelog.md` through this tool, all of which pi-workflow's `ROLE_ALLOW.director` deliberately blocks. The write-gate is routed around entirely. |
| S1-b | `architecture.md` is written to `.workflow/<id>/artifacts/` | pi-workflow reads it from `.workflow/shared/artifacts/`. Architect writes to the wrong path → `wf_stage_complete architecture` reports "artifact missing or stub" forever. |
| S1-c | `progress.md` present here, absent from pi-workflow's `ARTIFACT_MDS` | An artifact nothing scaffolds, gates, or reads. |
| S1-d | unknown / unset role falls back to `director` = full access | Mirrors pi-workflow's old implicit-director default, which pi-workflow P0-2 is removing. Fail-open on identity. |
| S1-e | tool bypasses pi-workflow's `tool_call` hook entirely | No CLR gate, no cross-namespace check, no tool-ceiling accounting. Writing `plan.md` while a CLR is open is blocked via `write` but allowed via `wf_write_artifact`. |

Decision: **`wf_write_artifact` does not belong in this repo.** Permission and
namespace policy must live in exactly one place — pi-workflow — or it will
diverge again by construction.

- S1-1 — move the tool into pi-workflow's `index.ts`, where it reuses the real
  `isPathAllowedForRole`, `artifactPath` (which routes `architecture.md` to the
  shared dir), and the CLR gate. Nothing is re-declared.
- S1-2 — delete `ARTIFACT_MDS`, `ROLE_ARTIFACT_ALLOW`, and the tool registration
  from this repo. Keep `withFileMutationQueue` if the live-output code uses it;
  otherwise drop.
- S1-3 — coordinate with pi-workflow C9: one canonical definition, reinstall both
  extensions from source, verify no duplicate tool name is registered (two
  extensions registering `wf_write_artifact` is undefined behaviour).
- AC: after the move, architect writing `architecture.md` lands in
  `.workflow/shared/artifacts/`; director writing `plan.md` is denied; writing
  any artifact while a CLR is open is denied.

---

## Milestone S2 — observability for the workflow

The workflow's cost and correctness are both invisible right now.

### S2-1 — surface per-agent usage
`SingleResult.usage` already carries `input`/`output`/`cacheRead`/`cacheWrite`/
`cost`/`turns`. Include a compact per-agent line in the tool result text, not
only in `details`, so the director sees the price of each dispatch and
pi-workflow P4 (token diet) has a baseline to measure against.

### S2-2 — propagate exit reason distinctly
Distinguish "agent finished", "agent hit its own tool ceiling", "aborted by
ESC", and "crashed". Today all non-zero paths look alike to the director, which
makes it retry things it should escalate.

### S2-3 — `--no-session` implication, documented
Children run with `--no-session` (`:432`), so they are **not** addressable
interactive sessions: `intercom` cannot reliably reach them and their transcript
dies with the process. This is the concrete reason pi-workflow P2 replaces
inter-agent `intercom` with a file-backed bus. Record it in `README.md` so the
next person doesn't re-attempt intercom-based coordination.

---

## Milestone S3 — live output + interrupt

Land `SUBAGENT_LIVE_DESIGN.md` (streaming overlay + ESC abort) as its own
milestone. Independent of S0–S2; do not let it block the blockers.

- Verify `AbortSignal` actually kills the child process tree, not just the
  parent's promise — a detached `pi` child that survives abort keeps writing
  artifacts after the director moved on, which corrupts stage state.
- AC: ESC during a long engineer run terminates the child within ~1s and no
  further writes to `.workflow/` occur afterward.

---

## Sequencing

```
S0-1 ─ S0-2 ─ S0-3 ─ S0-4   ← unblocks pi-workflow P0 entirely
   └─ S1 (needs S0-1 for the role to even be correct)
S2 — independent
S3 — independent
```

Ship S0-1 first and alone; it is a ~15-line change and everything else waits on
it.

## Non-goals

- No change to agent discovery, scoping, or the `agents/` format beyond the
  single `workflowRole` field.
- No re-implementation of workflow state here. This repo spawns processes and
  reports results; it does not own policy.
- No replacement of `intercom` — S2-3 only documents why it can't serve
  agent↔agent traffic.
