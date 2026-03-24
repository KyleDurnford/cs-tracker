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

## Implementation: Puppeteer + HTML/CSS

Cards are rendered by generating an HTML string from a template, loading it in a headless Chromium instance via `puppeteer`, and screenshotting the result.

### Why Puppeteer
- Card layout is pure HTML/CSS — easy to iterate visually without recompiling
- CSS box shadows handle tier glow natively
- Web fonts (`@font-face`) load from bundled files
- Flexbox/grid for layout is far less painful than canvas coordinate math
- Easy to preview cards in a browser during development

### Render Flow

```
1. Build data context object from PlayerMatchStats + ProcessedMatch
2. Inject into HTML template via string interpolation
3. Launch puppeteer (reuse single browser instance across renders)
4. Open new page, setContent(html), wait for fonts/images
5. page.screenshot({ type: 'png', clip: { width: 800, height: 450 } })
6. Close page, return Buffer
```

### Browser Instance Management

- One shared `Browser` instance is created at startup and reused
- Each card render opens a new `Page`, takes the screenshot, then closes the page
- Browser is restarted if it crashes (event listener on `disconnected`)
- Typical render time: ~300–700ms per card on local hardware

### Template Structure

```
src/
  renderer/
    card.template.html   ← master HTML/CSS template with {{placeholders}}
    card.renderer.ts     ← puppeteer orchestration + template injection
    assets/
      fonts/             ← bundled .woff2 font files
      bg-texture.png     ← dark background texture
      crosshair.svg      ← watermark overlay
```

### Template Injection

Simple `{{key}}` placeholder replacement (no template engine dependency):

```typescript
function injectTemplate(template: string, data: CardTemplateData): string {
  return template.replace(/\{\{(\w+)\}\}/g, (_, key) => String(data[key] ?? ''));
}
```

### Avatar Handling

- Steam avatar URL is fetched and converted to a base64 data URI
- Embedded directly in the HTML (`<img src="data:image/jpeg;base64,...">`)
- Avoids puppeteer needing external network access during render
- Fallback: inline SVG silhouette icon

### Fonts

Bundled locally as `.woff2` files, loaded via `@font-face` in the template CSS.
Recommended: **Barlow Condensed** (free, Google Fonts, supports weight 400–900).
