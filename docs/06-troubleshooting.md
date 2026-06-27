# 6. Troubleshooting

Every entry here is a real error from this build, with the symptom you'll actually see, why it
happens, and the fix. They're roughly in the order I hit them.

| # | Problem |
|---|---------|
| 1 | [PEP 668 / externally-managed-environment](#1-pep-668--externally-managed-environment) |
| 2 | [The daemon can't find uv / yt-dlp (PATH inheritance)](#2-the-daemon-cant-find-uv--yt-dlp-path-inheritance) |
| 3 | [Context-overflow / compaction failure](#3-context-overflow--compaction-failure) |
| 4 | [models.json vs openclaw.json provider confusion](#4-modelsjson-vs-openclawjson-provider-confusion) |
| 5 | [HuggingFace token "Bearer" prefix](#5-huggingface-token-bearer-prefix) |
| 6 | [Cron jobs fire in the wrong timezone](#6-cron-jobs-fire-in-the-wrong-timezone) |
| 7 | [Cron job runs on the wrong (expensive) model](#7-cron-job-runs-on-the-wrong-expensive-model) |

---

## 1. PEP 668 / externally-managed-environment

**Symptom** — installing the YouTube skills' dependency fails:

```bash
pip3 install yt-dlp
# error: externally-managed-environment
# × This environment is externally managed
```

**Cause** — Debian 13 (Trixie), like recent Debian/Ubuntu, marks the system Python as
*externally managed* ([PEP 668](https://peps.python.org/pep-0668/)). System `pip` refuses to install
into it so it can't fight with `apt`-managed packages.

**Fix** — don't use system `pip` for CLI tools. Install [`uv`](https://docs.astral.sh/uv/) and let it
manage `yt-dlp` in its own isolated location:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env          # put uv on PATH for this shell
uv --version
uv tool install yt-dlp               # installs the yt-dlp CLI into ~/.local/bin
```

(`pipx`, or a dedicated venv, would also work — `uv` was chosen here for speed. Avoid
`pip install --break-system-packages`; it does exactly what it says.)

---

## 2. The daemon can't find uv / yt-dlp (PATH inheritance)

**Symptom** — `yt-dlp` works perfectly when *you* run it, but a YouTube skill invoked through the
gateway fails with something like `yt-dlp: command not found` or a non-zero exit from the skill.

**Cause** — `uv tool install` puts `yt-dlp` in `~/.local/bin`, which is on **your interactive shell's**
PATH (you added it via `source $HOME/.local/bin/env`, typically from `.bashrc`/`.profile`). The
**OpenClaw gateway daemon does not run inside your interactive shell**, so it never sources those
files and doesn't see `~/.local/bin`. Tools that work in your terminal are invisible to the daemon.

**Fix** — make the tool reachable from the daemon's own environment. Any of:

- **Add `~/.local/bin` to the daemon's PATH.** If the gateway runs under systemd, set it in the unit
  (or a drop-in):

  ```ini
  # ~/.config/systemd/user/<gateway-unit>.service.d/path.conf  (example)
  [Service]
  Environment=PATH=%h/.local/bin:/usr/local/bin:/usr/bin:/bin
  ```

  Then `systemctl --user daemon-reload` and restart the gateway.

- **Ensure the daemon starts from a login shell** that sources your profile (so `~/.local/bin/env`
  runs), if you launch it that way.

- **Reference `yt-dlp` by absolute path** (`~/.local/bin/yt-dlp`) where the skill lets you.

**Verify** the daemon's view, not just yours — restart the gateway and exercise the skill end-to-end
(`npx openclaw gateway restart`, then trigger a YouTube summary). Confirming `which yt-dlp` in your own
shell proves nothing about the daemon.

---

## 3. Context-overflow / compaction failure

**Symptom** — a long session dies with an error like:

```text
prompt too large (precheck)
```

and auto-compaction can't recover it.

**Cause** — the conversation grew past the model's usable context window. OpenClaw tries to
auto-compact (summarize) the history to fit, but if the prompt is already over the limit at the
pre-flight check, compaction can't run — the request is rejected before it starts. A `contextWindow`
set larger than the model actually supports makes this worse: OpenClaw thinks there's room that
isn't there.

**Fix** — two parts:

1. **Clear the session** to drop the oversized history and start fresh.
2. **Set a sane `contextWindow`** on the model in `models.providers[...]models[]` — match it to what
   the model genuinely supports (e.g. `131072` for the Ollama qwen model, `160000` for DeepSeek),
   not an aspirational number. See [Config files §3.2](03-config-files.md#32-the-providers-block).

Honest context limits let auto-compaction trigger *before* the hard ceiling, so sessions degrade
gracefully instead of hitting the precheck wall.

---

## 4. models.json vs openclaw.json provider confusion

**Symptom** — you edit model/provider settings, restart, and nothing changes. You're sure you changed
the right thing, but the agent ignores it.

**Cause** — you're editing the wrong file. There are two:

- `~/.openclaw/openclaw.json` — the **global** config. This is where `models.providers` (DeepSeek,
  Ollama, HuggingFace), the model allowlist, and `model.primary`/`fallbacks` live.
- `~/.openclaw/agents/main/agent/models.json` — **per-agent** model state for the `main` agent. It is
  *not* where you hand-declare providers.

I lost time hand-editing `models.json` expecting it to define providers; provider definitions belong
in `openclaw.json`.

**Fix** — make provider/model changes in `~/.openclaw/openclaw.json`, then:

```bash
jq . ~/.openclaw/openclaw.json       # confirm it's valid JSON first
npx openclaw gateway restart
npx openclaw doctor --fix
```

If an edit "isn't taking," confirm the file path before changing anything else. See
[Config files §3.1](03-config-files.md#31-two-files-two-jobs).

---

## 5. HuggingFace token "Bearer" prefix

**Symptom** — the HuggingFace provider returns `401 Unauthorized` even though the token is correct.

**Cause** — the HTTP client adds the `Authorization: Bearer <token>` header for you. If you store the
token already prefixed with `Bearer ` (copied that way from a docs snippet), the header becomes
`Bearer Bearer hf_...` and auth fails.

**Fix** — store the **raw** token only, no prefix:

```bash
# ~/.openclaw/.env
HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxxxxxx     # ✅ raw token
# HF_TOKEN=Bearer hf_xxxx...             # ❌ never include "Bearer"
```

Restart the gateway after fixing it. The provider's `apiKey` in `openclaw.json` references this var as
`"$HF_TOKEN"` — see [Config files §3.4](03-config-files.md#34-env-references-and-interpolation).

---

## 6. Cron jobs fire in the wrong timezone

**Symptom** — a job scheduled for, say, 09:30 runs in the middle of the night.

**Cause** — the cron expression was evaluated in a different timezone than you assumed (often UTC)
because `schedule.tz` was omitted. `30 9 * * *` means 09:30 *in whatever zone the scheduler uses*.

**Fix** — always set an explicit IANA timezone on the job:

```json
"schedule": { "kind": "cron", "expr": "30 9 * * *", "tz": "Pacific/Auckland" }
```

See [Cron jobs §4.4](04-cron-jobs.md#44-gotcha-set-the-timezone-explicitly).

---

## 7. Cron job runs on the wrong (expensive) model

**Symptom** — scheduled jobs you expected to be free start showing up on your cloud-provider bill.

**Cause** — the job didn't pin a model, so it inherited `agents.defaults.model.primary` — the paid
cloud model used for interactive chat — instead of the free local one.

**Fix** — set `payload.model` explicitly on **every** cron job, and make sure that model is in the
[allowlist](03-config-files.md#33-allowlist-vs-primaryfallbacks):

```json
"payload": {
  "kind": "agentTurn",
  "model": "ollama/qwen3.6:35b-a3b-mxfp8",
  "message": "...",
  "toolsAllow": ["web_search", "web_fetch"]
}
```

`payload.model` is independent of `model.primary` — that's the feature that lets background work run
on the free local model while chat stays on the cloud primary. See
[Cron jobs §4.3](04-cron-jobs.md#43-cron-jobs-pin-their-own-model).
