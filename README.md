# OpenClaw on Raspberry Pi 5 — Complete Setup Guide

> **Author:** Senthil (via Hermes)  
> **Date:** June 2026  
> **Target audience:** Developers who want to run a personal AI agent on a Pi with local + cloud LLMs, Telegram integration, and automated cron jobs.

---

## Table of Contents

1. [Hardware & OS](#1-hardware--os)
2. [Installing OpenClaw](#2-installing-openclaw)
3. [Configuring LLM Models](#3-configuring-llm-models)
4. [Setting Up Telegram](#4-setting-up-telegram)
5. [Agent Identity & Personality](#5-agent-identity--personality)
6. [Installing Skills](#6-installing-skills)
7. [Creating Cron Jobs](#7-creating-cron-jobs)
8. [Migrating OS Cron to OpenClaw Cron](#8-migrating-os-cron-to-openclaw-cron)
9. [Full Configuration Reference](#9-full-configuration-reference)
10. [Tips & Lessons Learned](#10-tips--lessons-learned)

---

## 1. Hardware & OS

| Component | Spec |
|-----------|------|
| Board | Raspberry Pi 5 Model B Rev 1.1 |
| Architecture | aarch64 (ARM 64-bit) |
| RAM | 16 GB |
| Storage | 256 GB microSD (235 GB usable, ~11 GB used) |
| OS | Debian GNU/Linux 13 (Trixie) |
| Node.js | v24.18.0 |
| OpenClaw | 2026.6.10 |

### Why a Pi 5?

- **Low power** — runs 24/7 for pennies a month.
- **Silent** — sits in a corner, zero noise.
- **Enough horsepower** — 16 GB RAM comfortably runs OpenClaw, Node.js, and background agents.
- **Local LLMs offloaded** — Ollama runs on a separate machine (`192.168.1.4`) with a GPU, so the Pi only handles orchestration.

### Initial Pi Setup (checklist)

```bash
# Flash Debian Trixie to SD card using Raspberry Pi Imager
# Enable SSH, set hostname, configure Wi-Fi in imager

# Post-boot:
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs git curl jq python3
```

---

## 2. Installing OpenClaw

```bash
# Install globally via npm
npm install -g openclaw

# Verify
openclaw --version
# → OpenClaw 2026.6.10
```

OpenClaw installs to `~/.npm-global/lib/node_modules/openclaw/`.  
Your workspace lives at `~/.openclaw/workspace/`.

### First Run

On first start, OpenClaw creates `BOOTSTRAP.md` in the workspace — follow it, configure your identity, then delete it.

```bash
openclaw gateway start
```

The gateway runs on `localhost:18789` by default (loopback-only for security).

---

## 3. Configuring LLM Models

All model configuration lives in `~/.openclaw/openclaw.json`.

### Our Setup: Multi-Provider with Fallback Chain

```json
{
  "models": {
    "providers": {
      "deepseek": {
        "baseUrl": "https://api.deepseek.com/v1",
        "apiKey": "${DEEPSEEK_API_KEY:-}",
        "api": "openai-completions",
        "models": [
          {
            "id": "deepseek-v4-pro",
            "name": "DeepSeek V4 Pro",
            "reasoning": true,
            "input": ["text"],
            "contextWindow": 160000,
            "maxTokens": 8192
          }
        ]
      },
      "ollama": {
        "apiKey": "ollama-local",
        "baseUrl": "http://192.168.1.4:11434/v1",
        "models": [
          {
            "id": "qwen3.6:35b-a3b-mxfp8",
            "name": "qwen3.6:35b-a3b-mxfp8",
            "contextWindow": 131072
          }
        ]
      },
      "huggingface": {
        "apiKey": "$HF_TOKEN",
        "baseUrl": "https://router.huggingface.co/v1",
        "models": [
          {
            "id": "zai-org/GLM-5.2:novita",
            "name": "zai-org/GLM-5.2:novita",
            "contextWindow": 131072
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
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
}
```

### What This Means

| Tier | Model | Type | Cost |
|------|-------|------|------|
| **Primary** | `deepseek/deepseek-v4-pro` | Cloud API | ~$2.50/M tokens |
| Fallback 1 | `ollama/qwen3.6:35b-a3b-mxfp8` | Local LAN | Free |
| Fallback 2 | `google/gemini-2.5-flash-lite` | Cloud API | Low |
| Fallback 3 | `huggingface/zai-org/GLM-5.2:novita` | Cloud API | Free tier |

**Strategy:** Primary model is deepseek for quality conversations. If it's down or rate-limited, OpenClaw automatically falls back through the chain. For cron jobs (background tasks), we explicitly set the local Ollama model to avoid API costs.

### Ollama Setup (on separate GPU machine)

```bash
# On the GPU machine (192.168.1.4):
ollama pull qwen3.6:35b-a3b-mxfp8
ollama pull qwen3:8b
ollama pull nomic-embed-text

# Expose on LAN:
# Set OLLAMA_HOST=0.0.0.0:11434 in ollama.service
```

### API Keys

Store these as environment variables or in a `.env` file:

```bash
export DEEPSEEK_API_KEY="sk-..."
export TELEGRAM_BOT_TOKEN="123456:ABC..."
export HF_TOKEN="hf_..."
export OPENCLAW_GATEWAY_TOKEN="your-gateway-token"
```

---

## 4. Setting Up Telegram

### 4.1 Create a Telegram Bot

1. Chat with [@BotFather](https://t.me/BotFather) on Telegram
2. Send `/newbot` and follow prompts
3. Save the bot token you receive
4. Set commands if desired: `/setcommands`

### 4.2 Configure OpenClaw for Telegram

In `~/.openclaw/openclaw.json`:

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "streaming": {
        "mode": "off"
      },
      "groups": {
        "*": {
          "requireMention": true
        }
      },
      "botToken": "$TELEGRAM_BOT_TOKEN"
    }
  }
}
```

**Key settings:**
- `requireMention: true` — in groups, the bot only responds when @mentioned (prevents noise)
- `streaming.mode: "off"` — sends complete messages, not characters as typed

### 4.3 Direct Chat vs Groups vs Topics

| Context | How it works |
|---------|-------------|
| **Direct chat** | Bot responds to every message automatically |
| **Group (no topics)** | Bot only responds when @mentioned |
| **Group with topics** | Each topic is a thread — cron jobs can target specific topics |

**Group topic IDs** are visible in the topic link:  
`https://t.me/c/3760976053/10` → chat ID `-1003760976053`, thread ID `10`

### 4.4 Testing

```bash
# Send a test message via Bot API
curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d chat_id="YOUR_CHAT_ID" \
  --data-urlencode text="Hello from your Pi!"
```

---

## 5. Agent Identity & Personality

OpenClaw reads these files on startup:

```
~/.openclaw/workspace/
├── SOUL.md       # Personality, tone, boundaries
├── IDENTITY.md   # Name, role, expertise areas
├── USER.md       # About you (the human)
├── AGENTS.md     # Workspace rules, memory conventions
├── TOOLS.md      # Environment-specific notes (SSH hosts, cameras, etc.)
├── MEMORY.md     # Long-term curated memory
├── HEARTBEAT.md  # Periodic check tasks (optional)
└── memory/       # Daily notes (YYYY-MM-DD.md)
```

### Our IDENTITY.md

```markdown
Name: Hermes
Role: Technical advisor and personal assistant
Expertise: Java, Spring Boot, Quarkus, Kubernetes, AI/LLMs,
           RAG, MCP, Agents, Cloud, DevOps, Raspberry Pi
```

### Our SOUL.md highlights

- Be genuinely helpful, not performatively helpful
- Have opinions
- Be resourceful before asking
- Private things stay private
- In group chats: participate, don't dominate

---

## 6. Installing Skills

OpenClaw skills extend the agent's capabilities. They're like plugins.

### Built-in Skills (40+ available)

Located at `~/.npm-global/lib/node_modules/openclaw/skills/`:
`canvas`, `diagram-maker`, `healthcheck`, `notion`, `taskflow`, `github`, `coding-agent`, `meme-maker`, `voice-call`, and many more.

### Installed Custom Skills

```bash
# These are in ~/.openclaw/workspace/skills/
ls ~/.openclaw/workspace/skills/
# → clawdex, stock-analysis, weather, youtube-full, youtube-watcher
```

| Skill | Purpose |
|-------|---------|
| `youtube-watcher` | Fetch transcripts + summarize YouTube videos |
| `youtube-full` | Full YouTube search, channel browsing, playlists |
| `stock-analysis` | Yahoo Finance data, portfolio tracking, stock scoring |
| `weather` | Current weather and forecasts (no API key) |
| `clawdex` | Security check for ClawHub skills |

### Installing a Skill

```bash
# Via ClawHub
openclaw skills install youtube-watcher

# Or manually — clone into ~/.openclaw/workspace/skills/<name>/
```

---

## 7. Creating Cron Jobs

OpenClaw has a built-in cron system. Jobs run as **isolated agent turns** — they get their own session, execute, and deliver results.

### Our Two Cron Jobs

```bash
openclaw cron list
```

| Job | Schedule | Model | Delivers To |
|-----|----------|-------|-------------|
| **Weekly AI & Tech Earnings Tracker** | Sat 7:00 AM NZST | `ollama/qwen3.6:35b-a3b-mxfp8` | Group topic #2 |
| **Daily GitHub Trends Digest** | Daily 9:30 AM NZST | `ollama/qwen3.6:35b-a3b-mxfp8` | Group topic #10 `#github-trends` |

### Creating a Cron Job via the API

**Example: Daily GitHub Trends Digest**

```json
{
  "name": "Daily GitHub Trends Digest",
  "description": "Daily 9:30 AM NZST — GitHub trending repos filtered by technical interests",
  "schedule": {
    "kind": "cron",
    "expr": "30 9 * * *",
    "tz": "Pacific/Auckland"
  },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "Fetch today's GitHub Trending repositories and deliver a filtered digest.\n\nSTEPS:\n1. Use web_fetch on https://github.com/trending...",
    "model": "ollama/qwen3.6:35b-a3b-mxfp8",
    "timeoutSeconds": 300,
    "toolsAllow": ["web_search", "web_fetch"]
  },
  "delivery": {
    "mode": "announce",
    "channel": "telegram",
    "to": "-1003760976053",
    "threadId": "10"
  }
}
```

### Cron Job Anatomy

| Field | Purpose |
|-------|---------|
| `schedule.kind` | `"cron"` — standard cron expression |
| `schedule.expr` | Cron expr in the specified timezone (not UTC) |
| `schedule.tz` | IANA timezone (e.g., `"Pacific/Auckland"`) |
| `sessionTarget` | `"isolated"` — runs in its own temporary session |
| `payload.kind` | `"agentTurn"` — runs a full agent with the given prompt |
| `payload.model` | Override the default model (use local for cost savings) |
| `payload.toolsAllow` | Restrict which tools the agent can use (security) |
| `delivery.mode` | `"announce"` — sends result to a chat channel |
| `delivery.threadId` | Topic/thread ID for group topics |

### Schedule Types

| Kind | Example | Use Case |
|------|---------|----------|
| `cron` | `"30 9 * * *"` Pacific/Auckland | Daily at 9:30 AM |
| `every` | `{"everyMs": 3600000}` | Every hour |
| `at` | `"2026-07-01T15:00:00"` | One-shot reminder |

### Managing Cron Jobs

```bash
# List all jobs
openclaw cron list

# Get details
openclaw cron get <job-id>

# Force run (test)
openclaw cron run <job-id> --force

# Disable
openclaw cron update <job-id> --enabled false

# Delete
openclaw cron remove <job-id>
```

---

## 8. Migrating OS Cron to OpenClaw Cron

We started with a bash script in the OS crontab. Here's how we migrated it.

### Before (OS Crontab)

```bash
# ~/.openclaw/workspace/github-trends-digest.sh
# A ~100-line bash script that:
#   1. Scrapes github.com/trending with Python
#   2. Sends to Ollama for filtering
#   3. Sends result to Telegram via Bot API (hardcoded token)
#
# crontab entry:
# 30 9 * * * /home/taihoro/.openclaw/workspace/github-trends-digest.sh
```

**Problems with this approach:**
- Hardcoded bot token in a shell script
- Direct Telegram API calls bypass OpenClaw's delivery system
- No error visibility — silent failures in log files
- Dead code paths (two competing scraping methods)
- No automatic retry or fallback

### After (OpenClaw Cron)

1. **Create the OpenClaw cron job** (see Section 7)
2. **Disable the OS crontab entry:**

```bash
crontab -l | sed 's|^30 9.*github-trends|# DISABLED - migrated to OpenClaw cron: &|' | crontab -
```

3. **Test with force run:**

```bash
openclaw cron run <job-id> --force
```

### Benefits

| Aspect | OS Cron | OpenClaw Cron |
|--------|---------|---------------|
| Delivery | Raw Telegram API | Managed via announce |
| Error handling | Silent log files | Visible run status, retries |
| Model selection | Hardcoded in script | Configurable per job |
| Tool access | Limited to curl/Python | Full web_fetch, web_search |
| Maintenance | Shell script debugging | Prompt tweaking, structured config |
| Security | Bot token in plaintext | No tokens in job config |

---

## 9. Full Configuration Reference

### Gateway Settings

```json
{
  "gateway": {
    "mode": "local",
    "port": 18789,
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "$OPENCLAW_GATEWAY_TOKEN"
    },
    "tailscale": {
      "mode": "off"
    }
  }
}
```

**Security notes:**
- Gateway binds to `loopback` only — not exposed to LAN
- Token auth enabled
- Tailscale off by default (enable if you want remote access)

### Model Fallback Chain

```
deepseek-v4-pro (primary)
  ↓ if fails
ollama/qwen3.6:35b (local, free)
  ↓ if fails
google/gemini-2.5-flash-lite (cloud, cheap)
  ↓ if fails
huggingface/GLM-5.2 (cloud, free tier)
```

This means your agent is **highly available** — even if DeepSeek is down, it falls back to your local Ollama instance.

### Tools & Permissions

Cron jobs can be restricted to specific tools:

```json
"toolsAllow": ["web_search", "web_fetch"]
```

This limits what the isolated agent can do — no file writes, no exec, just web access.

---

## 10. Tips & Lessons Learned

### 🔑 API Keys
- Never hardcode tokens in scripts or config files
- Use environment variables: `${DEEPSEEK_API_KEY:-}`
- The `:-` syntax means "use empty string if not set" — safe defaults

### 💰 Cost Optimization
- **Conversations** → use cloud model (quality matters)
- **Background cron jobs** → use local Ollama (free, good enough)
- Set `model` explicitly on cron jobs to avoid billing surprises

### 🧵 Group Topics
- Thread IDs are visible in topic links: `t.me/c/3760976053/10`
- The `to` field for groups needs the `-100` prefix: `-1003760976053`
- Each cron job can target a different topic — great for organization

### 🔄 Fallback Strategy
- Always have a local model in your fallback chain
- Test fallbacks: intentionally break the primary and verify the chain works
- Ollama on LAN is fast enough for cron jobs, might be slow for real-time chat

### 📊 Monitoring
- Check cron run history: `openclaw cron runs <job-id>`
- Run statuses: `ok`, `error`, `skipped`
- Delivery status: `delivered`, `failed`
- Consecutive error counter helps spot persistent issues

### 🛠️ Common Commands Cheat Sheet

```bash
# Gateway
openclaw gateway status          # Is it running?
openclaw gateway restart         # Restart after config changes
openclaw status                  # Full system status

# Cron
openclaw cron list               # All jobs
openclaw cron get <id>           # Job details
openclaw cron run <id> --force   # Test run
openclaw cron runs <id>          # Run history

# Skills
openclaw skills list             # Installed skills
openclaw skills install <name>   # Install from ClawHub

# Memory
openclaw memory index --force    # Rebuild memory index (if broken)

# Sessions
openclaw sessions list           # Active sessions

# Config
openclaw gateway config get      # View current config
```

### 🎯 Architecture Diagram

```
┌─────────────────────────────────────────────────┐
│              Raspberry Pi 5 (16 GB)              │
│                                                   │
│  ┌──────────┐    ┌──────────────────────────┐   │
│  │ OpenClaw  │    │   ~/.openclaw/workspace/  │   │
│  │ Gateway   │◄──►│   SOUL.md, AGENTS.md,     │   │
│  │ :18789    │    │   MEMORY.md, memory/      │   │
│  └────┬─────┘    └──────────────────────────┘   │
│       │                                          │
│  ┌────┴─────────────────────────────────────┐   │
│  │           Cron Scheduler                  │   │
│  │  ┌──────────────────┐ ┌────────────────┐ │   │
│  │  │ GitHub Trends     │ │ Earnings       │ │   │
│  │  │ Daily 9:30 NZST   │ │ Sat 7:00 NZST  │ │   │
│  │  │ → Topic #10       │ │ → Topic #2     │ │   │
│  │  └──────────────────┘ └────────────────┘ │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────┘
                       │
         ┌─────────────┼─────────────┐
         │              │              │
    ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
    │ DeepSeek │   │ Ollama  │   │ Google  │
    │  (API)   │   │ (LAN)   │   │ Gemini  │
    │  Cloud   │   │ 192.168 │   │  (API)  │
    │  $$$     │   │  Free   │   │   $     │
    └─────────┘   └─────────┘   └─────────┘
         │              │              │
         └──────────────┼──────────────┘
                        │
                 ┌──────┴──────┐
                 │  Telegram    │
                 │  ┌────────┐  │
                 │  │ Direct │  │
                 │  │ Group  │  │
                 │  │ Topics │  │
                 │  └────────┘  │
                 └─────────────┘
```

---

## Appendix: Migrating More OS Cron Jobs

If you have other OS crontab entries you want to migrate:

1. **Audit:** `crontab -l` — list all entries
2. **For each job, ask:**
   - What does it do? (fetch data, send notification, run backup?)
   - Can an OpenClaw agent do it with `web_fetch`/`web_search`/`exec`?
   - Would it benefit from error handling, retries, or delivery tracking?
3. **Create the OpenClaw equivalent** as an `agentTurn` cron job
4. **Comment out** the old crontab entry (don't delete immediately)
5. **Run both in parallel** for a few days to compare
6. **Delete** the old entry once confident

---

*This README documents a real, running setup as of June 2026. Your mileage may vary — adapt to your own models, channels, and use cases.*
