# Spec: Match Watcher

## Responsibility

Detects when a registered player has completed a new CS2 match and triggers the stats pipeline.

---

## How CS2 Match Data Is Accessed

CS2 match data requires two layers:

### Layer 1: Match Share Code (via Steam API)
- Each player provides their **Match Auth Code** and **Steam ID** (configured once)
- The Steam Web API endpoint `ICSGOPlayers_730/GetNextMatchSharingCode/v1` returns the latest match share code for a player, given their current known share code
- This gives us a chain: `known_code → next_code → next_code → ...`
- We store the last known share code per player and poll for the next one

**Required from each player:**
- Steam ID (SteamID64 format)
- Match Auth Token (from CS2 game settings → "Game" tab → "Share Match History")
- Last known match share code (or `CSGO-xxxxx-xxxxx-xxxxx-xxxxx-xxxxx` format)

### Layer 2: Match Details (via Steam Game Coordinator)
- Once we have a share code, decode it to a `matchid`, `outcomeid`, `token`
- Use the `globaloffensive` npm package (Node.js Steam GC client) to request full match details
- The GC returns a `matchList` protobuf with per-round, per-player stats
- **Requires a dedicated Steam bot account** (see below) — the GC client logs in as a Steam user

---

## Polling Strategy

```
Every 5 minutes (configurable):
  For each registered player:
    1. Call GetNextMatchSharingCode with (steamId, authCode, lastKnownCode)
    2. If response returns a new code:
       a. Decode the share code → matchId
       b. Check DB: has this matchId already been processed?
       c. If no: add to processing queue
       d. Update lastKnownCode in DB
    3. If API returns "no new match": skip
```

**Rate limiting:** Steam API allows ~100k requests/day. 10 players × 288 polls/day = 2,880 requests. Well within limits.

---

## Share Code Decoder

Match share codes follow a known encoding scheme (base57 encoded integers).

```typescript
interface DecodedShareCode {
  matchId: bigint;
  outcomeId: bigint;
  token: number;
}

function decodeShareCode(shareCode: string): DecodedShareCode
```

Open-source implementations exist (e.g., `csgo-sharecode` npm package).

---

## Match Queue

- In-memory queue with DB persistence
- Prevents duplicate processing on restart
- Feeds into the Stats Processor

```typescript
interface MatchQueueItem {
  matchId: string;
  outcomeId: string;
  token: number;
  triggeredBy: string; // steamId of the player who triggered discovery
  discoveredAt: Date;
  status: 'pending' | 'processing' | 'done' | 'failed';
}
```

---

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Steam API rate limited (429) | Exponential backoff, skip this poll cycle |
| GC request timeout | Retry up to 3 times, then mark as failed |
| Invalid/expired auth code | Log warning, notify Discord channel, continue |
| Duplicate match ID | Skip silently (idempotent) |

---

## Steam Bot Account

The `globaloffensive` GC client must be logged in as a real Steam account. A dedicated throwaway Steam account is strongly recommended so your main account isn't affected by bot activity.

**Setup steps:**
1. Create a new Steam account at `store.steampowered.com`
2. Add CS2 to the account (it's free)
3. Launch CS2 once to initialize the account with the GC (accept the terms)
4. Enable Steam Guard (Email-based is fine; avoids mobile auth complexity)
5. Add credentials to `.env` (see below)

**Bot account behavior:**
- Logs in with `steam-user` (handles auth, sessions, Steam Guard codes)
- Launches CS2 app via the GC (`globaloffensive` package)
- Only requests match data — never plays, never connects to game servers
- The account does NOT need CS2 Prime status

**Steam Guard on first run:**
- First login will email a code to the bot account's email
- The bot prompts for the code in the terminal on first run
- After first login, the session token is saved to `steam-session.json` for reuse

## Configuration

```env
POLL_INTERVAL_SECONDS=300
STEAM_API_KEY=your_steam_web_api_key
STEAM_BOT_USERNAME=your_bot_steam_username
STEAM_BOT_PASSWORD=your_bot_steam_password
```

```json
// players.json
[
  {
    "name": "PlayerName",
    "steamId": "76561198XXXXXXXXX",
    "authCode": "XXXXXXXXXXXXXXXXXX",
    "lastShareCode": "CSGO-XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"
  }
]
```
