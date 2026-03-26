# Spec: Card Renderer

## Responsibility

Generates a PNG player stat card for each registered player in a match, styled as a **Pokémon-style trading card**. The map is the card art. Performance determines rarity — a great game produces a full art holographic; a bad game is a plain common.

---

## Card Dimensions

- **Size:** 500 × 700 px (5:7 ratio — standard Pokémon card proportions)
- **Format:** PNG
- **DPI:** 144 (2x retina)

---

## Card Layout (Standard)

```
┌────────────────────────────────────┐
│ PLAYERNAME            ADR: 87 HP   │  ← header bar (frame color = card type)
│ [TYPE ICON]  Rifler · Premier      │
├────────────────────────────────────┤
│ ┌──────────────────────────────┐   │
│ │                              │   │
│ │   [MAP ART — fills box]      │   │  ← map screenshot/art as illustration
│ │                              │   │
│ │              ┌──────────┐    │   │
│ │              │  AVATAR  │    │   │  ← player avatar, bottom-right of art
│ │              └──────────┘    │   │
│ └──────────────────────────────┘   │
│  MapName · 13–5 · Win · 2026-03-26 │  ← flavor/species line
├────────────────────────────────────┤
│ ─────────────────────────────────  │
│ [⚡][⚡]  Flamethrower       180   │  ← Move 1 name + damage value
│   Deployed 3 molotovs dealing an   │
│   average of 60 damage each.       │
│ ─────────────────────────────────  │
│ [🔥][🔥]  Headhunter          24  │  ← Move 2 name + damage value
│   Eliminated 24 enemies with a     │
│   64% headshot rate.               │
├────────────────────────────────────┤
│ Weakness: ●    K / D / A           │  ← bottom stats row
│ ◆ UNCOMMON   Match #1234   1.41★  │  ← rarity + match id + rating
└────────────────────────────────────┘
```

---

## Card Rarity

Rarity is determined by the player's HLTV-style rating for that match:

| Rarity | Symbol | Rating | Frame Style |
|--------|--------|--------|-------------|
| Common | ● | < 0.80 | Plain beige/gray frame, no effects |
| Uncommon | ◆ | 0.80 – 1.00 | Lightly colored frame, subtle border |
| Rare | ★ | 1.00 – 1.20 | Metallic sheen on frame via CSS gradient |
| Double Rare | ★★ | 1.20 – 1.40 | Animated shimmer border, foil accent lines |
| Full Art Holo | ★★★ | ≥ 1.40 | Map art bleeds to card edge, rainbow prismatic CSS overlay, no inner illustration border |

### Full Art Holographic Layout

When rarity is ★★★, the card art covers the entire card face edge-to-edge. All text elements (header, moves, footer) are overlaid on a semi-transparent frosted glass panel so they remain legible over the art. A rainbow shimmer gradient is animated across the card via CSS `@keyframes`.

```css
/* Holo shimmer — applied as ::after overlay on full-art cards */
@keyframes holoShimmer {
  0%   { background-position: 0% 50%; }
  50%  { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}
.holo-overlay {
  background: linear-gradient(
    125deg,
    rgba(255,0,128,0.15) 0%,
    rgba(255,200,0,0.15) 20%,
    rgba(0,255,128,0.15) 40%,
    rgba(0,128,255,0.15) 60%,
    rgba(200,0,255,0.15) 80%,
    rgba(255,0,128,0.15) 100%
  );
  background-size: 300% 300%;
  animation: holoShimmer 4s ease infinite;
  mix-blend-mode: color-dodge;
}
```

---

## Card Type

Each player is assigned a card type based on their dominant playstyle in the match. The type determines the frame color and energy icons used in the moves section.

| Type | Icon | Color | Trigger Condition |
|------|------|-------|-------------------|
| Fire | 🔥 | `#FF6B35` | Highest stat is molotov/utility damage |
| Lightning | ⚡ | `#FFD700` | Highest stat is kill count or KD ratio |
| Water | 💧 | `#4DA6FF` | Highest stat is flash assists or assists |
| Psychic | 🔮 | `#CC44FF` | Majority of kills are AWP kills |
| Dark | 🌑 | `#555588` | Highest stat is clutch wins |
| Colorless | ⬜ | `#BBBBBB` | No single dominant stat (balanced all-rounder) |

---

## Moves

Two moves are shown on each card, selected from the player's top two standout stats for the match. Each move has a name, flavor text description, and a damage/power number.

### Move Pool

| Stat Category | Move Name | Damage Value | Flavor Text Template |
|---------------|-----------|-------------|----------------------|
| Kill count | **Death Sentence** | kills | "Eliminated {{kills}} enemies without mercy." |
| ADR | **Suppression Fire** | ADR (rounded) | "Averaged {{adr}} damage per round, keeping enemies pinned." |
| Headshot % | **Headhunter** | kill count | "Took down {{kills}} enemies with a {{hs}}% headshot rate." |
| Molotov damage | **Flamethrower** | molotov dmg | "Deployed {{molotovs}} molotovs dealing {{molotovDmg}} total damage." |
| Flash assists | **Blinding Light** | flash assists | "Blinded enemies to secure {{flashAssists}} assisted kills." |
| AWP kills | **Dead Eye** | AWP kills | "Picked off {{awpKills}} targets with surgical precision." |
| Clutch wins | **Clutch Gene** | clutch wins | "Converted {{clutchWins}} clutch situations against the odds." |
| MVP rounds | **Round Commander** | MVPs | "Named MVP in {{mvps}} rounds, dictating the pace of play." |
| Utility damage | **Grenadier** | util dmg | "Dealt {{utilDmg}} utility damage across the match." |
| Assists | **Team Player** | assists | "Contributed {{assists}} assists, setting up teammates to close out." |
| Entry kills | **First Blood** | entry kills | "Opened {{entryKills}} rounds with the first kill of the round." |
| Ace (any) | **Nuclear Strike** | 5 | "Single-handedly eliminated the entire enemy team." *(shown if any ace earned)* |

**Selection logic:** Rank each player stat against its historical average (or a fixed threshold). The two stats furthest above average become Move 1 and Move 2. If an ace was earned, Nuclear Strike always takes Move 1 slot.

---

## Map Art

The card illustration area displays the CS2 map the match was played on. Sources (in priority order):

1. **Bundled map artwork** — a set of pre-cropped, stylized map images stored in `assets/maps/<mapname>.jpg` (one per official map). These are the primary source and avoid any network call at render time.
2. **Fallback** — a solid dark gradient with the map name rendered in large text as a watermark.

Map name is normalized from the match data (e.g., `de_mirage` → `mirage`).

---

## Visual Tiers (Frame Colors)

Card frame and inner border color comes from the combination of card type + rarity:

| Rarity | Frame Base | Border Effect |
|--------|-----------|---------------|
| Common | `#C8A97B` Tan | None |
| Uncommon | Type color (desaturated) | Thin accent line |
| Rare | Type color (full) | Inner metallic gradient |
| Double Rare | Type color + silver shimmer | Animated pulse |
| Full Art Holo | Type color + rainbow overlay | Full holo animation |

---

## Typography

| Element | Font | Size | Weight |
|---------|------|------|--------|
| Player name | Gill Sans / Futura (Pokemon-style sans) | 22px | 700 |
| HP / ADR label | Same | 14px | 400 |
| ADR value | Same | 22px | 900 |
| Move name | Same | 16px | 700 |
| Move flavor text | Same | 12px | 400, italic |
| Damage number | Same | 20px | 900 |
| Footer stats | Same | 11px | 400 |
| Rarity / rating | Same | 11px | 600 |

Font: **Gill Sans Nova** or **Nunito** (both free, readable at small sizes, rounded like Pokemon card text).

---

## Avatar

- Steam avatar embedded as base64 data URI (prevents puppeteer network calls)
- Displayed in the **bottom-right corner of the illustration box**, 80×80 px, circle-clipped
- Thin border ring in the card type's color
- Fallback: inline SVG silhouette

---

## Win/Loss Treatment

| Result | Effect |
|--------|--------|
| Win | Green `#22C55E` accent on score in footer |
| Loss | Red `#EF4444` accent on score in footer |
| Draw | Gray `#888888` accent |

No separate badge — the score line in the flavor text already conveys the result.

---

## Renderer API

```typescript
interface CardRenderOptions {
  stats: PlayerMatchStats;
  match: ProcessedMatch;
  avatarBuffer: Buffer | null;
  mapImageBuffer: Buffer | null; // pre-loaded map art, null = use fallback
}

async function renderPlayerCard(options: CardRenderOptions): Promise<Buffer>
// Returns PNG buffer ready for Discord upload
```

---

## Implementation: Puppeteer + HTML/CSS

Cards are rendered by injecting data into an HTML template, loading it in headless Chromium via `puppeteer`, and screenshotting the result.

### Render Flow

```
1. Build CardTemplateData from PlayerMatchStats + ProcessedMatch
2. Select card type, rarity, and two moves (selection logic above)
3. Convert avatar + map image to base64 data URIs
4. Inject all values into HTML template via {{placeholder}} replacement
5. Launch puppeteer (reuse single browser instance across renders)
6. Open new Page, page.setContent(html), waitForNetworkIdle
7. page.screenshot({ type: 'png', clip: { x:0, y:0, width:500, height:700 } })
8. Close page, return Buffer
```

### Browser Instance Management

- One shared `Browser` instance created at startup, reused for all renders
- Each render opens a new `Page`, screenshots it, then closes it
- Browser auto-restarts on `disconnected` event
- Typical render time: ~300–700ms per card on local hardware

### Template Structure

```
src/
  renderer/
    card.template.html      ← HTML/CSS card template with {{placeholders}}
    card.template.holo.html ← Full art variant (art bleeds to edges)
    card.renderer.ts        ← puppeteer orchestration + template injection
    moves.ts                ← move selection logic
    assets/
      fonts/                ← bundled .woff2 font files
      maps/                 ← map art images (mirage.jpg, dust2.jpg, etc.)
```

### Template Injection

Simple `{{key}}` replacement, no template engine dependency:

```typescript
function injectTemplate(template: string, data: CardTemplateData): string {
  return template.replace(/\{\{(\w+)\}\}/g, (_, key) => String(data[key] ?? ''));
}
```

### Fonts

Bundled locally as `.woff2` files, loaded via `@font-face` in the template CSS. No runtime font fetching.
