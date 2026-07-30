# Kiro Agent Tracing with Phoenix OSS

Traces every Kiro IDE session locally using [Phoenix](https://phoenix.arize.com) (self-hosted).
All data stays on your machine — no cloud account needed.

---

## How it works

```
Kiro IDE event (preToolUse / postToolUse / agentStop / promptSubmit)
     │
     ▼
.kiro/hooks/phoenix-tracing.json   ← fires the hook script on each event
     │
     ▼
~/.phoenix/hooks/phoenix-hook-kiro ← Python script using OTel SDK
     │  reads event JSON from STDIN, accumulates state, builds OTLP span
     ▼
http://localhost:6006/v1/traces    ← Phoenix OTLP/protobuf endpoint
     │
     ▼
Phoenix UI  http://localhost:6006
```

Each Kiro turn produces two kinds of spans visible in Phoenix:

| Span kind | Hook event    | Contains |
|-----------|--------------|----------|
| `TOOL`    | `postToolUse` | Tool name, input, output — child of the LLM span |
| `LLM`     | `agentStop`   | User prompt, assistant response, session ID |

---

## Prerequisites

| Requirement | Status |
|-------------|--------|
| Python 3.x | ✅ `/c/dev/software/Python312/python.exe` |
| Phoenix running at `http://localhost:6006` | ✅ |
| `opentelemetry-sdk` | ✅ installed |
| `opentelemetry-exporter-otlp-proto-http` | ✅ installed |

---

## File layout

```
~/.phoenix/
├── hooks/
│   └── phoenix-hook-kiro       ← hook script (executable)
├── logs/
│   └── kiro.log                ← written on every hook invocation
└── state/
    └── <session-id>.json       ← per-session sidecar (auto-managed)

<workspace>/
├── bin/
│   └── phoenix-hook-kiro       ← workspace copy (source of truth)
├── docs/
│   └── kiro-tracing-setup.md   ← this document
└── .kiro/
    └── hooks/
        └── phoenix-tracing.json ← Kiro IDE hook wiring
```

---

## Installation

### 1. Start Phoenix

```bash
# Option A — pip
pip install arize-phoenix
phoenix serve

# Option B — Docker
docker run -p 6006:6006 arizephoenix/phoenix:latest
```

Verify it's up:

```bash
curl -sf http://localhost:6006/v1/traces -X POST \
  -H "Content-Type: application/x-protobuf" --data-binary '' \
  && echo "Phoenix accepts OTLP"
```

### 2. Install OTel SDK (one-time)

```bash
python -m pip install opentelemetry-sdk opentelemetry-exporter-otlp-proto-http
```

### 3. Deploy the hook script

The script lives in `bin/phoenix-hook-kiro` in this workspace. Copy it to `~/.phoenix/hooks/`:

```bash
mkdir -p ~/.phoenix/hooks ~/.phoenix/logs ~/.phoenix/state
cp bin/phoenix-hook-kiro ~/.phoenix/hooks/phoenix-hook-kiro
chmod +x ~/.phoenix/hooks/phoenix-hook-kiro
```

### 4. Hook wiring

`.kiro/hooks/phoenix-tracing.json` is already present in this workspace. Kiro picks it up automatically — no restart needed.

---

## Configuration

Set these environment variables in your shell profile before starting Kiro:

| Variable | Default | Purpose |
|----------|---------|---------|
| `PHOENIX_ENDPOINT` | `http://localhost:6006` | Phoenix base URL |
| `PHOENIX_PROJECT`  | `kiro` | Project name shown in Phoenix UI |
| `PHOENIX_USER_ID`  | _(empty)_ | Your name — added to every span |
| `PHOENIX_DRY_RUN`  | `false` | Run hooks without sending spans |
| `PHOENIX_VERBOSE`  | `false` | Write debug output to stderr |

Example:

```bash
export PHOENIX_PROJECT="my-project"
export PHOENIX_USER_ID="ram"
```

---

## Verify the setup

### Smoke-test the hook script

Run a full simulated turn through the script:

```bash
PYTHON=/c/dev/software/Python312/python.exe
HOOK=~/.phoenix/hooks/phoenix-hook-kiro
SID="test-$(date +%s)"

echo '{"hook_event_name":"agentSpawn","session_id":"'"$SID"'"}' | $PYTHON $HOOK ide
echo '{"hook_event_name":"userPromptSubmit","session_id":"'"$SID"'","prompt":"hello"}' | $PYTHON $HOOK ide
echo '{"hook_event_name":"preToolUse","session_id":"'"$SID"'","tool_name":"read_file","tool_input":{"path":"pom.xml"}}' | $PYTHON $HOOK ide
echo '{"hook_event_name":"postToolUse","session_id":"'"$SID"'","tool_name":"read_file","tool_response":{"content":"..."}}' | $PYTHON $HOOK ide
echo '{"hook_event_name":"stop","session_id":"'"$SID"'","assistant_response":"Done."}' | $PYTHON $HOOK ide
```

Expected output in `~/.phoenix/logs/kiro.log`:

```
INFO agentSpawn session=test-...
INFO userPromptSubmit session=test-...
INFO preToolUse session=test-... tool=read_file
INFO emitted TOOL span tool=read_file session=test-...
INFO emitted LLM span session=test-...
```

No `ERROR` lines = spans successfully sent.

### Check Phoenix UI

1. Open **http://localhost:6006**
2. Select project **kiro** (or whichever `PHOENIX_PROJECT` is set to)
3. After a real Kiro session you'll see LLM spans with nested TOOL children

### Watch the log live

```bash
tail -f ~/.phoenix/logs/kiro.log
```

---

## IDE hook events wired

| Hook name | Kiro event | Span emitted |
|-----------|-----------|--------------|
| `phoenix-trace-session-start` | `promptSubmit` | None — captures prompt text |
| `phoenix-trace-pre-tool` | `preToolUse` | None — records tool input + start time |
| `phoenix-trace-post-tool` | `postToolUse` | **TOOL** span |
| `phoenix-trace-turn` | `agentStop` | **LLM** span |

---

## Span attributes

### LLM span (`agentStop`)

| Attribute | Value |
|-----------|-------|
| `openinference.span.kind` | `LLM` |
| `input.value` | User prompt |
| `output.value` | Assistant response |
| `session.id` | Kiro session UUID |
| `kiro.agent_name` | Agent name |
| `llm.model_name` | Model ID (when provided by event) |
| `user.id` | `PHOENIX_USER_ID` (when set) |

### TOOL span (`postToolUse`)

| Attribute | Value |
|-----------|-------|
| `openinference.span.kind` | `TOOL` |
| `tool.name` | Name of the tool called |
| `input.value` | Serialised tool input JSON (capped 4 KB) |
| `output.value` | Serialised tool response JSON (capped 4 KB) |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| No traces in Phoenix UI | Check log: `tail -20 ~/.phoenix/logs/kiro.log`. Confirm Phoenix: `curl -sf http://localhost:6006/v1/traces -X POST -H "Content-Type: application/x-protobuf" --data-binary ''` |
| `ERROR` in log with HTTP code | Run smoke-test above manually to see full error |
| Hook not firing | Confirm `.kiro/hooks/phoenix-tracing.json` exists. Check Kiro "Agent Hooks" panel in the Explorer view |
| `opentelemetry packages not installed` | Run: `python -m pip install opentelemetry-sdk opentelemetry-exporter-otlp-proto-http` |
| `no pending tool slot` | Concurrent tool calls broke FIFO matching — orphan TOOL span still emitted |
| Test without sending data | `export PHOENIX_DRY_RUN=true` |

---

## Known limitations

- **Token counts.** Kiro meters in credits, not tokens. Token attributes are omitted.
- **FIFO tool matching.** Pre/post tool events are matched via a FIFO stack. Concurrent tool calls within one turn can mismatch.
- **Turn duration.** IDE hooks don't expose turn start time, so LLM span duration is approximate.
