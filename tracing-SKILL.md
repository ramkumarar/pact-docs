---
name: manage-kiro-tracing
description: Set up and configure Phoenix (self-hosted) tracing for Kiro sessions — both the IDE and the CLI. Use when users want to set up Kiro tracing, configure Phoenix for Kiro, enable/disable tracing, evaluate an agentic session, or troubleshoot Kiro tracing issues. Triggers on "set up kiro tracing", "configure Phoenix for Kiro", "enable kiro tracing", "kiro agent tracing", "evaluate kiro session", or any request about connecting Kiro to Phoenix for observability.
---

# Setup Kiro Tracing with Phoenix

Configure OpenInference tracing for **Kiro** sessions (IDE and CLI) using **Phoenix OSS** (self-hosted). Spans are sent directly from hooks to Phoenix over OTLP/HTTP — no cloud credentials, no background process, and no backend-specific Python dependencies needed. Each traced session emits LLM turns, tool calls, model information, and turn duration.

Phoenix runs entirely on your machine and does not send any data over the internet.

## How to Use This Skill

**This skill follows a decision tree workflow.** Start by asking the user where they are in the setup process:

1. **Is Phoenix running?**
   - Check: `curl -sf http://localhost:6006/v1/traces >/dev/null && echo "Phoenix is running" || echo "Phoenix not reachable"`
   - If not running → Go to [Install Phoenix](#install-phoenix)

2. **Which Kiro surface do they want to trace?**
   - Kiro IDE → Go to [IDE Hooks](#ide-hooks)
   - Kiro CLI → Go to [CLI Hooks](#cli-hooks)
   - Both → Do both sections

3. **Are they troubleshooting?**
   - Yes → Jump to [Troubleshoot](#troubleshoot)

**Important:** Only follow the relevant path. Don't go through all sections.

---

## Install Phoenix

Ask if Phoenix is already running. If not:

```bash
# Option A: pip
pip install arize-phoenix
phoenix serve

# Option B: Docker
docker run -p 6006:6006 arizephoenix/phoenix:latest
```

Phoenix UI is at `http://localhost:6006`. Verify it's up:

```bash
curl -sf http://localhost:6006/v1/traces >/dev/null && echo "Phoenix is running" || echo "Phoenix not reachable"
```

No API key or account is required. Phoenix is fully local.

---

## How Spans Reach Phoenix

Spans are sent via HTTP POST to `http://localhost:6006/v1/traces` using the OTLP/HTTP+Protobuf format. The hook script reads the Kiro event from STDIN, builds an OpenInference-compliant span, and posts it directly. No OpenTelemetry SDK is needed — only Python's stdlib `urllib` and `struct` are used.

The Phoenix OTLP endpoint accepts:
- **URL**: `http://localhost:6006/v1/traces`
- **Method**: POST
- **Content-Type**: `application/x-protobuf`
- **Body**: OTLP ExportTraceServiceRequest (protobuf-encoded)

Optionally, if you have `opentelemetry-exporter-otlp-proto-http` installed, you can use it instead of raw urllib. Either approach works.

---

## IDE Hooks

The Kiro IDE hook system uses JSON files stored in `.kiro/hooks/` inside the workspace. Hooks fire on IDE events and can run shell commands or invoke the agent.

### IDE Hook Events

| Event | Description | Can block? |
|-------|-------------|------------|
| `SessionStart` | Session begins | No |
| `UserPromptSubmit` | User submits a prompt | Yes |
| `PreToolUse` | Before a tool executes | Yes |
| `PostToolUse` | After a tool executes | No |
| `Stop` | Agent completes its turn | No |
| `PostFileSave` | After agent saves a file | No |
| `PostFileCreate` | After agent creates a file | No |
| `PostFileDelete` | After agent deletes a file | No |
| `PreTaskExec` | Before a spec task starts | Yes |
| `PostTaskExec` | After a spec task finishes | No |

### IDE Hook File Format

Hooks are JSON files in `.kiro/hooks/` with this schema:

```json
{
  "version": "v1",
  "hooks": [
    {
      "name": "phoenix-trace-tool",
      "trigger": "PostToolUse",
      "action": { "type": "command", "command": "~/.phoenix/hooks/phoenix-hook-kiro ide" },
      "timeout": 10,
      "enabled": true
    },
    {
      "name": "phoenix-trace-turn",
      "trigger": "Stop",
      "action": { "type": "command", "command": "~/.phoenix/hooks/phoenix-hook-kiro ide" },
      "timeout": 10,
      "enabled": true
    }
  ]
}
```

Save this as `.kiro/hooks/phoenix-tracing.json` in your workspace.

**Key points:**
- The hook script receives the event payload as JSON on STDIN (Kiro IDE passes `{ "hook_event_name": "...", ... }`)
- `PostToolUse` emits a TOOL span for each tool call
- `Stop` emits the parent LLM turn span

For complete tracing (including session start and prompt context), you can also add `SessionStart` and `UserPromptSubmit` hooks that accumulate state without emitting spans.

### IDE Hook — Validate

1. Confirm the file exists: `cat .kiro/hooks/phoenix-tracing.json`
2. Confirm Phoenix is reachable: `curl -sf http://localhost:6006/v1/traces >/dev/null`
3. Run a Kiro session in the IDE and check the Phoenix UI at `http://localhost:6006`

---

## CLI Hooks

The Kiro CLI hook system is defined in agent JSON files under `~/.kiro/agents/<name>.json`. The five lifecycle events are:

| Event | Emits span? | Description |
|-------|-------------|-------------|
| `agentSpawn` | No | Agent activated — initialize per-session state |
| `userPromptSubmit` | No | User submits a prompt — capture prompt and generate trace/span IDs |
| `preToolUse` | No | Before a tool runs — push tool input + start time to a FIFO stack |
| `postToolUse` | Yes | After a tool runs — emit a TOOL span |
| `stop` | Yes | Turn finished — emit the parent LLM span |

Hook events arrive via STDIN as JSON:

```json
{
  "hook_event_name": "postToolUse",
  "cwd": "/current/working/directory",
  "session_id": "abc123-def456-789",
  "tool_name": "read",
  "tool_input": { ... },
  "tool_response": { ... }
}
```

The `stop` event includes `assistant_response` and `session_id`.

### Create the CLI Agent File

Ask the user for:
1. **Agent name** (default: `phoenix-traced`) — written to `~/.kiro/agents/<name>.json`
2. **Project name** (default: `kiro`) — groups traces in Phoenix UI
3. **Set as default?** — if yes, run `kiro-cli agent set-default <name>` so `kiro-cli chat` uses it automatically

Create `~/.kiro/agents/phoenix-traced.json`:

```json
{
  "name": "phoenix-traced",
  "description": "Kiro CLI agent with Phoenix OSS tracing hooks.",
  "prompt": null,
  "mcpServers": {},
  "tools": ["*"],
  "toolAliases": {},
  "allowedTools": [],
  "resources": [],
  "hooks": {
    "agentSpawn":       [{ "command": "~/.phoenix/hooks/phoenix-hook-kiro cli" }],
    "userPromptSubmit": [{ "command": "~/.phoenix/hooks/phoenix-hook-kiro cli" }],
    "preToolUse":       [{ "command": "~/.phoenix/hooks/phoenix-hook-kiro cli" }],
    "postToolUse":      [{ "command": "~/.phoenix/hooks/phoenix-hook-kiro cli" }],
    "stop":             [{ "command": "~/.phoenix/hooks/phoenix-hook-kiro cli" }]
  },
  "toolsSettings": {},
  "includeMcpJson": true,
  "model": null
}
```

All five events route to a single `phoenix-hook-kiro` script that dispatches based on `hook_event_name` in the JSON payload. The `cli` argument tells the script it is running from the CLI context.

If the user already has an agent they want to trace, merge the five `hooks` entries above into their existing agent JSON — do not overwrite the rest of the agent definition.

### Set as Default (optional)

```bash
kiro-cli agent set-default phoenix-traced
```

Then just use:

```bash
kiro-cli chat
```

Or pass explicitly:

```bash
kiro-cli chat --agent phoenix-traced
```

### CLI Hooks — Validate

1. Confirm agent file: `cat ~/.kiro/agents/phoenix-traced.json`
2. Confirm Phoenix reachable: `curl -sf http://localhost:6006/v1/traces >/dev/null`
3. Run a session and check Phoenix UI at `http://localhost:6006`

---

## The Hook Script

Both IDE and CLI hooks call the same script (`~/.phoenix/hooks/phoenix-hook-kiro`) with a surface argument (`ide` or `cli`). Install it:

```bash
mkdir -p ~/.phoenix/hooks
```

The script should:
1. Read JSON from STDIN
2. Dispatch on `hook_event_name`
3. For state-accumulating events (`agentSpawn`, `userPromptSubmit`, `preToolUse`): write sidecar state to `~/.phoenix/state/<session_id>.json`
4. For emitting events (`postToolUse`, `stop`): read state, build an OpenInference span, POST to `http://localhost:6006/v1/traces`

The Phoenix OTLP endpoint (`/v1/traces`) accepts standard OTLP/HTTP protobuf. You can use:
- Raw `urllib` + manual protobuf encoding (zero dependencies)
- `opentelemetry-exporter-otlp-proto-http` if already installed

### Configure the endpoint

The script reads the Phoenix endpoint from an environment variable or a config file:

```bash
export PHOENIX_ENDPOINT="http://localhost:6006"     # default
export PHOENIX_PROJECT="kiro"                        # default project name
export PHOENIX_USER_ID="your-name"                   # optional, for team attribution
```

Or create `~/.phoenix/hooks/config.yaml`:

```yaml
endpoint: http://localhost:6006
project_name: kiro
```

### Dry run and debug

```bash
PHOENIX_DRY_RUN=true kiro-cli chat --agent phoenix-traced   # no spans sent
PHOENIX_VERBOSE=true kiro-cli chat --agent phoenix-traced   # verbose hook logging
```

Logs are written to `~/.phoenix/logs/kiro.log`.

---

## Span Attributes

### LLM turn span (emitted on `stop`)

| Attribute | Description |
|-----------|-------------|
| `session.id` | Kiro session UUID |
| `openinference.span.kind` | `LLM` |
| `input.value` | User prompt |
| `output.value` | Assistant response |
| `llm.model_name` | Model ID (from session sidecar when available) |
| `llm.token_count.prompt` / `.completion` / `.total` | Token counts (omitted when 0) |
| `kiro.turn_duration_ms` | Turn duration in milliseconds |
| `kiro.agent_name` | Agent name (e.g. `phoenix-traced`) |
| `kiro.context_usage_percentage` | Context window usage percentage |

### TOOL span (emitted on `postToolUse`)

| Attribute | Description |
|-----------|-------------|
| `tool.name` | Tool name |
| `tool.description` | Purpose of the call (from `__tool_use_purpose` in input) |
| `input.value` | Serialized tool input JSON |
| `output.value` | Serialized tool response JSON |

TOOL spans are parented to the LLM turn span via the FIFO tool-state stack. This assumes serial tool execution — concurrent tool calls within a single turn would mismatch.

---

## Hook Events Summary

### CLI hooks (agent JSON)

| Event | Emits span? | Span kind | Purpose |
|-------|-------------|-----------|---------|
| `agentSpawn` | No | — | Initialize session state |
| `userPromptSubmit` | No | — | Capture prompt, generate trace/span IDs |
| `preToolUse` | No | — | Push tool input + start time to FIFO stack |
| `postToolUse` | Yes | `TOOL` | Pop tool slot, emit TOOL span |
| `stop` | Yes | `LLM` | Emit parent LLM turn span |

### IDE hooks (.kiro/hooks/)

| Trigger | Emits span? | Span kind | Purpose |
|---------|-------------|-----------|---------|
| `SessionStart` | No | — | Initialize session state |
| `UserPromptSubmit` | No | — | Capture prompt, generate trace/span IDs |
| `PreToolUse` | No | — | Push tool input + start time |
| `PostToolUse` | Yes | `TOOL` | Emit TOOL span |
| `Stop` | Yes | `LLM` | Emit LLM turn span |

---

## Known Limitations

- **Token counts are typically 0.** Kiro meters in credits, not tokens. Token count attributes are omitted when the value is 0.
- **FIFO tool matching.** Kiro does not expose a tool-call ID, so pre/post tool events are matched using a FIFO stack. Concurrent tool calls within a single turn would mismatch. An orphan TOOL span is emitted when the stack is empty — check `~/.phoenix/logs/kiro.log` for `no pending tool slot`.
- **Sidecar read is fail-soft (CLI).** The session sidecar at `~/.kiro/sessions/cli/<session_id>.json` may not exist or may lag hook events due to a flush race. When this happens, the LLM span is emitted with basic attributes only (no model name or duration).
- **IDE context.** The IDE does not provide a session sidecar file — span enrichment relies entirely on what the hook payload includes.

---

## Troubleshoot

| Problem | Fix |
|---------|-----|
| Traces not appearing | Check Phoenix is running: `curl -sf http://localhost:6006/v1/traces`. Check hook log: `tail -20 ~/.phoenix/logs/kiro.log` |
| Phoenix unreachable | Start Phoenix: `phoenix serve` or `docker run -p 6006:6006 arizephoenix/phoenix:latest` |
| CLI hooks not firing | Verify the agent JSON has all five hooks under `hooks` and that each `command` resolves to the hook script. Run `kiro-cli agent validate --path ~/.kiro/agents/<agent>.json` |
| IDE hooks not firing | Verify `.kiro/hooks/phoenix-tracing.json` exists and has `"enabled": true` |
| Wrong agent in use (CLI) | Pass `--agent <name>` to `kiro-cli chat`, or set default: `kiro-cli agent set-default <name>` |
| LLM spans missing model name / duration (CLI) | Session sidecar at `~/.kiro/sessions/cli/<session_id>.json` was unavailable when `stop` fired. Enrichment is fail-soft — span is emitted without those attributes |
| Tool spans mismatched or orphaned | Concurrent tool execution broke the FIFO match. Search hook log for `no pending tool slot` |
| Test without sending data | Set `PHOENIX_DRY_RUN=true` before launching Kiro |
| Verbose logging | Set `PHOENIX_VERBOSE=true` before launching Kiro |
| Wrong project name | Set `PHOENIX_PROJECT=<name>` or update `project_name` in `~/.phoenix/hooks/config.yaml` |
| Spans missing user attribution | Set `PHOENIX_USER_ID=<name>` before launching Kiro |
