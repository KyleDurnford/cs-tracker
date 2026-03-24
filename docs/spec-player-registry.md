# Spec: Player Registry

## Responsibility

Stores and manages the list of tracked players — their Steam identifiers, auth tokens, and last known match state.

---

## Player Record

```typescript
interface RegisteredPlayer {
  // Identity
  steamId: string;           // SteamID64, e.g. "76561198XXXXXXXXX"
  displayName: string;       // Friendly name used in Discord posts
  discordUserId?: string;    // Optional: link to a Discord user for @mentions

  // CS2 API access
  authCode: string;          // Match history auth token from CS2 settings
  lastShareCode: string;     // Last known CSGO-XXXXX-... share code

  // State tracking
  addedAt: Date;
  lastSeenMatchAt: Date | null;
  active: boolean;           // Can be disabled without deleting
}
```

---

## Storage

Stored in two places:

### 1. `players.json` (initial config / seed file)
Human-editable flat file for easy setup. Loaded on first run.

```json
[
  {
    "steamId": "76561198XXXXXXXXX",
    "displayName": "Frag Machine",
    "authCode": "XXXXXXXXXXXXXXXXXX",
    "lastShareCode": "CSGO-XXXXX-XXXXX-XXXXX-XXXXX-XXXXX",
    "discordUserId": "123456789012345678"
  }
]
```

### 2. SQLite `players` table (runtime state)
- Source of truth at runtime
- `lastShareCode` updated here as matches are discovered
- `players.json` is only read on first startup / when no DB exists

---

## SQLite Schema

```sql
CREATE TABLE players (
  steam_id        TEXT PRIMARY KEY,
  display_name    TEXT NOT NULL,
  discord_user_id TEXT,
  auth_code       TEXT NOT NULL,
  last_share_code TEXT NOT NULL,
  added_at        TEXT NOT NULL,
  last_seen_match TEXT,
  active          INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE processed_matches (
  match_id    TEXT PRIMARY KEY,
  processed_at TEXT NOT NULL,
  posted      INTEGER NOT NULL DEFAULT 0
);
```

---

## Auth Code Privacy

- Auth codes are stored only in the local SQLite DB and `players.json`
- Neither file should be committed to version control (both in `.gitignore`)
- The auth code only grants access to read match history — it cannot modify the Steam account
- Recommend players rotate their auth code periodically (it's easy to regenerate in CS2)

---

## Adding Players

**Via slash command (recommended):**
```
/register steamId:76561198XXX authCode:XXXXXXXXXX shareCode:CSGO-XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
```

**Via `players.json`** (then restart or `/reload`):
Edit the JSON file manually, then restart the bot or run `/forcepoll`.

---

## How to Get Your Auth Code and Share Code

1. Open CS2
2. Go to **Settings → Game**
3. Enable "Share Match History" → copy the **Auth Code**
4. Go to **Your Profile → Matches** → click any recent match → copy the **Share Code** from the URL or share button
5. Your **Steam ID** can be found at `steamidfinder.com` or via your Steam profile URL

---

## Data Retention

- Match records kept indefinitely (small footprint: ~1KB per match)
- Future: configurable retention window + aggregate stats rollup
