# pi-subagents

![version](https://img.shields.io/badge/pi--packages-%5E0.79.0-blue?style=flat-square)

A [pi coding agent](https://github.com/earendil-works/pi-coding-agent) extension that provides a `subagent` tool — enabling the main pi agent to spawn child pi processes as focused, disposable sub-agents.

## Overview

`pi-subagents` registers a built-in `subagent` tool so the main pi agent can delegate bounded tasks (scouting, planning, file editing, reviewing, etc.) to separate pi processes running in `--mode json`. Each subagent runs in its own process with a clean prompt, a defined tool set, and optional model/thinking overrides.

The extension handles all lifecycle management: spawning the child process, streaming progress events, rendering a live activity feed in the TUI, collecting token-usage metadata, and surfacing a final output. Subagents that time out or hit errors produce structured tool results, not crashes.

## Installation

This is a [pi package](https://github.com/earendil-works/pi-coding-agent). Add it to your project's `settings.json`:

```json
{
  "packages": ["git:github.com/juniorsundar/pi-subagents"]
}
```

**Requirements**

- [`@earendil-works/pi-coding-agent`](https://github.com/earendil-works/pi-coding-agent) `^0.79.0`
- [`@earendil-works/pi-tui`](https://github.com/earendil-works/pi-tui) `^0.79.0`

## Agent Definition Files

Subagents are defined as `.md` files in `~/.pi/agent/agents/` (or a custom `agentsDir`). Each file contains YAML frontmatter and a body that becomes the subagent's system prompt.

```markdown
---
name: scout
description: Fast codebase recon that returns compressed context for handoff
model: opencode-go/glm-5.1
tools: read,bash
timeout: 120
systemPromptMode: replace
inheritProjectContext: true
inheritSkills: true
inheritExtensions: true
defaultProgress: true
---

You are a fast, focused recon agent. Your goal is to explore an unfamiliar
codebase and return a compressed summary covering:
- Project structure and entrypoints
- Key conventions and patterns
- Dependencies and build commands
- Risks and recommended implementation strategy

Be concise. Return your findings as plain text with paths and key observations.
```

### Frontmatter Fields

| Field | Type | Description |
|---|---|---|
| `name` | `string` (required) | Agent type identifier (matches the filename) |
| `description` | `string` | Shown in the tool description when listing agents |
| `model` | `string` | Default model for this subagent |
| `tools` | `string \| string[]` | Comma-separated (or array) of allowed tool names |
| `thinking` | `string` | Thinking budget override (e.g. `"high"`) |
| `systemPromptMode` | `"replace" \| "append"` | Whether to replace or append to the default system prompt |
| `inheritProjectContext` | `boolean` (default: `true`) | Include project context files |
| `inheritSkills` | `boolean` (default: `true`) | Inherit loaded skills |
| `inheritExtensions` | `boolean` (default: `true`) | Inherit loaded extensions |
| `timeout` | `number` | Timeout in seconds (defaults vary by agent type; global default is 300s) |
| `output` | `string` | Output format specifier |
| `defaultProgress` | `boolean` | Whether to emit default progress indicators |

Unknown frontmatter keys are preserved as `extra` and passed through.

## Provided Tool

### `subagent`

| Property | Description |
|---|---|
| **Parameters** | `agent_type` (string, required), `prompt` (string, required), `model` (string, optional), `thinking` (string, optional) |
| **Returns** | Tool result containing the subagent's text output, plus a `details` object with `agentId`, `agentType`, `model`, `duration`, `usage`, and `activityFeed` |
| **TUI** | Live progress with spinner for in-progress tool calls, token count, and tool-call count. Expandable activity feed |

The tool dynamically lists available agents from the agents directory. On execution it:

1. Creates a task directory under `.pi/subagents/<agentId>/`
2. Writes the task prompt to `task.md`
3. Parses the agent definition file
4. Builds the appropriate `pi --mode json` command with flags for model, tools, system prompt, and inheritance
5. Spawns a child pi process
6. Streams typed NDJSON progress events from the child's stdout
7. Tails `progress.jsonl` for real-time activity feed updates
8. Returns the final output from `output.md` along with metadata

#### Override Precedence

The model can be set at three levels (highest to lowest priority):

1. `model` parameter in the tool call (`subagent(agent_type="scout", model="gpt-4o")`)
2. `model` field in the agent definition file
3. Falls back to pi's default model

## Architecture

```
┌─────────────────────┐
│  Main pi agent      │
│  (calls subagent)   │
└──┬──────────────────┘
   │ subagent tool
   ▼
┌─────────────────────────────┐
│  spawner.ts                 │
│  · parses agent definition  │
│  · builds pi command        │
│  · spawns child process     │
│  · streams progress events  │
└──┬──────────────────────────┘
   │ pi --mode json -p <task>
   ▼
┌─────────────────────┐
│  Child pi process   │
│  (subagent runs)    │
│  → NDJSON stdout    │
└─────────────────────┘
```

### Key Modules

| Module | Responsibility |
|---|---|
| `src/index.ts` | Extension entrypoint — registers the `subagent` tool with renderers |
| `src/spawner.ts` | Subagent lifecycle: spawn, stream, timeout, cancellation, orphan reaping |
| `src/agent-definition-parser.ts` | Parse YAML frontmatter from `.md` agent definition files |
| `src/command-builder.ts` | Build CLI args and env from an agent definition + overrides |
| `src/stream-processor.ts` | Pure async generator that types NDJSON lines into `ProgressEvent` objects |
| `src/activity-feed-formatter.ts` | Transform raw progress events into collapsed/expanded TUI views |
| `src/activity-feed-renderer.ts` | Render activity feed as TUI components |
| `src/activity-feed-tool-formatting.ts` | Tool argument summarization for compact UI display |
| `src/tail-progress.ts` | Async generator that tails `progress.jsonl` like `tail -f` |
| `src/process-registry.ts` | Register/deregister child processes for cancellation and orphan recovery |

### Lifecycle

1. **Setup** — Generate `agentId`, create task directory, write `task.md`, parse definition, build command
2. **Spawn** — Fork `pi --mode json --no-session` with appropriate flags
3. **Stream** — Pipe child stdout through `withRawLines` (persists raw lines to `events.jsonl`) into `processStream` (typed event generator)
4. **Progress** — `tailProgress` tails `progress.jsonl` and delivers formatted `ActivityFeedOutput` snapshots to the TUI via `onProgress` callback
5. **Completion** — Wait for child exit, drain stream, write final `output.md`, extract usage metadata, assemble result

### Timeout & Cancellation

- **Per-agent timeouts** are defined by the `timeout` frontmatter field (in seconds)
- **Fallback timeouts** are type-specific: `scout` (120s), `planner` (300s), `worker` (600s), default (300s)
- On timeout, `SIGTERM` is sent, a timeout error is written to `output.md`, and the tool returns gracefully
- An `AbortSignal` can be passed in to cancel execution at any point
- Orphaned subagent processes are reaped on the next `spawnSubagent` call

## TUI Rendering

The extension provides rich TUI rendering via `renderCall` and `renderResult`:

- **Pending state**: shows agent type and model with a "pending..." label
- **Running state**: live spinner for in-progress tool calls, token count, and running-tool counter
- **Expanded view**: full activity feed showing tool calls, thinking blocks, text output, and usage
- **Collapsed view**: last N events; earlier events are summarized as "… N older events hidden …"
- **Final result**: metadata block (agent, model, duration, token breakdown) followed by the subagent's output

## Error Handling

| Scenario | Behavior |
|---|---|
| Unknown agent type | `UnknownAgentError` with list of available agents |
| Timeout | Graceful tool result with error message, no crash |
| Child process crash | Error result with the child's stderr |
| Missing or invalid frontmatter | Parse error thrown at spawn time |
| Progress callback failure | Best-effort — UI rendering errors are silently ignored |

## Development

```bash
# Install dependencies
npm install

# Run tests
npm test

# Watch mode
npm run test:watch

# Type-check
npm run typecheck
```

Tests are written with [Vitest](https://vitest.dev/) and live in `test/`. The test suite covers:

- Agent definition parsing (frontmatter, YAML errors, edge cases)
- Command building (flags, model overrides, inheritance)
- Activity feed formatting (collapsed/expanded views, tool merging)
- Activity feed rendering (TUI component output)
- Stream processing (NDJSON parsing, skeleton refresh, error recovery)
- Spawner integration (success, timeout, cancellation, orphan reaping)
- E2E error paths (unknown agent, timeout signal, bad definition)
- Process registry (register/deregister, orphan reaping)

## License

[MIT](LICENSE)
