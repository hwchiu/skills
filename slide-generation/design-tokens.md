# Design Tokens

> Single source of truth for all colors, fonts, and theme configuration. Never hardcode visual values — always reference tokens.

## Theme System

Call `setTheme()` at the top of every part file, immediately after `require`:

```js
const { COLORS, FONTS, setTheme } = require("./design-system");
setTheme("light");   // "light" for projector, "dark" for screen
```

**How it works:** `setTheme()` mutates the shared `COLORS` object via `Object.assign()`. Safe because each part file runs as a separate `node` process.

## Themes

| Theme | bg | text | Best For |
|-------|----|------|----------|
| `light` | `FFFDF8` warm cream | `2D2926` dark brown | **Projector / classroom** (default) |
| `dark` | `0D1117` near-black | `E6EDF3` light grey | Screen / dark room |

## Color Tokens

### Surface Colors

| Token | Light | Dark | Usage |
|-------|-------|------|-------|
| `COLORS.bg` | `FFFDF8` | `0D1117` | Slide background |
| `COLORS.bg2` | `F5F1EB` | `161B22` | Card backgrounds, panels |
| `COLORS.bg3` | `ECE6DD` | `1C2128` | Column headers, secondary fills |
| `COLORS.border` | `D6CCBF` | `30363D` | Card borders, separators |
| `COLORS.cardBg` | `F5F1EB` | `161B22` | Generic card fill |
| `COLORS.cardSuccess` | `E5F0E9` | `0F2A1A` | Pros/advantage card fill |
| `COLORS.cardDanger` | `F8ECEC` | `2A0F0F` | Cons/limitation card fill |
| `COLORS.cardWarn` | `F3EDE0` | `2A1F00` | Warning card fill |

### Text Colors

| Token | Light | Dark | Usage |
|-------|-------|------|-------|
| `COLORS.text` | `2D2926` | `E6EDF3` | Primary text |
| `COLORS.textMuted` | `8A8078` | `8B949E` | Secondary text, metadata |
| `COLORS.circleText` | `FFFFFF` | `FFFFFF` | Text inside filled circles |

### Semantic Colors

| Token | Light | Dark | Usage |
|-------|-------|------|-------|
| `COLORS.accent` | `4A7FB5` | `58A6FF` | Links, highlights, default accent |
| `COLORS.success` | `4A9968` | `3FB950` | Good/advantage/positive |
| `COLORS.danger` | `C4605B` | `F85149` | Bad/limitation/negative |
| `COLORS.warning` | `B8892C` | `D29922` | Caution, attention |

### Component Colors

These identify specific infrastructure/architecture components. Use consistently across ALL slides so viewers build visual associations.

| Component | Token | Light | Dark | When to use |
|-----------|-------|-------|------|-------------|
| Frontend / Nginx / Web | `COLORS.frontend` | `5B8DB8` | `1F6FEB` | Web servers, reverse proxies, static assets |
| Backend / App Server | `COLORS.backend` | `5A9B6E` | `238636` | Application logic, API servers |
| Database | `COLORS.database` | `C4804A` | `E36209` | Any data store (SQL, NoSQL, Redis) |
| Infra (LB/Cache/MQ) | `COLORS.infra` | `8B6AB5` | `6E40C9` | Load balancers, message queues, caches |
| Container / Pod | `COLORS.container` | `4A9B8E` | `0D8A6C` | Docker, Kubernetes pods |
| Client / Browser | `COLORS.client` | `908880` | `8B949E` | End users, browsers, mobile |
| CDN | `COLORS.cdn` | `4A917E` | `1A7F64` | CDN, edge servers |

### Internal Tokens (used by helpers.js)

| Token | Light | Dark | Used by |
|-------|-------|------|---------|
| `COLORS.shadowColor` | `9E9488` | `000000` | Card drop shadows |
| `COLORS.shadowOpacity` | `0.15` | `0.3` | Shadow transparency |
| `COLORS.meterBg` | `D6CCBF` | `252D38` | Complexity meter track |
| `COLORS.tipBg` | `EAF0F7` | `0A1929` | Tip bar background |
| `COLORS.dangerTagBg` | `F8ECEC` | `3D1515` | Alert bar tag background |
| `COLORS.codeBg` | `2D2926` | `0D1117` | Code card background |

## Fonts

| Token | Value | Usage |
|-------|-------|-------|
| `FONTS.title` | `Calibri` | Slide titles, headings, big numbers |
| `FONTS.body` | `Calibri` | Body text, bullet points, labels |
| `FONTS.code` | `Consolas` | Code snippets, protocol labels, technical tags |

## Typography Scale

| Element | Font | Size | Weight | Color |
|---------|------|------|--------|-------|
| Cover title | `FONTS.title` | 72pt | Bold | `COLORS.text` or `COLORS.accent` |
| Slide title (header) | `FONTS.title` | 16pt | Bold | `COLORS.text` |
| Section label | `FONTS.body` | 8.5pt | Normal | `COLORS.textMuted` |
| Body text / card title | `FONTS.body` | 10.5pt | Bold | `COLORS.text` |
| Body sub-text | `FONTS.body` | 9pt | Normal | `COLORS.textMuted` |
| Code | `FONTS.code` | 9.5pt | Normal | `COLORS.text` |
| Arrow/tag labels | `FONTS.code` | 7.5pt | Bold | Matches arrow color |
| Complexity label | `FONTS.body` | 7.5pt | Normal | `COLORS.textMuted` |
| Big metric number | `FONTS.title` | 34pt | Bold | Component color |
| Metric label | `FONTS.body` | 12pt | Bold | `COLORS.text` |

## Rules

1. **Never write raw hex** — always `COLORS.xxx`
2. **pptxgenjs format** — hex strings WITHOUT `#` prefix: `"4A7FB5"` not `"#4A7FB5"`
3. **Component colors are semantic** — a Database node is ALWAYS `COLORS.database`, never `COLORS.backend`
4. **Use `nameColor` to match border** — when a node card has `borderColor: COLORS.frontend`, also set `nameColor: COLORS.frontend`
5. **Keep the two themes in sync** — if you add a new token to `light`, add it to `dark` too
