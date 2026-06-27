# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a **documentation-only** repository: a polished, publishable guide to running **OpenClaw**
(a personal AI-agent runtime) on a Raspberry Pi 5, with inference offloaded to a local Ollama model
on a separate machine. The deliverable *is* the prose, config examples, and command transcripts —
there is no application source, build system, or test suite. "Working here" means editing Markdown
and the sanitised example configs.

The repo is the **canonical reference**; a narrative Substack post is derived from it later, so keep
the repo authoritative and self-consistent.

## Structure

```
README.md            # overview, architecture, quickstart, proof-it-works, security, doc index, links
docs/
  01-pi-setup.md         # headless Pi: SSH, hostname/mDNS, Bluetooth, connectivity
  02-openclaw-setup.md   # Node → OpenClaw → Telegram → Ollama → models → plugins → skills (the journey)
  03-config-files.md     # openclaw.json vs agents/main/agent/models.json; providers; allowlist vs primary/fallbacks; .env
  04-cron-jobs.md        # scheduled digests; per-job model pinning; timezone gotcha; OS→OpenClaw migration
  05-skills.md           # youtube-full vs youtube-watcher, weather, stock-analysis; uv/yt-dlp deps
  06-troubleshooting.md  # every real error from the build, with cause + fix
examples/
  openclaw.json.example  # canonical sanitised config — the docs point here
  .env.example           # canonical sanitised env template
assets/hermestechbot.png # architecture diagram
.gitignore               # ignores real .env / openclaw.json / *.bak / *.log / SSH keys; allows *.example
```

## Hard rules

- **Never commit secrets or real identifying data.** Every example and command MUST use placeholders.
  The established placeholder vocabulary (use these consistently):
  `<user>` (Pi login), `<pi-host>` (hostname), `<pi-ip>` / `<ollama-ip>` (LAN IPs), `<bt-mac>`
  (Bluetooth MAC), `<PAIRING-CODE>` (Telegram pairing code), `123456789` (Telegram user/group ID).
  In JSON, `OLLAMA_HOST_IP` and `USER` are the in-string placeholders.
  API keys/tokens are always env-var references (`$TELEGRAM_BOT_TOKEN`, `${DEEPSEEK_API_KEY:-}`), never
  literals. If you ever paste from the author's real machine (`~/Downloads/openclaw.json`, history,
  `pi-setup.md`), scrub `taihoro`, `192.168.1.4/.12/.14/.20`, the MX-Keys MAC, and any pairing code first.
- **`examples/` is the single source of truth for config.** When config appears inline in a doc, it
  must agree with `examples/openclaw.json.example`. Don't let them drift.
- **Cross-page links use GitHub heading anchors** (lowercase, punctuation stripped, spaces→hyphens;
  a `/` between words yields a double hyphen). The pages link to each other heavily — if you rename a
  heading, update every inbound link. Validate with the checks below.

## Validate before considering work done

```bash
# No real secrets/identity leaked into the repo:
grep -rinE 'taihoro|D3:CF:DB:59:2F:64|LRGXZZMD|192\.168\.1\.(4|12|14|20)' . --exclude-dir=.git
# Example config is valid JSON:
python3 -m json.tool examples/openclaw.json.example >/dev/null
# .gitignore protects real config but not the templates:
git check-ignore .env openclaw.json                       # should print (ignored)
git check-ignore examples/.env.example examples/openclaw.json.example  # should print nothing
```

Also eyeball that every relative link resolves and that cross-page anchors point at real headings.

## Domain context (so edits stay accurate)

- **Topology:** Telegram → OpenClaw on the Pi (agent/orchestrator — makes ALL tool/API/internet calls)
  → OpenAI-compatible HTTP → Ollama on a MacBook (inference only, never touches the internet).
- **Model policy:** primary `deepseek/deepseek-v4-pro`; fallbacks → local Ollama qwen → Gemini
  Flash-Lite → HuggingFace GLM-5.2. The **allowlist** (`agents.defaults.models`) is which models are
  *permitted*; **`model.primary`/`fallbacks`** is which is *chosen* and in what order. Cron jobs pin
  their own `payload.model` independent of the primary (so background work runs free on local Ollama).
- **Config lives in `~/.openclaw/openclaw.json`** (global: providers, gateway, channels, plugins,
  skills), NOT `~/.openclaw/agents/main/agent/models.json` (per-agent state) — this confusion is a
  documented pitfall; keep it straight.
- The agent persona is "Hermes"/"HermesTechBot"; the human is "Senthil".
