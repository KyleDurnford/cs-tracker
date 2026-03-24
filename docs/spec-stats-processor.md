# Spec: Stats Processor

## Responsibility

Takes raw match data from the Steam GC response and transforms it into clean, display-ready stat objects per player.

---

## Input

Raw protobuf match data from the `globaloffensive` GC client, containing:
- Match metadata (map, mode, date, duration, score)
- Per-round data (winner, bomb events, etc.)
- Per-player data (kills, deaths, assists, headshots, damage, etc.)

---

## Output: `ProcessedMatch`

```typescript
interface ProcessedMatch {
  matchId: string;
  map: string;
  mode: 'Premier' | 'Competitive' | 'Deathmatch' | 'Other';
  playedAt: Date;
  durationSeconds: number;
  score: { ct: number; t: number };     // final half scores
  winner: 'CT' | 'T' | 'Draw';
  players: PlayerMatchStats[];
}
```

---

## Output: `PlayerMatchStats`

```typescript
interface PlayerMatchStats {
  steamId: string;
  name: string;
  // Team
  team: 'CT' | 'T';
  // Core stats
  kills: number;
  deaths: number;
  assists: number;
  kd: number;                    // kills / deaths
  // Damage
  adr: number;                   // avg damage per round
  totalDamage: number;
  // Accuracy
  headshots: number;
  headshotPercent: number;       // hs / kills
  // Multi-kills
  oneKills: number;
  twoKills: number;
  threeKills: number;
  fourKills: number;
  fiveKills: number;             // aces
  // Impact
  mvpCount: number;
  utilDamage: number;
  flashAssists: number;
  // Rating
  hltv2Rating: number;           // computed approximation
  // Performance tier for card styling
  performanceTier: 'S' | 'A' | 'B' | 'C' | 'D';
}
```

---

## HLTV 2.0 Rating Approximation

A simplified rating formula based on public knowledge of HLTV rating components:

```
Rating = 0.0073 * KAST
       + 0.3591 * KPR
       - 0.5329 * DPR
       + 0.2372 * Impact
       + 0.0032 * ADR
       + 0.1587
```

Where:
- `KAST` = % of rounds with Kill, Assist, Survived, or Traded
- `KPR` = kills per round
- `DPR` = deaths per round
- `Impact` = (2.13 × KPR) + (0.42 × APR) - 0.41
- `ADR` = average damage per round

*Note: This is a known public approximation — not exact HLTV data.*

---

## Performance Tier

Used to style the player card with a color/glow:

| Tier | Rating Range | Card Style |
|------|-------------|------------|
| S    | ≥ 1.20      | Gold glow  |
| A    | 1.05–1.19   | Green glow |
| B    | 0.90–1.04   | Blue       |
| C    | 0.75–0.89   | Gray       |
| D    | < 0.75      | Dim/muted  |

---

## Best Stat Highlights

Additional computed fields for card callouts:

```typescript
interface StatHighlights {
  bestStat: string;        // e.g. "3 Aces" | "87 ADR" | "71% HS"
  clutchesWon: number;     // 1v1, 1v2, etc. — if available from GC data
  openingKills: number;    // first kills in round
}
```

---

## Tracked Player Filter

Only players in the registry (`players.json`) get individual cards. Other players in the same match are processed for context (e.g., full scoreboard) but don't get cards posted.

---

## DB Storage

Processed matches are stored in SQLite for:
- Deduplication (prevent double-posting)
- Future: aggregate stats, leaderboards, streaks
