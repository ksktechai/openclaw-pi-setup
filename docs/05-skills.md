# 5. Skills Guide

Skills are installable abilities the agent can call. Four are enabled in this setup: two YouTube
skills, weather, and stock analysis. This page covers what each does, how they differ, and the one
genuinely fiddly part — the `yt-dlp` / `uv` dependency the YouTube skills need.

```bash
npx openclaw skills list             # what's installed and enabled
npx openclaw skills install <ref>    # install from ClawHub
```

A walkthrough that mirrors this YouTube setup: <https://www.youtube.com/watch?v=NqY0wF4YKXo>.

---

## 5.1 The enabled skills

| Skill | Install ref | What it does | External deps |
|-------|-------------|--------------|---------------|
| `youtube-full` | `youtube-full` (via ClawHub) | Search YouTube, browse channels and playlists | Transcript API key, `yt-dlp` |
| `youtube-watcher` | `@michaelgathara/youtube-watcher` | Fetch a video's transcript and summarize it | Transcript API key, `yt-dlp` |
| `weather` | `@steipete/weather` | Current conditions + forecast | none (uses wttr.in / Open-Meteo) |
| `stock-analysis` | `@udiedrichsen/stock-analysis` | Yahoo Finance data, portfolio tracking, scoring | none |

Enable a skill in `openclaw.json` under `skills.entries` (and restart the gateway):

```json
"skills": {
  "entries": {
    "weather":        { "enabled": true },
    "stock-analysis": { "enabled": true }
  }
}
```

---

## 5.2 youtube-full vs. youtube-watcher

They sound similar but solve different problems — they're complementary, not duplicates:

- **`youtube-full`** — *discovery*. Search YouTube, list a channel's uploads, walk a playlist. Use it
  to *find* videos ("latest from this channel", "top videos about X").
- **`youtube-watcher`** — *consumption*. Given a specific video, pull its transcript and summarize it.
  Use it to *digest* a video you already have the URL for.

A natural pairing: `youtube-full` finds candidates, `youtube-watcher` summarizes the one you pick.

### Transcript API key

Both rely on a transcript service. Sign up at <https://transcriptapi.com/signup>, then put the key in
`~/.openclaw/.env`:

```bash
TRANSCRIPT_API_KEY=your-transcriptapi-key
```

Restart the gateway so it picks up the new variable.

---

## 5.3 weather and stock-analysis

- **`weather`** needs no API key — it pulls from free services (wttr.in / Open-Meteo). You can sanity-
  check the data source directly: `curl -s "wttr.in/Auckland?T"`.
- **`stock-analysis`** pulls from Yahoo Finance for quotes, portfolio tracking, and a scoring model.
  Also keyless.

These two are the easy wins — install, enable, restart, done.

---

## 5.4 The `yt-dlp` / `uv` dependency (the fiddly part)

The YouTube skills shell out to **`yt-dlp`**. Installing it is where Debian Trixie pushes back, and
where the gateway daemon adds a second twist. Both are covered in detail in Troubleshooting; the
short version:

**1. Don't `pip install` it.** On Debian Trixie, `pip3 install yt-dlp` fails with
`externally-managed-environment` (PEP 668). Install [`uv`](https://docs.astral.sh/uv/) and use it
instead:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
uv --version
uv tool install yt-dlp        # installs yt-dlp into ~/.local/bin
```

See [Troubleshooting §1](06-troubleshooting.md#1-pep-668--externally-managed-environment).

**2. Make sure the daemon can find it.** `uv tool install` puts `yt-dlp` on *your interactive shell's*
PATH (`~/.local/bin`), but the **gateway daemon doesn't inherit that PATH** — so the skill can fail
with "command not found" even though `yt-dlp` runs fine when you type it. Put `~/.local/bin` on the
daemon's PATH (or reference `yt-dlp` by absolute path).

See [Troubleshooting §2](06-troubleshooting.md#2-the-daemon-cant-find-uv--yt-dlp-path-inheritance).

After installing the dependency and fixing PATH, restart and confirm:

```bash
npx openclaw gateway restart
npx openclaw skills list
```

---

## 5.5 Vetting skills before you install (clawdex)

Skills run code on your Pi, so it's worth checking one before enabling it. The `clawdex` skill (and
its API) returns a security assessment for a ClawHub skill:

```bash
npx openclaw skills install @wearekoi/clawdex
curl -s "https://clawdex.koi.security/api/skill/youtube-full"
curl -s "https://clawdex.koi.security/api/skill/weather"
```

Treat a skill like any third-party dependency — glance at what it does and what it can reach before
turning it on.

---

Next: **[6. Troubleshooting](06-troubleshooting.md)** for every error referenced above.
