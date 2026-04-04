# Helpers API Reference

> Complete documentation for every exported function in `src/helpers.js`. Each entry includes all parameters, default values, and usage examples.

## Canvas Constants

These internal constants define the slide geometry. They are not exported but affect all helpers.

```js
const W        = 10;      // slide width (inches)
const H        = 5.5;     // slide height (inches)
const HEADER_H = 0.52;    // standard header height
const BOTTOM_H = 1.95;    // two-column bottom panel height
const BOTTOM_Y = H - BOTTOM_H; // 3.55 — top of bottom panel
```

---

## Slide Scaffolding

### `initSlide(pres)`

Creates a new blank slide with the theme background color applied.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `pres` | PptxGenJS | ✅ | The presentation object |

**Returns:** `slide` object

```js
const slide = initSlide(pres);
// Sets: slide.background = { color: COLORS.bg }
```

---

### `addSlideHeader(slide, pres, opts)`

Adds a complete header bar: accent strip, title, optional subtitle, optional complexity meter, optional part label, and divider line.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.title` | string | `""` | Main slide title (16pt bold) |
| `opts.subtitle` | string | `""` | Subtitle below title (8.5pt muted) |
| `opts.partLabel` | string | `""` | Top-right label, e.g. `"PART 1"` |
| `opts.accentColor` | string | `COLORS.accent` | Left accent strip color |
| `opts.complexity` | number\|null | `null` | 1–10 scale; `null` hides meter |
| `opts.maxComplexity` | number | `10` | Max value for complexity meter |

**Spatial layout:**
- Header bar: `x=0, y=0, w=10, h=0.52`
- Accent strip: `x=0, y=0.09, w=0.05, h=0.34`
- Title: `x=0.18, valign=middle`
- Complexity meter: `x=6.75, y=0.12`
- Part label: `x=7.8, align=right`
- Divider line: `y=0.52`

```js
addSlideHeader(slide, pres, {
  title: "Starting Point: The Simplest Deployment",
  partLabel: "PART 1",
  accentColor: COLORS.backend,
  complexity: 3,
});
```

---

### `addBottomPanel(slide, pres, pros, cons, opts)`

Draws a two-column pros/cons panel at the bottom of the slide.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `pros` | string[] or {title,sub}[] | `[]` | Left column items (green) |
| `cons` | string[] or {title,sub}[] | `[]` | Right column items (red) |
| `opts.y` | number | `3.55` | Top Y position of panel |
| `opts.h` | number | `1.95` | Panel height |

**Spatial layout:**
- Left column (pros): `x=0, w=5, fill=cardSuccess`
- Right column (cons): `x=5, w=5, fill=cardDanger`
- Items start `y + 0.3`, spaced `0.28"` apart

```js
addBottomPanel(slide, pres, [
  "Simple deployment",
  "Easy debugging",
], [
  "Single Point of Failure",
  "Cannot scale independently",
], { y: 3.3, h: 2.2 });
```

---

## Node Cards

### `addNodeCard(slide, pres, opts)`

Draws a component card with emoji icon, name, and metadata. Used for servers, databases, load balancers, etc.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.x` | number | — | X position (required) |
| `opts.y` | number | — | Y position (required) |
| `opts.w` | number | `1.3` | Width |
| `opts.h` | number | `1.05` | Height |
| `opts.emoji` | string | `""` | Top icon (e.g. `"🌐"`, `"⚙️"`) |
| `opts.name` | string | `""` | Component name (bold, 10.5pt) |
| `opts.meta` | string | `""` | Subtitle/port info (muted, 8pt) |
| `opts.borderColor` | string | `COLORS.accent` | Border color |
| `opts.nameColor` | string | `COLORS.text` | Name text color (set to match border for emphasis) |

```js
addNodeCard(slide, pres, {
  x: 2.4, y: 0.98, w: 2.0, h: 1.8,
  emoji: "🌐", name: "Frontend", meta: "Nginx :80",
  borderColor: COLORS.frontend, nameColor: COLORS.frontend,
});
```

---

### `addMiniNode(slide, pres, opts)`

Compact inline node — single row with emoji + label. Used inside zone borders or as secondary elements.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.x` | number | — | X position (required) |
| `opts.y` | number | — | Y position (required) |
| `opts.w` | number | `1.15` | Width |
| `opts.h` | number | `0.38` | Height |
| `opts.emoji` | string | `""` | Left emoji (12pt) |
| `opts.label` | string | `""` | Text label (bold) |
| `opts.borderColor` | string | `COLORS.border` | Border color |

```js
addMiniNode(slide, pres, {
  x: 6.8, y: 0.9, w: 1.1, h: 0.45,
  emoji: "⚙️", label: "Server",
  borderColor: COLORS.backend,
});
```

---

## Arrows & Connectors

### `addHArrow(slide, pres, opts)`

Horizontal arrow with a pill-badge label (engineer handbook style).

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.x` | number | — | Start X (required) |
| `opts.y` | number | — | Y position (required) |
| `opts.w` | number | `0.55` | Arrow width/length |
| `opts.label` | string | `""` | Protocol/action label (e.g. `"HTTP"`, `"SQL"`) |
| `opts.color` | string | `COLORS.accent` | Arrow and badge color |

**Spatial:** Line at `y + 0.15`. Pill badge at `y - 0.04`, height `0.22"`.

```js
addHArrow(slide, pres, { x: 4.47, y: 1.75, w: 0.56, label: "proxy", color: COLORS.frontend });
```

---

### `addDashedHArrow(slide, pres, opts)`

Dashed horizontal arrow — used for response/return flows.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.x` | number | — | Start X (required) |
| `opts.y` | number | — | Y position (required) |
| `opts.w` | number | `0.55` | Arrow width |
| `opts.label` | string | `""` | Response label |
| `opts.color` | string | `COLORS.accent` | Color |

```js
addDashedHArrow(slide, pres, { x: 4.47, y: 2.2, w: 0.56, label: "response", color: COLORS.frontend });
```

---

### `addVArrow(slide, pres, opts)`

Vertical arrow — top to bottom connector.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.x` | number | — | X position (required) |
| `opts.y` | number | — | Start Y (required) |
| `opts.h` | number | `0.4` | Arrow height |
| `opts.label` | string | `""` | Label text |
| `opts.color` | string | `COLORS.accent` | Color |

```js
addVArrow(slide, pres, { x: 3.0, y: 2.5, h: 0.5, label: "deploy", color: COLORS.container });
```

---

## Zones & Borders

### `addZoneBorder(slide, pres, opts)`

Dashed rounded rectangle used to group components (e.g., a VM, K8s namespace, VPC).

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.x` | number | — | X position (required) |
| `opts.y` | number | — | Y position (required) |
| `opts.w` | number | — | Width (required) |
| `opts.h` | number | — | Height (required) |
| `opts.color` | string | `COLORS.border` | Border color |
| `opts.label` | string | `""` | Label text (top-left) |

```js
addZoneBorder(slide, pres, {
  x: 2.15, y: 0.75, w: 7.65, h: 2.6,
  color: COLORS.backend, label: "ubuntu-01",
});
```

---

## Information Bars

### `addTipBar(slide, pres, opts)`

Blue accent tip bar with 💡 icon. Full-width.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.text` | string | — | Tip text (required) |
| `opts.y` | number | — | Y position (required) |

```js
addTipBar(slide, pres, { y: 5.0, text: "Design your TTL strategy carefully" });
```

---

### `addAlertBar(slide, pres, opts)`

Red danger alert bar with ⚠️ icon. Full-width.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.text` | string | — | Alert text (required) |
| `opts.y` | number | — | Y position (required) |

```js
addAlertBar(slide, pres, { y: 4.8, text: "This approach has significant drawbacks" });
```

---

### `addCommentBar(slide, pres, text, opts)`

Grey italic code-comment style bar (`// comment`).

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `text` | string | — | Comment text (required) |
| `opts.y` | number | — | Y position |

```js
addCommentBar(slide, pres, "This pattern is also called the Sidecar pattern", { y: 4.5 });
```

---

## Comparison Helpers

### `addCompareHeading(slide, pres, opts)`

Section heading for a comparison column (colored background strip).

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.x` | number | — | X position (required) |
| `opts.y` | number | — | Y position (required) |
| `opts.w` | number | — | Width (required) |
| `opts.label` | string | — | Heading text (required) |
| `opts.type` | string | `"bad"` | `"good"` (green) or `"bad"` (red) |

```js
addCompareHeading(slide, pres, { x: 0.3, y: 0.62, w: 4.4, label: "❌  Before Containers", type: "bad" });
addCompareHeading(slide, pres, { x: 5.2, y: 0.62, w: 4.4, label: "✅  With Containers", type: "good" });
```

---

### `addCompareItem(slide, pres, opts)`

Single comparison row with emoji icon, title, and optional subtitle.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.x` | number | — | X position (required) |
| `opts.y` | number | — | Y position (required) |
| `opts.w` | number | `4.3` | Width |
| `opts.emoji` | string | `"✓"` | Icon emoji |
| `opts.title` | string | — | Main text (bold, 10.5pt) |
| `opts.sub` | string | `null` | Subtitle (9pt muted) |
| `opts.type` | string | `"neutral"` | `"good"`, `"bad"`, `"warning"`, `"neutral"` |

**Row height:** `0.56"` with sub, `0.4"` without.

```js
addCompareItem(slide, pres, {
  x: 0.3, y: 2.48, w: 4.4,
  emoji: "✓", title: "No Code Changes", sub: "Simply upgrade machine specs",
  type: "good",
});
```

---

## Card Helpers

### `addSummaryCard(slide, pres, opts)`

Summary card for recap slides — icon + title + bullet list.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.x` | number | — | X position (required) |
| `opts.y` | number | — | Y position (required) |
| `opts.w` | number | `2.1` | Width |
| `opts.h` | number | `1.85` | Height |
| `opts.icon` | string | `"📌"` | Top emoji (22pt) |
| `opts.title` | string | `""` | Card title (11pt bold) |
| `opts.items` | string[] | `[]` | Bullet items (each gets `•` prefix) |
| `opts.color` | string | `COLORS.accent` | Border and title color |
| `opts.status` | string | `null` | Optional status icon in top-right (`"✅"`, `"❌"`) |

```js
addSummaryCard(slide, pres, {
  x: 0.3, y: 0.65, w: 4.5, h: 2.25,
  icon: "🖥️", title: "Single-Server Deployment", color: COLORS.backend,
  items: ["Simple setup", "SPOF risk", "Cannot scale"],
});
```

---

### `addMetricCard(slide, pres, opts)`

Big-number KPI card with value, label, and sub-text.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.x` | number | — | X position (required) |
| `opts.y` | number | — | Y position (required) |
| `opts.w` | number | `2.8` | Width |
| `opts.h` | number | `1.4` | Height |
| `opts.value` | string | — | Large display value (34pt bold) |
| `opts.label` | string | — | Label below value (12pt bold) |
| `opts.sub` | string | `""` | Small sub-text (9pt muted) |
| `opts.color` | string | `COLORS.accent` | Border and value color |

```js
addMetricCard(slide, pres, {
  x: 0.4, y: 0.75, w: 2.9, h: 1.9,
  value: "CPU > 70%", label: "Sustained 5+ min",
  sub: "Not an occasional spike", color: COLORS.danger,
});
```

---

### `addCodeCard(slide, pres, opts)`

Dark-background code snippet card with optional language label.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `opts.x` | number | — | X position (required) |
| `opts.y` | number | — | Y position (required) |
| `opts.w` | number | — | Width (required) |
| `opts.h` | number | — | Height (required) |
| `opts.code` | string | `""` | Multi-line code string |
| `opts.language` | string | `""` | Language label (e.g. `"Dockerfile"`, `"YAML"`) |

**Note:** Language label renders at `y - 0.16` (above the card).

```js
addCodeCard(slide, pres, {
  x: 5.3, y: 1.05, w: 2.0, h: 1.8,
  language: "Dockerfile",
  code: "FROM python:3.11-slim\nCOPY . /app\nRUN pip install -r req.txt\nCMD [\"uvicorn\", \"main:app\"]",
});
```

---

### `addKnowledgeCards(slide, pres, cards, opts)`

Row of small info cards at the bottom of a slide.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `cards` | {icon, title, text}[] | — | Array of card objects |
| `opts.y` | number | — | Y position |

```js
addKnowledgeCards(slide, pres, [
  { icon: "📘", title: "RFC 7230", text: "HTTP/1.1 message syntax" },
  { icon: "📗", title: "CAP Theorem", text: "Consistency vs Availability" },
], { y: 4.8 });
```

---

## Layout Helpers

### `addThreeCols(slide, pres, cols, opts)`

Three equal columns with icon headers and bullet items.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `cols` | array | — | 3 column objects (required) |
| `cols[n].title` | string | — | Column header text |
| `cols[n].icon` | string | `""` | Header emoji |
| `cols[n].color` | string | `COLORS.accent` | Header text color |
| `cols[n].items` | {text, sub?}[] | `[]` | Bullet items |
| `opts.y` | number | `HEADER_H + 0.1` | Start Y |
| `opts.h` | number | `H - HEADER_H - 0.2` | Column height |

**Column positions:** `x = 0.2, 3.43, 6.66` (each `w ≈ 3.13"`)

```js
addThreeCols(slide, pres, [
  { title: "① CDN", icon: "☁️", color: COLORS.cdn, items: [{ text: "Static assets" }] },
  { title: "② Redis", icon: "⚡", color: COLORS.infra, items: [{ text: "API caching" }] },
  { title: "③ Local", icon: "🧠", color: COLORS.warning, items: [{ text: "Hot config" }] },
], { y: 0.62, h: 4.38 });
```

---

## Icon Helper (`src/icon-helper.js`)

### `iconToBase64(IconComponent, color, size)`

Renders a `react-icons` component to a PNG base64 data URI.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `IconComponent` | React component | — | e.g. `FaServer` from `react-icons/fa` |
| `color` | string | — | Color WITH `#` prefix: `"#4A7FB5"` |
| `size` | number | `256` | Output PNG size in pixels |

**Returns:** `Promise<string>` — `"image/png;base64,..."` data URI

**⚠️ Must use `await`** — this is async!

```js
const { FaServer } = require("react-icons/fa");
const { iconToBase64 } = require("./icon-helper");

const icon = await iconToBase64(FaServer, "#" + COLORS.backend, 256);
slide.addImage({ data: icon, x: 1, y: 1, w: 0.5, h: 0.5 });
```
