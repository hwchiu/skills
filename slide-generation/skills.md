# Slide Generation — Design System & Style Guide

> Reusable design conventions for programmatic slide decks built with **pptxgenjs** in Node.js. This guide is topic-agnostic — apply it to any presentation subject.

---

## 1. Project Structure

```
src/
  design-system.js   # Colors, fonts, theme switching (single source of truth)
  helpers.js         # All shared slide-building utilities
  partN.js           # Each part is a standalone script generating its own .pptx
  merge.js           # Merges multiple part .pptx files into one final deck
output/              # Generated .pptx files
```

### How It Works

- Each **part file** is a standalone Node.js script that imports `design-system.js` and `helpers.js`, builds slides, and saves a `.pptx` file.
- Parts are independent — run `node src/partN.js` to generate one part, or `node src/merge.js` to rebuild and merge all parts into a final deck.
- To create a new presentation, create a new part file following the same pattern: require the design system, call `setTheme()`, define `buildSlideN()` functions, and export via `main()`.

---

## 2. Theme System

### Switching Themes

Every part file must call `setTheme("light")` immediately after requiring `design-system.js`:

```js
const { COLORS, FONTS, setTheme } = require("./design-system");
setTheme("light");
```

This mutates the shared `COLORS` object. Safe because each part runs in a separate `node` process.

### Available Themes

| Theme | Background | Text | Best For |
|-------|-----------|------|----------|
| `dark` | `0D1117` (near-black) | `E6EDF3` (light grey) | Screen reading, dark rooms |
| `light` | `FFFDF8` (warm cream) | `2D2926` (dark brown) | **Projector / classroom** (current default) |

### Color Token Reference (light theme)

| Token | Hex | Usage |
|-------|-----|-------|
| `COLORS.bg` | `FFFDF8` | Slide background |
| `COLORS.bg2` | `F5F1EB` | Card/panel backgrounds |
| `COLORS.bg3` | `ECE6DD` | Secondary card fills, column headers |
| `COLORS.border` | `D6CCBF` | Card borders, separators |
| `COLORS.text` | `2D2926` | Primary text |
| `COLORS.textMuted` | `8A8078` | Secondary/meta text |
| `COLORS.accent` | `4A7FB5` | Links, highlights, default borders |
| `COLORS.success` | `4A9968` | Good/advantage indicators |
| `COLORS.danger` | `C4605B` | Bad/limitation indicators |
| `COLORS.warning` | `B8892C` | Warning indicators |

### Component Color Assignments

| Component | Token | Hex | Visual |
|-----------|-------|-----|--------|
| Frontend / Nginx | `COLORS.frontend` | `5B8DB8` | Muted blue |
| Backend / App | `COLORS.backend` | `5A9B6E` | Sage green |
| Database | `COLORS.database` | `C4804A` | Warm orange |
| LB / Cache / MQ | `COLORS.infra` | `8B6AB5` | Soft purple |
| Container / Pod | `COLORS.container` | `4A9B8E` | Teal |
| Client / Browser | `COLORS.client` | `908880` | Warm grey |

**Rule:** Never hardcode hex colors. Always use `COLORS.*` tokens.

---

## 3. Canvas & Layout

- **Canvas size:** 10 x 5.5 inches (pptxgenjs widescreen default)
- **All coordinates in inches** — `x`, `y`, `w`, `h` are floats
- **Background:** Always `slide.background = { color: COLORS.bg }`
- **Header height:** 0.52 inches (`HEADER_H` constant)
- **Bottom panel height:** 1.95 inches (`BOTTOM_H` constant)

### Layout Principles

1. **Fill the slide** — no large empty areas. Content should span the full canvas.
2. **Readable text** — minimum font size 9pt for body, 7.5pt only for labels/tags.
3. **Use PowerPoint native text** — Excalidraw is for diagrams/images only. All text goes through `slide.addText()` for easy future editing.
4. **No page numbers** — partLabel should be a short section identifier (e.g. `"PART 1"`), not `"03 / 50"`.

---

## 4. Helper Functions Reference

All helpers are in `src/helpers.js`. Import what you need:

```js
const {
  W, H, HEADER_H, BOTTOM_H, BOTTOM_Y,
  initSlide, addSlideHeader, addBottomPanel,
  addNodeCard, addMiniNode, addHArrow, addDashedHArrow, addVArrow,
  addZoneBorder, addAlertBar, addTipBar, addCommentBar, addKnowledgeCards,
  addCompareHeading, addCompareItem, addSummaryCard, addMetricCard,
  addThreeCols, addCodeCard,
} = require("./helpers");
```

### Key Helpers

| Helper | Purpose | Key Options |
|--------|---------|-------------|
| `initSlide(pres)` | Create slide with background | Returns slide object |
| `addSlideHeader(slide, pres, opts)` | Header bar with title, part label, complexity meter | `title`, `partLabel`, `complexity`, `accentColor` |
| `addBottomPanel(slide, pres, pros, cons, opts)` | Two-column pros/cons panel | Items can be `string` or `{title, sub}` |
| `addNodeCard(slide, pres, opts)` | Rounded service node | `emoji`, `name`, `meta`, `borderColor`, `nameColor` |
| `addMiniNode(slide, pres, opts)` | Small inline node | `emoji`, `label`, `borderColor` |
| `addHArrow(slide, pres, opts)` | Horizontal arrow with pill-badge label | `x`, `y`, `w`, `label`, `color` |
| `addDashedHArrow(slide, pres, opts)` | Dashed arrow for responses | `reverse` for direction |
| `addVArrow(slide, pres, opts)` | Vertical arrow | `h` can be negative for upward |
| `addZoneBorder(slide, pres, opts)` | Dashed border around a group | `label` for tab on top |
| `addTipBar(slide, pres, opts)` | Insight bar at bottom | `y`, `text` |
| `addAlertBar(slide, pres, opts)` | Red alert bar | `message`, `tags[]` |
| `addCommentBar(slide, pres, opts)` | `// code-comment` style bar | `message`, `sub` |
| `addKnowledgeCards(slide, pres, cards, opts)` | Bottom knowledge cards (3-column) | `[{title, body, color}]` |
| `addCompareHeading(slide, pres, opts)` | Comparison section heading | `type: "good"/"bad"` |
| `addCompareItem(slide, pres, opts)` | Comparison list row | `emoji`, `title`, `sub`, `type` |
| `addSummaryCard(slide, pres, opts)` | Summary card with bullet list | `icon`, `title`, `items[]`, `color` |
| `addMetricCard(slide, pres, opts)` | Big number KPI card | `value`, `label`, `sub`, `color` |
| `addThreeCols(slide, pres, cols, opts)` | Three-column layout | `[{title, color, icon, items[]}]` |
| `addCodeCard(slide, pres, opts)` | Dark code snippet card | `code`, `language` |

---

## 5. Visual Style Guide (Engineer Handbook Style)

The current style is inspired by a "terminal / engineer handbook" aesthetic optimized for classroom projectors.

### Arrow Labels
- Use **pill-badge labels** on arrows (rounded rect background with protocol text)
- Arrow labels use `FONTS.code` (Consolas) for technical protocol names
- Example: `addHArrow(slide, pres, { label: "HTTP", color: COLORS.frontend })`

### Node Cards
- Use **color-matched names** via `nameColor` option
- Node border and name text should use the same component color
- Example: `addNodeCard(slide, pres, { name: "Nginx", borderColor: COLORS.frontend, nameColor: COLORS.frontend })`

### Insight Bars
- Use `addCommentBar()` for code-comment style (`// insight message`)
- Use `addTipBar()` for tips at slide bottom
- Use `addAlertBar()` for pain points and warnings

### Knowledge Cards
- Use `addKnowledgeCards()` for bottom 3-column educational content
- Each card has a color-coded top accent strip
- Keep titles short, body text concise

### Response Arrows
- Use `addDashedHArrow()` for response/reverse flows
- Dashed lines visually distinguish request from response

### Typography

| Element | Font | Size | Weight |
|---------|------|------|--------|
| Slide title | Calibri | 16pt | Bold |
| Part label | Calibri | 8.5pt | Normal |
| Body text | Calibri | 10.5pt | Bold for titles |
| Sub-text | Calibri | 9pt | Normal |
| Code | Consolas | 9.5pt | Normal |
| Labels/tags | Consolas | 7.5pt | Bold |
| Big metrics | Calibri | 34pt | Bold |

---

## 6. Slide Template Patterns

These are reusable layout patterns. Mix and match for any topic:

| Pattern | Use Case | Key Elements |
|---------|----------|-------------|
| Cover | Opening slide | Big title text, optional journey/outline cards on right |
| Section Opener | Start of a new section | Large section number, bullet list of upcoming topics |
| Architecture Diagram | System/flow diagrams | Diagram area top 65%, pros/cons bottom 35% via `addBottomPanel` |
| Comparison | Side-by-side evaluation | Left/right columns via `addCompareHeading` + `addCompareItem` |
| Concept Explainer | Teaching a concept | Large icon/emoji left, 3-4 bullet points right |
| Code Walkthrough | Showing code | Dark code card via `addCodeCard`, optional diagram beside it |
| Summary | Section recap | 2x3 card grid via `addSummaryCard` |
| Three-Column | Categorized content | Three equal columns via `addThreeCols` |
| Metrics/KPI | Key numbers | Big number cards via `addMetricCard` |

---

## 7. Language & Content

- **All user-visible slide text should be in English** unless the presentation specifically targets a non-English audience.
- Code snippets inside `addCodeCard` should use English comments for international readability.
- Keep text concise — slides are visual aids, not documents.

---

## 8. Build & QA

### Build Commands

```bash
# Generate a single part
node src/<part-file>.js      # outputs output/<part-file>.pptx

# Merge all parts into final deck
node src/merge.js            # outputs output/cloud_native_slides_final.pptx
```

### QA Checklist

- Architecture diagrams use icons/emojis, not ASCII art
- Component colors match the design system table
- No text overflow or element overlap
- Slide fills the canvas — no large empty areas
- Fonts render correctly (Calibri for text, Consolas for code)
- If complexity meter is used, it progresses logically across slides

### No Test Suite

Visual QA only. Open the generated `.pptx` files in PowerPoint/Google Slides to verify rendering.

---

## 9. Shape Types

Always use `pres.ShapeType.*` for shapes:
- `pres.ShapeType.rect` — rectangle
- `pres.ShapeType.roundRect` — rounded rectangle
- `pres.ShapeType.ellipse` — circle/ellipse
- `pres.ShapeType.line` — line/arrow

The `pres` object must be passed into helper functions for shape access.

---

## 10. Template Master Support (Optional)

The system supports two modes — **with or without** a `.pptx` template:

| Mode | When | What Happens |
|------|------|-------------|
| **No template** | `templates/` is empty | Uses built-in design-system colors, helpers draw all chrome (header, background, etc.) |
| **With template** | `.pptx` file in `templates/` | Post-processes with python-pptx to apply master layouts; helpers skip elements the master provides |

### When No Template Exists (Default)

Everything works as described in this guide — `design-system.js` defines colors, `helpers.js` draws headers/backgrounds/footers, slides are self-contained.

### When a Template Is Provided

#### One-Time Setup

1. **Place template** in `templates/` (e.g., `templates/school.pptx`)

2. **Analyze it** — extract layouts, colors, safe zones:
   ```python
   from pptx import Presentation
   tmpl = Presentation("templates/school.pptx")
   for layout in tmpl.slide_layouts:
       print(f"Layout: {layout.name}")
       for ph in layout.placeholders:
           print(f"  Placeholder {ph.placeholder_format.idx}: {ph.name}")
   ```

3. **Update `design-system.js`** — match the template's palette and fonts

4. **Define safe zones** — the content area that doesn't overlap master chrome:
   ```js
   const SAFE = { x: 0.5, y: 0.8, w: 9.0, h: 4.2 };
   ```

5. **Adjust `helpers.js`** — skip elements the master already provides (background, logo, footer); fit content within safe zones

#### Build Flow

```
node src/partN.js                    → output/partN.pptx (content only)
python3 src/apply-template.py        → output/partN.pptx (with master applied)
node src/merge.js                    → output/final.pptx
```

The `apply-template.py` step is **skipped entirely** when no template exists.

#### Template Checklist

When receiving a new template:
- [ ] Extract layout names and placeholder positions
- [ ] Map template colors → `design-system.js` tokens
- [ ] Map template fonts → `FONTS` object
- [ ] Measure safe zones (content area boundaries)
- [ ] Identify which elements the master provides (skip in helpers)
- [ ] Test with a single slide before full generation

---

## 11. Creating a New Presentation

To start a new deck on any topic:

1. Create `src/myTopic.js` (or split into multiple part files for large decks)
2. Add the boilerplate:
   ```js
   const { COLORS, FONTS, setTheme } = require("./design-system");
   setTheme("light");
   const { initSlide, addSlideHeader, /* ...helpers you need */ } = require("./helpers");
   ```
3. Define `buildSlideN(pres)` functions for each slide
4. Write `async function main()` that creates `new pptxgen()`, calls each builder, and saves
5. Run `node src/myTopic.js` to generate
6. If a template exists in `templates/`, run `python3 src/apply-template.py` to apply masters
7. If merging multiple parts, add the file path to `merge.js`

### Commit Convention

```
feat: add <topic> slides (<description>)
fix: correct <issue> in <file>
style: update theme/colors
```
