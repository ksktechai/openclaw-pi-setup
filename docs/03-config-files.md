# 3. Config Files Explained

OpenClaw spreads its configuration across a few files, and the single biggest source of confusion
in this whole setup was *which file owns what*. This page maps the layout, explains the `providers`
block, draws the line between the **allowlist** and the **primary/fallback** selection, and covers
how `.env` variables get referenced.

The canonical, sanitised config is [examples/openclaw.json.example](../examples/openclaw.json.example)
and [examples/.env.example](../examples/.env.example). Copy them to `~/.openclaw/openclaw.json` and
`~/.openclaw/.env` and fill in your own values.

> Placeholders in the examples: `OLLAMA_HOST_IP` is your Ollama machine's LAN IP, `USER` is your Pi
> login. Replace them before use.

---

## 3.1 Two files, two jobs

```
~/.openclaw/
├── openclaw.json                       # GLOBAL config — providers, gateway, channels, plugins, skills
├── .env                                # secrets, referenced by name from openclaw.json (never committed)
├── agents/
│   └── main/
│       └── agent/
│           ├── agent.json              # the "main" agent's own settings
│           └── models.json             # per-agent model state/overrides
└── workspace/                          # the agent's working files, memory, skills
```

| File | Scope | What belongs here |
|------|-------|-------------------|
| `~/.openclaw/openclaw.json` | **Global** | `models.providers`, the model allowlist + primary/fallbacks, `gateway`, `channels`, `plugins`, `skills`, `commands` |
| `~/.openclaw/agents/main/agent/models.json` | **Per-agent** | Machine-managed model state for the `main` agent — generally *not* where you hand-define providers |
| `~/.openclaw/.env` | **Secrets** | API keys/tokens, referenced by name from `openclaw.json` |

> **The trap I fell into:** I hand-edited `agents/main/agent/models.json` expecting it to define my
> providers, restarted repeatedly, and nothing changed — because **providers live in the global
> `openclaw.json` under `models.providers`**. `models.json` is per-agent state, not the place you
> declare DeepSeek/Ollama/HuggingFace. If an edit "isn't taking," check you're in the right file
> first. (See [Troubleshooting §4](06-troubleshooting.md#4-modelsjson-vs-openclawjson-provider-confusion).)
>
> Edit `openclaw.json`, then always `npx openclaw gateway restart` and `npx openclaw doctor --fix`.
> Validate JSON with `jq . ~/.openclaw/openclaw.json` before restarting — a stray comma is the most
> common "gateway won't start" cause.

---

## 3.2 The `providers` block

Under `models.providers`, each provider is an OpenAI-compatible endpoint plus the models it serves:

```json
"models": {
  "providers": {
    "deepseek": {
      "baseUrl": "https://api.deepseek.com/v1",
      "apiKey": "${DEEPSEEK_API_KEY:-}",
      "api": "openai-completions",
      "models": [
        { "id": "deepseek-v4-pro", "name": "DeepSeek V4 Pro",
          "reasoning": true, "input": ["text"], "contextWindow": 160000, "maxTokens": 8192 }
      ]
    },
    "ollama": {
      "apiKey": "ollama-local",
      "baseUrl": "http://OLLAMA_HOST_IP:11434/v1",
      "models": [
        { "id": "qwen3.6:35b-a3b-mxfp8", "name": "qwen3.6:35b-a3b-mxfp8", "contextWindow": 131072 }
      ]
    },
    "huggingface": {
      "apiKey": "$HF_TOKEN",
      "baseUrl": "https://router.huggingface.co/v1",
      "models": [
        { "id": "zai-org/GLM-5.2:novita", "name": "zai-org/GLM-5.2:novita", "contextWindow": 131072 }
      ]
    }
  }
}
```

Field by field:

| Field | Meaning |
|-------|---------|
| `baseUrl` | The provider's OpenAI-compatible endpoint. For Ollama this is `http://<host>:11434/v1` — the **`/v1`** suffix is what selects the OpenAI-compatible API. |
| `apiKey` | A reference to an env var (`$HF_TOKEN`, `${DEEPSEEK_API_KEY:-}`), or a literal for keyless local endpoints (Ollama accepts any non-empty string — `"ollama-local"` here). |
| `api` | The protocol, e.g. `"openai-completions"`. |
| `models[].id` | The model ID sent to the provider. |
| `models[].contextWindow` | Max tokens the model accepts. **Set this honestly** — too high and you'll trigger context-overflow / compaction failures ([Troubleshooting §3](06-troubleshooting.md#3-context-overflow--compaction-failure)). |
| `models[].maxTokens` | Max tokens to generate per response. |
| `models[].reasoning` | Whether this is a reasoning model. |

Model references elsewhere use `provider/id`, e.g. `ollama/qwen3.6:35b-a3b-mxfp8`,
`deepseek/deepseek-v4-pro`, `huggingface/zai-org/GLM-5.2:novita`.

---

## 3.3 Allowlist vs. primary/fallbacks

These two live next to each other under `agents.defaults` and are easy to conflate — but they answer
different questions:

```json
"agents": {
  "defaults": {
    "models": {
      "deepseek/deepseek-v4-pro": {},
      "google/gemini-2.5-flash-lite": {},
      "ollama/qwen3.6:35b-a3b-mxfp8": {},
      "huggingface/zai-org/GLM-5.2:novita": {}
    },
    "model": {
      "primary": "deepseek/deepseek-v4-pro",
      "fallbacks": [
        "ollama/qwen3.6:35b-a3b-mxfp8",
        "google/gemini-2.5-flash-lite",
        "huggingface/zai-org/GLM-5.2:novita"
      ]
    }
  }
}
```

- **`agents.defaults.models`** is the **allowlist** — the full set of models this agent is *permitted*
  to use. A model has to appear here to be selectable at all (including by cron jobs that pin their
  own model — see [§3.5](#35-where-cron-fits)).
- **`agents.defaults.model.primary` / `.fallbacks`** is the **selection policy** — which allowed
  model the agent reaches for *first*, and the ordered chain it falls through if that one is down,
  rate-limited, or erroring.

So: the allowlist says *what's on the menu*; primary/fallbacks says *what you order, and your backup
order*. A model in `fallbacks` must also be in the allowlist.

The chain here means the agent stays available even if DeepSeek is down: it drops to the free local
Ollama model, then to cheap Gemini, then to the HuggingFace free tier.

---

## 3.4 `.env` references and interpolation

Secrets never sit inside `openclaw.json`. They live in `~/.openclaw/.env` and are referenced by name;
the gateway interpolates them at load time.

```bash
# ~/.openclaw/.env
DEEPSEEK_API_KEY=sk-...
HF_TOKEN=hf_...                 # raw token only — the client adds "Bearer"
TELEGRAM_BOT_TOKEN=123456789:...
OPENCLAW_GATEWAY_TOKEN=...
BRAVE_API_KEY=...
TRANSCRIPT_API_KEY=...
```

Two reference forms appear in `openclaw.json`:

| Form | Meaning |
|------|---------|
| `"$HF_TOKEN"` | Substitute the value of `HF_TOKEN`. |
| `"${DEEPSEEK_API_KEY:-}"` | Substitute `DEEPSEEK_API_KEY`, or an empty string if it's unset (a safe default that won't crash config parsing). |

After editing `.env`, restart the gateway so it re-reads the file:
`npx openclaw gateway restart`.

> Both `~/.openclaw/.env` and `~/.openclaw/openclaw.json` are gitignored in this repo — only the
> `*.example` versions are committed. See the [security note in the README](../README.md#security--secrets).

---

## 3.5 Where cron fits

Cron jobs can **override the model per job**, independent of `model.primary`. The job's pinned model
must still be in the allowlist (§3.3). This is how the scheduled digests run on the *free local Ollama*
model while interactive chat stays on DeepSeek — covered in **[4. Cron jobs](04-cron-jobs.md)**.

---

## 3.6 Other notable blocks

| Block | Purpose |
|-------|---------|
| `gateway` | `bind: "loopback"` (not exposed to the LAN), `auth.mode: "token"` with `$OPENCLAW_GATEWAY_TOKEN`, `port: 18789`. Tailscale off by default. |
| `gateway.nodes.denyCommands` | Hard deny-list for sensitive device actions (camera, SMS, calendar writes, …). |
| `channels.telegram` | The Telegram front-end (token, group mention rules, streaming). |
| `plugins.entries` | Enable/configure plugins (Brave, Google/Gemini search, Telegram, provider plugins). |
| `commands.ownerAllowFrom` | Restricts who can issue owner commands, e.g. `["telegram:123456789"]`. |
| `skills.entries` | Per-skill enable/disable flags. |
| `tools.web` | Toggles `web.search` (provider `brave`) and `web.fetch`. |
