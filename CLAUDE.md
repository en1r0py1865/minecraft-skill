# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenClaw skill pack for Minecraft Java Edition. Contains four independent skills that are installed individually into `~/.openclaw/skills/` and triggered by semantic matching in OpenClaw conversations.

## Architecture

Skills are organized in a **three-tier architecture** based on connectivity requirements:

- **Tier 1 (Offline knowledge):** `minecraft-wiki`, `minecraft-building-server` — no external dependencies, pure reference material
- **Tier 2 (Bridge-dependent):** `minecraft-bridge` — local HTTP service (`bridge-server.js`) wrapping Mineflayer for live bot control via REST API on `localhost:3001`
- **Tier 3 (RCON-independent):** `minecraft-server-admin` — connects directly to Minecraft server console via RCON TCP (port 25575), completely independent of the bridge

Each skill directory follows the same structure:
- `SKILL.md` — YAML frontmatter (name, description, metadata) + skill body (activation rules, runtime instructions, safety rules)
- `references/` — detailed reference docs loaded on-demand during skill execution
- `scripts/` — executable scripts (Node.js, shell)

### Key Design Principles

- **No cross-skill runtime dependencies at framework level.** OpenClaw triggers skills independently; skills that need bridge data must health-check `GET /status` themselves and degrade gracefully.
- **SKILL.md frontmatter `description` is the trigger filter** — keep it focused on when to activate. Dependency checks and execution logic go in the SKILL.md body.
- **Progressive disclosure**: SKILL.md bodies reference `references/` files rather than inlining all content.

## Commands

```bash
# Bridge server
node minecraft-bridge/bridge-server.js          # start bridge (requires mineflayer deps)
bash minecraft-bridge/scripts/start.sh           # convenience wrapper with PID file
bash minecraft-bridge/scripts/stop.sh            # stop bridge

# RCON commands
node minecraft-server-admin/scripts/rcon.js "list"           # execute RCON command
node minecraft-server-admin/scripts/log-analyzer.js          # parse server logs

# Publishing (requires clawhub CLI)
clawhub publish ./<skill-dir> --slug <skill-name> --version 1.0.0
```

## Environment Variables

Bridge: `MC_HOST`, `MC_PORT`, `MC_BOT_USERNAME`, `MC_BRIDGE_PORT` (default 3001), `MC_VERSION`, `MC_AUTH`

RCON: `MC_RCON_HOST`, `MC_RCON_PORT`, `MC_RCON_PASSWORD`, `MC_SERVER_LOG`

## Bridge API

`bridge-server.js` is a single-file Node.js HTTP server (no framework). Route dispatch is a handler map keyed by `"METHOD /path"`. All responses are JSON with `success` field. All endpoints except `GET /status` require bot connection (503 if disconnected). Auto-reconnect on disconnect with 5s interval.

## Content Language

Reference docs and dependency evaluation are written in a mix of English and Chinese. SKILL.md files default to English. Match the user's language in responses.
