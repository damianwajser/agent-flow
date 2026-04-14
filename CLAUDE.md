# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Agent Flow

Real-time visualization of Claude Code agent orchestration. It renders an interactive node graph showing agents, tool calls, subagent branching, and coordination as they happen. Originally built by Simon Patole for [CraftMyGame](https://craftmygame.com), forked under Apache 2.0.

Three distribution targets share the same core code:
- **Standalone web app** (`pnpm run dev`) — Next.js app + event relay server
- **VS Code extension** (`extension/`) — embeds the web UI as a webview
- **npx CLI** (`app/`) — `npx agent-flow-app` bundles relay + static export

## Build & Development Commands

```bash
pnpm i                    # install all workspace dependencies
pnpm run setup            # one-time: install Claude Code hooks (~/.claude/settings.json)
pnpm run setup -- --force # re-install hooks even if already configured

# Development
pnpm run dev              # start relay server + Next.js web app (http://localhost:3000)
pnpm run dev:demo         # web app with mock/demo data (no live session needed)
pnpm run dev:relay        # relay server only (port 3001)
pnpm run dev:web          # Next.js dev server only
pnpm run dev:extension    # watch-build extension (use with VS Code F5 debug)

# Production builds
pnpm run build:all        # full build: webview assets + extension bundle
pnpm run build:web        # Next.js standalone build
pnpm run build:webview    # Vite build → extension/dist/webview/
pnpm run build:extension  # esbuild → extension/dist/extension.js
pnpm run build:app        # standalone CLI app bundle

# Extension
cd extension && pnpm run lint      # TypeScript type-check (tsc --noEmit)
cd extension && pnpm run package   # produce .vsix
```

No test suite exists. Type-checking is done via `tsc --noEmit` in the extension package.

## Architecture

### Monorepo layout (pnpm workspaces)

```
extension/     # VS Code extension (Node.js, esbuild, CJS)
  src/         # Extension host: session watcher, hook server, transcript parser, protocol
web/           # Next.js 16 app (React 19, Tailwind 4, Vite for webview build)
  app/         # Next.js app router (single page)
  components/  # agent-visualizer/ — all visualization UI
    agent-visualizer/canvas/  # Canvas 2D rendering (draw-agents, draw-edges, draw-tool-calls, etc.)
  hooks/       # React hooks (simulation state machine, camera, interaction, audio)
    simulation/               # Event processing pipeline: process-event → handle-*-events → animate
  lib/         # Shared utilities (colors, canvas constants, bridge types, mock scenarios)
scripts/       # Dev relay server, setup script, build helpers
app/           # Standalone CLI entry point (npx agent-flow-app)
```

### Event pipeline

1. **Claude Code** emits hook events (SessionStart, PreToolUse, PostToolUse, SubagentStart, etc.)
2. **Hook script** (`~/.claude/agent-flow/hook.js`) — installed by `pnpm run setup` — forwards events via HTTP POST to the relay/extension
3. **Extension host** or **relay server** (`scripts/relay.ts`):
   - `HookServer` receives HTTP hook events on a dynamic port
   - `SessionWatcher` / `scanForActiveSessions` tails JSONL transcript files from `~/.claude/projects/`
   - `TranscriptParser` converts raw JSONL transcript entries into typed `AgentEvent`s
   - `SubagentWatcher` monitors `{sessionId}/subagents/*.jsonl` for subagent transcripts
   - Events are broadcast to the webview via VS Code messaging or SSE (relay)
4. **Web UI** receives events through `use-agent-simulation` hook → simulation state machine processes them into a visual graph with nodes, edges, tool call bubbles, and particles

### Key shared modules (extension/src/)

- **protocol.ts** — All TypeScript types for the event system (`AgentEvent`, `SessionInfo`, `WatchedSession`, extension↔webview message protocol)
- **constants.ts** — All magic numbers and strings (timeouts, truncation limits, token estimates, color values)
- **transcript-parser.ts** — Parses Claude Code JSONL transcripts into `AgentEvent`s
- **hook-server.ts** — Local HTTP server that receives forwarded hook events
- **discovery.ts** — Discovery file management (`~/.claude/agent-flow/{hash}-{pid}.json`) for multi-instance coordination
- **session-watcher.ts** — Watches `~/.claude/projects/` for active Claude Code session JSONL files
- **tool-summarizer.ts** — Generates human-readable summaries for tool calls

### Canvas rendering (web/components/agent-visualizer/canvas/)

The visualization uses a custom Canvas 2D renderer (no React-based graph library). The canvas draws:
- Agent nodes (`draw-agents.ts`) — positioned by d3-force simulation
- Edges between agents (`draw-edges.ts`) — parent/child relationships
- Tool call bubbles (`draw-bubbles.ts`, `draw-tool-calls.ts`) — orbiting around agent nodes
- Discovery markers (`draw-discoveries.ts`) — file/pattern access visualization
- Service nodes (`draw-service-nodes.ts`) — MCP service hexagonal nodes
- Particles (`draw-particles.ts`) and effects (`draw-effects.ts`) — visual feedback

### Web ↔ Extension bridge

In VS Code, the web UI runs inside an iframe. Communication uses:
- `web/lib/vscode-bridge.ts` + `web/hooks/use-vscode-bridge.ts` — webview side
- `extension/src/webview-provider.ts` — extension host side
- `scripts/vscode-shim.js` — shim for standalone/dev mode (replaces `acquireVsCodeApi`)

In standalone mode (`pnpm run dev` or `npx agent-flow-app`), events come via SSE from the relay server instead.

### Simulation state machine (web/hooks/simulation/)

- `process-event.ts` — Entry point; dispatches to specialized handlers
- `handle-agent-events.ts` — agent_spawn, agent_complete, agent_idle
- `handle-tool-events.ts` — tool_call_start, tool_call_end
- `handle-subagent-events.ts` — subagent_dispatch, subagent_return
- `handle-message-events.ts` — message, context_update, model_detected
- `types.ts` — Full simulation state type (`SimulationState`, `AgentNode`, `ToolCallState`, etc.)
- `animate.ts` — Animation frame loop, d3-force tick, particle updates

## Key conventions

- **pnpm only** — lockfile is `pnpm-lock.yaml`, workspace defined in `pnpm-workspace.yaml`
- **No React graph library** — canvas rendering is fully custom (Canvas 2D API + d3-force for layout)
- **Extension imports from web** — the webview build (`vite.config.webview.ts`) bundles web components into `extension/dist/webview/`
- **Relay imports from extension** — `scripts/relay.ts` imports directly from `extension/src/` (protocol, transcript-parser, constants, etc.)
- **TypeScript throughout** — extension targets ES2022/CJS (for Node.js), web targets ES6/ESNext (for browser)
- **Tailwind 4** in web package (PostCSS plugin, no `tailwind.config.js`)
- **esbuild** for extension bundling, **Vite** for webview bundling, **Next.js** for dev/standalone web
