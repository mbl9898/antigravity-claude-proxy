# Model Update Guide

> How to update Gemini (or Claude) model IDs when new versions are released.

---

## Overview

Models are defined in **two places** that must both be updated together:

| Location | Purpose |
|---|---|
| `src/constants.js` | Source-of-truth defaults — used on fresh install or when user resets presets |
| `~/.config/antigravity-proxy/claude-presets.json` | Live saved presets — what the running server actually serves |

> **Important:** The running server reads presets from `claude-presets.json`, **not** `constants.js`. Always update both files, then restart the server.

---

## Preset Structure — Gemini 1M

The **Gemini 1M** preset has two model tiers:

| Env Var | Role | Current Model |
|---|---|---|
| `ANTHROPIC_MODEL` | Primary model (Opus slot) | `gemini-3.7-flash-tiered` |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Opus-tier fallback | `gemini-3.7-flash-tiered` |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Sonnet-tier / mid tasks | `gemini-3.8-flash-tiered` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Haiku-tier / fast tasks | `gemini-3.8-flash-tiered` |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Subagent spawning | `gemini-3.8-flash-tiered` |

---

## Step-by-Step: Updating to a New Model Release

### 1. Check available models on the live server

First confirm the new model ID exists on the backend:

```bash
curl -s http://localhost:39288/v1/models | python3 -c \
  "import sys,json; [print(m['id']) for m in json.load(sys.stdin).get('data',[])]" | sort
```

> **Tip:** Only models returned here are valid. Never set a model ID that isn't in this list — the server will fail to route requests.

---

### 2. Update `src/constants.js`

Open `src/constants.js` and find the `DEFAULT_PRESETS` array (~line 297) and `TEST_MODELS` (~line 291).

**`TEST_MODELS`** — update the gemini test model:
```js
export const TEST_MODELS = {
    claude: 'claude-sonnet-4-6',
    gemini: 'gemini-X.X-flash-tiered'   // ← update to new flash model
};
```

**`DEFAULT_PRESETS` — Gemini 1M block:**
```js
{
    name: 'Gemini 1M',
    config: {
        ANTHROPIC_AUTH_TOKEN: 'test',
        ANTHROPIC_BASE_URL: 'http://localhost:8080',
        ANTHROPIC_MODEL: 'gemini-X.X-flash-tiered',                // ← primary/opus model
        ANTHROPIC_DEFAULT_OPUS_MODEL: 'gemini-X.X-flash-tiered',
        ANTHROPIC_DEFAULT_SONNET_MODEL: 'gemini-X.X-flash-tiered', // ← sub-agent model
        ANTHROPIC_DEFAULT_HAIKU_MODEL: 'gemini-X.X-flash-tiered',
        CLAUDE_CODE_SUBAGENT_MODEL: 'gemini-X.X-flash-tiered',
        ENABLE_EXPERIMENTAL_MCP_CLI: 'true'
    }
}
```

---

### 3. Update `~/.config/antigravity-proxy/claude-presets.json`

Apply the same model changes to the live saved presets file:

```bash
nano ~/.config/antigravity-proxy/claude-presets.json
```

Find the `"Gemini 1M"` block and update all model IDs:

```json
{
  "name": "Gemini 1M",
  "config": {
    "ANTHROPIC_AUTH_TOKEN": "test",
    "ANTHROPIC_BASE_URL": "http://localhost:8080",
    "ANTHROPIC_MODEL": "gemini-X.X-flash-tiered",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "gemini-X.X-flash-tiered",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "gemini-X.X-flash-tiered",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "gemini-X.X-flash-tiered",
    "CLAUDE_CODE_SUBAGENT_MODEL": "gemini-X.X-flash-tiered",
    "ENABLE_EXPERIMENTAL_MCP_CLI": "true"
  }
}
```

---

### 4. Restart the server

```bash
# Kill the running process
kill $(lsof -ti :39288 | head -1)

# Wait then restart
sleep 2
nohup /Users/Muhammad/.local/bin/antigravity-claude-proxy \
  > ~/.config/antigravity-proxy/out.log 2>&1 &

echo "Started PID: $!"
```

---

### 5. Verify everything is working

```bash
# 1. Check server is up on the correct port
curl -s http://localhost:39288/health | python3 -c \
  "import sys,json; d=json.load(sys.stdin); print('Status:', d.get('status'))"

# 2. Confirm presets have the new model IDs
curl -s http://localhost:39288/api/claude/presets | python3 -m json.tool

# 3. Confirm new model ID is in the model list
curl -s http://localhost:39288/v1/models | python3 -c \
  "import sys,json; [print(m['id']) for m in json.load(sys.stdin).get('data',[])]" \
  | grep "gemini-X.X"
```

---

## Model Version History

| Date | Preset Slot | Old Model | New Model | Reason |
|---|---|---|---|---|
| 2026-09-04 | Gemini 1M — Primary | `gemini-pro-agent` | `gemini-3.7-flash-tiered` | Gemini 3.7 released |
| 2026-09-04 | Gemini 1M — Subagent | `gemini-3.7-flash-tiered` | `gemini-3.8-flash-tiered` | Gemini 3.8 released |

---

## Quick Reference

| File | What it controls |
|---|---|
| `src/constants.js` | Default preset definitions (source of truth) |
| `~/.claude/settings.json` | **Active Claude Code env config** — drives the actual Claude Code UI |
| `~/.config/antigravity-proxy/claude-presets.json` | Live saved presets served by the running server |
| `~/.config/antigravity-proxy/server-presets.json` | Server-level presets (retry, cooldown, etc.) |
| `~/.config/antigravity-proxy/out.log` | Server runtime logs |
| Binary | `/Users/Muhammad/.local/bin/antigravity-claude-proxy` |
| Live URL | `http://localhost:39288` |

> **Note:** The server port `39288` is set externally (not in this repo). Do not change it here — update the service configuration that launches the process if a port change is needed.
