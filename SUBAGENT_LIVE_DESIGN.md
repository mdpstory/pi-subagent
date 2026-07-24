# Subagent Live Output + Interrupt Design

## Requirement
Show real-time subagent execution in a TUI overlay with an interrupt button (ESC to cancel). Main agent should see subagent output while it streams.

## Architecture

### Challenge
The `subagent` tool (in the official pi extension) spawns a subprocess and collects output asynchronously. By the time `tool_result` fires, the subagent has already finished. Interrupting mid-execution requires:

1. **Spawning** the subprocess
2. **Streaming** output to a TUI overlay in parallel
3. **Accepting** keyboard input to trigger abort
4. **Propagating** abort via `AbortSignal` to kill the subprocess
5. **Returning** the result to the LLM

### Approach A: Tool Wrapper (Recommended)
Create a new tool `subagent-live` that replicates the subagent logic but integrates TUI display:

```typescript
registerTool({
  name: "subagent-live",
  execute: async (toolCallId, params, signal, onUpdate, ctx) => {
    const state = { lines: [], isRunning: true };
    let overlayHandle: any;
    
    // Show overlay before spawning
    const showUI = ctx.ui.custom(
      (tui, theme, _, done) => {
        const overlay = new LiveOverlay(state, () => {
          proc.kill("SIGTERM");
          overlayHandle?.hide?.();
        }, theme);
        return { render, handleInput, invalidate };
      },
      { overlay: true, overlayOptions: { width: "65%", anchor: "center" } }
    );

    // Spawn subagent while UI shows
    const proc = spawn("pi", piArgs);
    let output = "";

    proc.stdout.on("data", (chunk) => {
      output += chunk;
      state.lines.push(...chunk.toString().split("\n"));
      // Overlay rerenders automatically on state change
    });

    // Wait for process + overlay to close
    const exitCode = await waitProc(proc);
    state.isRunning = false;
    overlayHandle?.hide?.();
    
    return { content: [{ type: "text", text: output }] };
  }
});
```

**Pros:**
- No need to modify pi core
- Full control over subprocess lifecycle
- Overlay can show real-time updates via state mutations
- User can interrupt by hitting ESC

**Cons:**
- Duplicates subagent logic (maintenance burden)
- Requires spawning `pi` (not ideal but works)

### Approach B: Hook Tool Execution
Extend pi to fire events during tool execution:

```typescript
// In pi core (hypothetical)
onToolExecution("subagent", {
  beforeStart: (params) => overlay.show(params),
  onOutput: (chunk) => overlay.addLine(chunk),
  onAbort: () => overlay.markAborted(),
  onComplete: () => overlay.close(),
});
```

**Pros:**
- Cleanest integration
- Works with any tool, not just subagent
- No duplication

**Cons:**
- Requires changes to pi core
- Not available now

### Approach C: Custom Tool + Orchestration
Create a tool that calls subagent but wraps the result:

```typescript
registerTool({
  name: "subagent-with-ui",
  execute: async (_, params, signal, onUpdate, ctx) => {
    // Call the real subagent tool via LLM
    const result = await ctx.callTool("subagent", params, signal);
    // By the time we get here, it's done - can't show live output
  }
});
```

**Cons:**
- Can't show live output (defeats purpose)
- Double round-trip through LLM

---

## Recommended Implementation

**Use Approach A** (Tool Wrapper). Here's the minimal spec:

### File: `~/.pi/agent/extensions/subagent-live.ts`

```typescript
import { spawn } from "node:child_process";
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Container, Text } from "@earendil-works/pi-tui";
import { matchesKey, Key } from "@earendil-works/pi-tui";

interface LiveState {
  agent: string;
  task: string;
  lines: string[];
  isRunning: boolean;
  isAborted: boolean;
}

class LiveOverlay {
  constructor(private state: LiveState, private onAbort: () => void, private theme: any) {}

  handleInput(data: string): void {
    if (matchesKey(data, Key.escape)) {
      this.state.isAborted = true;
      this.onAbort();
    }
  }

  render(width: number): string[] {
    const lines: string[] = [];
    lines.push(this.theme.fg("accent", "━ Live Subagent"));
    lines.push(this.theme.fg("muted", `Agent: ${this.state.agent}`));
    lines.push(this.theme.fg("dim", `Task: ${this.state.task.slice(0, width - 10)}`));
    lines.push("─".repeat(width - 2));
    
    for (const line of this.state.lines.slice(-8)) {
      lines.push("  " + this.theme.fg("toolOutput", line));
    }
    
    lines.push("─".repeat(width - 2));
    lines.push(
      (this.state.isRunning ? this.theme.fg("warning", "▪ Running") : this.theme.fg("success", "✓ Done")) +
      this.theme.fg("dim", " [ESC to cancel]")
    );
    
    return lines;
  }

  invalidate(): void {}
  addLine(text: string): void {
    this.state.lines.push(...text.split("\n").filter(l => l.trim()));
  }
}

export default function (pi: ExtensionAPI) {
  pi.registerTool({
    name: "subagent-live",
    label: "Subagent (Live UI)",
    description: "Run subagent with live output display and interrupt support",
    parameters: Type.Object({
      agent: Type.String(),
      task: Type.String(),
      cwd: Type.Optional(Type.String()),
    }),

    async execute(_toolCallId, params: any, signal, onUpdate, ctx) {
      if (!ctx.hasUI) return { content: [{ type: "text", text: "No UI" }] };

      const state: LiveState = {
        agent: params.agent,
        task: params.task,
        lines: [],
        isRunning: true,
        isAborted: false,
      };

      let proc: any;
      let overlayHandle: any;

      // Show overlay (non-blocking)
      const uiPromise = ctx.ui.custom(
        (tui, theme, _, done) => {
          const overlay = new LiveOverlay(state, () => {
            if (proc) proc.kill("SIGTERM");
            overlayHandle?.hide?.();
          }, theme);

          return {
            render: (w: number) => overlay.render(w),
            invalidate: () => overlay.invalidate(),
            handleInput: (data: string) => {
              overlay.handleInput(data);
              tui.requestRender();
            },
          };
        },
        {
          overlay: true,
          overlayOptions: { width: "65%", anchor: "center", maxHeight: "60%" },
          onHandle: (h) => { overlayHandle = h; },
        }
      );

      // Spawn subagent
      const args = ["--mode", "json", "-p", "--no-session", `Task: ${params.task}`];
      let output = "";

      const procPromise = new Promise<string>((resolve, reject) => {
        proc = spawn("pi", args, { cwd: params.cwd ?? ctx.cwd });
        
        proc.stdout.on("data", (chunk: Buffer) => {
          const text = chunk.toString();
          output += text;
          state.lines.push(...text.split("\n").filter(l => l.trim()));
        });

        proc.on("close", (code: number) => {
          resolve(output);
        });

        proc.on("error", reject);
      });

      try {
        const result = await procPromise;
        state.isRunning = false;
        overlayHandle?.hide?.();

        return { content: [{ type: "text", text: result || "(no output)" }] };
      } catch (err: any) {
        return {
          content: [{ type: "text", text: `Error: ${err.message}` }],
          isError: true,
        };
      }
    },
  });
}
```

### UX Flow

1. User asks: `"use subagent-live to scout the codebase"`
2. Pi calls `subagent-live` tool
3. Extension spawns overlay (60% width, centered)
4. Extension spawns subprocess with task
5. As subprocess outputs, overlay updates in real-time (last 8 lines visible)
6. User can press ESC to abort (kills subprocess, closes overlay, returns error)
7. When done, overlay closes, result returned to LLM

### Testing

```bash
pi --extension ~/.pi/agent/extensions/subagent-live.ts \
  "use subagent-live to check this repo structure"
```

In TUI mode, should see:
- Centered overlay showing "Live Subagent"
- Agent name and task
- Lines of output as they arrive
- Status (Running / Done)
- "[ESC to cancel]" hint

---

## Known Limitations

1. **Subprocess spawning** - subagent-live spawns `pi` as a child process. This works but is not ideal for performance.
2. **Output buffering** - Only last 200 lines kept in memory. For very long-running agents, some output may scroll off.
3. **No progress indication** - Only shows final output, not intermediate tool calls or reasoning.
4. **Duplication** - Logic mirrors the official subagent tool. If subagent changes, this must be maintained.

---

## Future: Pi Core Integration

If pi adds `onToolStream` or similar hook:

```typescript
pi.on("toolStream", (event, ctx) => {
  if (event.toolName === "subagent") {
    ctx.ui.showOverlay({
      render: () => createLiveOverlay(event),
      onAbort: () => event.signal.abort(),
    });
  }
});
```

This would eliminate the need for a wrapper tool.
