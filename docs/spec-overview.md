# CS2 Discord Stats Bot — System Overview

## Purpose

A self-hosted Discord bot for a private group of friends that monitors CS2 matches, fetches post-game stats, and posts formatted player stat cards to a designated Discord channel automatically after each match completes.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        CS2 Stats Bot                         │
│                                                              │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────────┐  │
│  │  Match      │───▶│  Stats       │───▶│  Card          │  │
│  │  Watcher    │    │  Processor   │    │  Renderer      │  │
│  └─────────────┘    └──────────────┘    └────────────────┘  │
│         │                                        │           │
│         ▼                                        ▼           │
│  ┌─────────────┐                       ┌────────────────┐   │
│  │  Steam API  │                       │  Discord Bot   │   │
│  │  + GC Client│                       │  Poster        │   │
│  └─────────────┘                       └────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## Core Components

| Component | Responsibility | Spec Doc |
|-----------|---------------|----------|
| Match Watcher | Polls for new completed matches per registered player | `spec-match-watcher.md` |
| Stats Processor | Parses raw match data into structured stat objects | `spec-stats-processor.md` |
| Card Renderer | Generates visual player stat cards as images | `spec-card-renderer.md` |
| Discord Bot | Posts cards + match summaries to a channel | `spec-discord-bot.md` |
| Player Registry | Stores Steam IDs and auth codes for tracked players | `spec-player-registry.md` |

---

## Tech Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Runtime | Node.js (TypeScript) | Strong ecosystem for both Discord and Steam APIs |
| CS2 Data | Steam Web API + `steam-user` + `globaloffensive` npm | Access match history and live GC match data via dedicated Steam bot account |
| Discord | `discord.js` v14 | Well-maintained, slash command support |
| Image Generation | `puppeteer` + HTML/CSS templates | Screenshot-based card rendering; easy to style and iterate visually |
| Storage | SQLite via `better-sqlite3` | Lightweight, file-based, no infra needed for a friend group |
| Config | `.env` + JSON config file | Simple secrets + player registry |

---

## Data Flow

```
1. [Every 5 min] Match Watcher checks each registered player's recent match history
2. New match share code detected → fetch full match data via Steam GC
3. Match data parsed into per-player stat objects
4. Card Renderer generates one PNG per player
5. Discord Bot posts match summary embed + one card per player to the channel
6. Match ID stored in DB to prevent duplicate posts
```

---

## Deployment

- Runs as a single long-running Node.js process on a **local machine**
- Assumes always-on or near-always-on PC (the bot misses matches while it's off)
- Env vars for all secrets (bot token, Steam credentials, API keys)
- `pm2` recommended to keep the process alive and auto-restart on crash/reboot

---

## Scope Constraints (V1)

- Private use only, no multi-server support
- Max ~10 registered players
- Supports standard matchmaking and Premier mode only (no FACEIT/ESL in V1)
- No web dashboard — Discord-only interface
