# Spec: Discord Bot

## Responsibility

Posts match results and player stat cards to a designated Discord channel. Provides slash commands for manual triggers and configuration.

---

## Bot Setup

- Framework: `discord.js` v14
- Bot requires only the following permissions:
  - `Send Messages`
  - `Attach Files`
  - `Embed Links`
  - `Read Message History`
- No message content intent needed (slash commands only)

---

## Match Post Format

When a match is detected, the bot posts **one embed per tracked player** plus a **match summary embed**.

### Match Summary Embed

```
┌────────────────────────────────────────────┐
│  🎮  de_dust2  |  Premier                  │
│  Team Score: 13 – 8  |  CT Win             │
│  Played: March 24, 2026 at 9:41 PM         │
│  Duration: 47 min                          │
│  Tracked players: Player1, Player2         │
└────────────────────────────────────────────┘
```

Fields: Map, Mode, Score, Winner, Date/Time, Duration

### Player Card Post

Each tracked player who participated gets:
- The rendered PNG card uploaded as an attachment
- A small embed beneath linking to the Steam profile (optional)

**Post order:** Best-performing player card first (by rating), descending.

---

## Slash Commands

| Command | Description |
|---------|-------------|
| `/stats @player` | Show that player's last match card |
| `/laststats` | Show everyone's last match (re-posts last cards) |
| `/register <steamId> <authCode> <shareCode>` | Add a player to tracking (admin only) |
| `/unregister <steamId>` | Remove a player from tracking (admin only) |
| `/players` | List all registered tracked players |
| `/checkstatus` | Show bot status, last poll time, queue size |
| `/forcepoll` | Manually trigger a poll cycle (admin only) |

---

## Channel Configuration

```env
DISCORD_BOT_TOKEN=your_bot_token
DISCORD_GUILD_ID=your_server_id
DISCORD_CHANNEL_ID=your_stats_channel_id
```

The bot only posts to the configured channel. All slash commands are registered guild-wide (not globally) for faster availability.

---

## Duplicate Prevention

- The bot stores each `matchId` in SQLite after posting
- On startup, loads all posted match IDs into memory
- Before posting, checks: "have we posted this matchId?" — if yes, skip

---

## Error Notifications

If a match fails to process (e.g., GC timeout), the bot posts a brief error notice to the channel:

```
⚠️  Match [id] fetch failed for PlayerName. Will retry.
```

---

## Rate Limiting Considerations

- Discord allows 5 messages per 5 seconds per channel
- Multi-player match posts are queued and spaced 1s apart
- Card images are under 8MB (typical: ~200–500KB), within Discord's free tier limit

---

## Admin Role

Commands marked "admin only" check that the invoking user has a role named `CS-Bot-Admin` or is the server owner. Configurable via env var:

```env
DISCORD_ADMIN_ROLE_NAME=CS-Bot-Admin
```
