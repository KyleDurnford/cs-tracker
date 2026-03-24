# Spec: Card Renderer

## Responsibility

Generates a PNG player stat card image for each registered player in a match, styled similarly to Breaking Point cards from CDL — clean, bold, visually distinctive per performance tier.

---

## Card Dimensions

- **Size:** 800 × 450 px (16:9 landscape, fits Discord embeds well)
- **Format:** PNG
- **DPI:** 144 (2x retina)

---

## Card Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  [TIER GLOW BORDER]                                             │
│                                                                 │
│  ┌───────────┐   PLAYERNAME              MAP NAME              │
│  │           │   ──────────────          PREMIER / COMP        │
│  │  AVATAR   │   K  / D  / A            DATE                   │
│  │           │   24 / 8  / 3                                   │
│  │  (steam   │                                                  │
│  │   pfp)    │   ┌────────────────────────────────────────┐    │
│  └───────────┘   │  ADR   HS%   K/D   RATING   MVPS      │    │
│                  │  87.4  64%   3.00   1.41      3        │    │
│                  └────────────────────────────────────────┘    │
│                                                                 │
│  ┌──────────────────────────┐   ┌─────────────────────────┐   │
│  │  MULTI-KILL BADGES       │   │  HIGHLIGHT CALLOUT      │   │
│  │  2K×4  3K×2  4K×1  ACE×1│   │  "1 ACE  |  3 CLUTCHES" │   │
│  └──────────────────────────┘   └─────────────────────────┘   │
│                                                                 │
│  [WIN / LOSS / DRAW badge]    [SCORE  13 - 5]    [TIER badge]  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Visual Tiers

Each performance tier has a distinct color palette applied to the border glow, accent lines, and tier badge:

| Tier | Primary Color | Glow | Background Tint |
|------|--------------|------|-----------------|
| S    | `#FFD700` Gold | Strong warm glow | Dark amber |
| A    | `#00FF88` Green | Medium green glow | Dark green |
| B    | `#4DA6FF` Blue | Soft blue glow | Dark slate |
| C    | `#AAAAAA` Gray | None | Dark gray |
| D    | `#555555` Dim | None | Near-black |

Background: dark textured base (`#0D1117` to `#1A1F2E` gradient) with CS2-themed subtle overlay (e.g., crosshair watermark, map silhouette at low opacity).

---

## Typography

| Element | Font | Size | Weight |
|---------|------|------|--------|
| Player name | "Rajdhani" or "Barlow Condensed" | 42px | 700 |
| Main stats (K/D/A) | Same | 56px | 800 |
| Stat labels | Same | 18px | 400 |
| Tier badge | Same | 32px | 900 |
| Map/mode | Same | 22px | 500 |

Fonts loaded from bundled files (no runtime font fetching).

---

## Avatar

- Fetched from Steam CDN using the player's avatar URL (retrieved alongside steam profile data)
- Displayed as a circle-clipped image, 160×160 px
- Fallback: generic silhouette icon

---

## Win/Loss Badge

| Result | Text | Color |
|--------|------|-------|
| Win    | WIN  | Green `#00FF88` |
| Loss   | LOSS | Red `#FF4444` |
| Draw   | DRAW | Gray `#888888` |

---

## Multi-Kill Badges

Pill-shaped badges for each multi-kill type earned:

```
[ 2K ×4 ]  [ 3K ×2 ]  [ ACE ×1 ]
```

Only shown if count > 0. "ACE" badge gets a gold accent regardless of tier.

---

## Renderer API

```typescript
interface CardRenderOptions {
  stats: PlayerMatchStats;
  match: ProcessedMatch;
  avatarBuffer: Buffer | null;
}

async function renderPlayerCard(options: CardRenderOptions): Promise<Buffer>
// Returns PNG buffer ready for Discord upload
```

---

## Implementation Options

### Option A: `@napi-rs/canvas` (Recommended for V1)
- Pure Node.js Canvas API, no browser required
- Fast, low memory footprint
- Draw everything programmatically
- Con: More verbose layout code

### Option B: `puppeteer` + HTML template
- Design card as HTML/CSS, screenshot it
- Easier to style, easier to iterate visually
- Con: Requires Chromium (~300MB), slower render (~1-2s per card)
- Good for V2 if visual complexity increases

**V1 recommendation:** Start with `@napi-rs/canvas` for simplicity and minimal dependencies.
