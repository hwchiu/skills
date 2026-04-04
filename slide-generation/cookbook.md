# Cookbook — Complete Slide Recipes

> Copy-paste-ready `buildSlide` functions for every slide pattern. Each recipe is a complete, working function you can drop into any part file.

## Prerequisites

Every part file starts with:

```js
const pptxgen = require("pptxgenjs");
const { COLORS, FONTS, setTheme } = require("./design-system");
setTheme("light");
const {
  initSlide, addSlideHeader, addBottomPanel,
  addNodeCard, addMiniNode, addHArrow, addDashedHArrow, addVArrow,
  addZoneBorder, addTipBar, addAlertBar, addCommentBar,
  addKnowledgeCards, addCompareHeading, addCompareItem,
  addSummaryCard, addMetricCard, addThreeCols, addCodeCard,
} = require("./helpers");
```

---

## Pattern 1: Cover Page

A split-layout title slide with large title on the left and a stacked card list on the right.

**When to use:** First slide of the entire deck.

**Layout:**
```
┌──────────────────────────┬───────────────────────────────┐
│  Big Title (72pt)        │  ┌── Journey Card 1 ──────┐  │
│  "CLOUD"                 │  └────────────────────────-┘  │
│  "NATIVE" (accent)       │  ┌── Journey Card 2 ──────┐  │
│                          │  └────────────────────────-┘  │
│  Subtitle (26pt)         │  ┌── Journey Card 3 ──────┐  │
│  Description (13pt)      │  └────────────────────────-┘  │
│                     ║    │  ┌── Journey Card 4 ──────┐  │
│              divider║    │  └────────────────────────-┘  │
└──────────────────────────┴───────────────────────────────┘
```

```js
function buildCoverSlide(pres) {
  const slide = initSlide(pres);

  // ── Left half: Title block ─────────────────────────────
  slide.addText("CLOUD", {
    x: 0.5, y: 1.1, w: 4.2, h: 0.9,
    fontSize: 72, bold: true, color: COLORS.text, fontFace: FONTS.title,
    charSpacing: -1,
  });
  slide.addText("NATIVE", {
    x: 0.5, y: 1.9, w: 4.2, h: 0.9,
    fontSize: 72, bold: true, color: COLORS.accent, fontFace: FONTS.title,
    charSpacing: -1,
  });
  slide.addText("System Deployment in Practice", {
    x: 0.5, y: 3.0, w: 4.2, h: 0.55,
    fontSize: 26, bold: true, color: COLORS.text, fontFace: FONTS.title,
  });
  slide.addText("The complete evolution from monolithic to cloud native", {
    x: 0.5, y: 3.55, w: 4.2, h: 0.35,
    fontSize: 13, color: COLORS.textMuted, fontFace: FONTS.body,
  });

  // Vertical divider
  slide.addShape(pres.ShapeType.line, {
    x: 5.0, y: 0.3, w: 0.01, h: 4.9,
    line: { color: COLORS.border, width: 0.75 },
  });

  // ── Right half: Journey cards ──────────────────────────
  const cards = [
    { border: COLORS.backend,   num: "1", title: "Traditional Deployment",
      sub: "Single Server → Three-Tier", chip: "Low", chipColor: COLORS.success },
    { border: COLORS.infra,     num: "2", title: "Scale Out Challenges",
      sub: "LB / Session / DB Scaling",  chip: "High 🔺", chipColor: COLORS.danger },
    { border: COLORS.container, num: "3", title: "Container Revolution",
      sub: "Docker · Compose · Registry", chip: "Medium", chipColor: COLORS.container },
    { border: COLORS.accent,    num: "4", title: "12-Factor + DevOps",
      sub: "CI/CD · Observability",       chip: "Stable", chipColor: COLORS.accent },
  ];

  let cy = 0.65;
  cards.forEach((c) => {
    slide.addShape(pres.ShapeType.roundRect, {
      x: 5.1, y: cy, w: 4.7, h: 0.95, rectRadius: 0.1,
      fill: { color: COLORS.bg2 },
      line: { color: c.border, width: 1.0 },
    });
    // Numbered circle
    slide.addShape(pres.ShapeType.ellipse, {
      x: 5.25, y: cy + 0.35, w: 0.24, h: 0.24,
      fill: { color: c.border }, line: { color: c.border, width: 0 },
    });
    slide.addText(c.num, {
      x: 5.25, y: cy + 0.35, w: 0.24, h: 0.24,
      fontSize: 9, bold: true, color: "FFFFFF",
      align: "center", valign: "middle", fontFace: FONTS.body,
    });
    // Title + subtitle
    slide.addText(c.title, {
      x: 5.57, y: cy + 0.1, w: 3.1, h: 0.3,
      fontSize: 12, bold: true, color: COLORS.text, fontFace: FONTS.body,
    });
    slide.addText(c.sub, {
      x: 5.57, y: cy + 0.42, w: 3.1, h: 0.22,
      fontSize: 8.5, color: COLORS.textMuted, fontFace: FONTS.code,
    });
    // Chip badge
    slide.addShape(pres.ShapeType.roundRect, {
      x: 8.7, y: cy + 0.34, w: 1.0, h: 0.24, rectRadius: 0.06,
      fill: { color: COLORS.bg3 },
      line: { color: c.chipColor, width: 0.75 },
    });
    slide.addText(c.chip, {
      x: 8.7, y: cy + 0.34, w: 1.0, h: 0.24,
      fontSize: 8, color: c.chipColor, fontFace: FONTS.body,
      align: "center", valign: "middle",
    });
    cy += 1.05;
  });
}
```

---

## Pattern 2: Section Opener

An agenda/outline slide with a large transparent section number and a full-width list of items.

**When to use:** Start of each major section/part.

**Layout:**
```
┌──────────────────────────────────────────────────────────────┐
│ 00 (transparent)   Section Title (30pt)                      │
│                    Subtitle                                  │
├──────────────────────────────────────────────────────────────┤
│  ▌1  PART 1   Title                            Description  │
│  ▌2  PART 2   Title                            Description  │
│  ▌3  PART 3   Title                            Description  │
│  ...                                                         │
└──────────────────────────────────────────────────────────────┘
```

```js
function buildSectionOpener(pres) {
  const slide = initSlide(pres);

  // Large transparent section number
  slide.addText("00", {
    x: 0.1, y: 0.2, w: 1.8, h: 1.0,
    fontSize: 110, bold: true, color: COLORS.accent,
    fontFace: FONTS.title, transparency: 75,
  });
  slide.addText("Course Outline", {
    x: 1.0, y: 0.28, w: 5.0, h: 0.55,
    fontSize: 30, bold: true, color: COLORS.accent, fontFace: FONTS.title,
  });
  slide.addText("Agenda — The Complete Evolution Path", {
    x: 1.0, y: 0.75, w: 5.0, h: 0.3,
    fontSize: 13, color: COLORS.textMuted, fontFace: FONTS.body,
  });

  const rows = [
    { color: COLORS.backend,   num: "1", part: "PART 1", title: "Traditional Deployment",  sub: "Single Server → Three-Tier" },
    { color: COLORS.infra,     num: "2", part: "PART 2", title: "Scale Out Challenges",     sub: "LB / Session / DB Scaling" },
    { color: COLORS.container, num: "3", part: "PART 3", title: "Container Revolution",     sub: "Docker / Compose / Registry" },
    // ... add more rows as needed
  ];

  let ry = 1.38;
  rows.forEach((r) => {
    // Row background
    slide.addShape(pres.ShapeType.roundRect, {
      x: 0.4, y: ry, w: 9.2, h: 0.58, rectRadius: 0.08,
      fill: { color: COLORS.bg2 },
      line: { color: COLORS.border, width: 0.5 },
    });
    // Left color accent strip
    slide.addShape(pres.ShapeType.rect, {
      x: 0.4, y: ry, w: 0.1, h: 0.58,
      fill: { color: r.color },
    });
    // Numbered circle
    slide.addShape(pres.ShapeType.ellipse, {
      x: 0.62, y: ry + 0.17, w: 0.24, h: 0.24,
      fill: { color: r.color }, line: { color: r.color, width: 0 },
    });
    slide.addText(r.num, {
      x: 0.62, y: ry + 0.17, w: 0.24, h: 0.24,
      fontSize: 9, bold: true, color: "FFFFFF",
      align: "center", valign: "middle", fontFace: FONTS.body,
    });
    slide.addText(r.part, {
      x: 1.0, y: ry + 0.06, w: 1.0, h: 0.25,
      fontSize: 9, bold: true, color: r.color, fontFace: FONTS.body,
    });
    slide.addText(r.title, {
      x: 1.0, y: ry + 0.28, w: 4.5, h: 0.25,
      fontSize: 12, bold: true, color: COLORS.text, fontFace: FONTS.body,
    });
    slide.addText(r.sub, {
      x: 6.2, y: ry + 0.17, w: 3.2, h: 0.25,
      fontSize: 10, color: COLORS.textMuted, fontFace: FONTS.body,
      align: "right", valign: "middle",
    });
    ry += 0.65;
  });
}
```

---

## Pattern 3: Architecture Diagram + Bottom Panel

The most common slide pattern. Shows a data flow of node cards connected by arrows, with a pros/cons or tip panel at the bottom.

**When to use:** Explaining system architecture, data flow, request lifecycle, deployment topology.

**Layout:**
```
┌──────────────────────────────────────────────────────────────┐
│  Header: "Starting Point: The Simplest Deployment"  PART 1  │
│  ─────────────────────────── divider ─────────────────────── │
├──────────────────────────────────────────────────────────────┤
│         ┌─────────────────────── Zone Border ────────────┐   │
│  👤     │  🌐           ⚙️           🗄️                  │   │
│ Client──│→ Frontend ──→ Backend ──→ Database             │   │
│         │  Nginx        FastAPI      Postgres             │   │
│         └────────────────────────────────────────────────-┘   │
│══════════════════════════════════════════════════════════════│
│  ✅ Advantages          │  ⚠️ Limitations                    │
│  • Simple deployment    │  • Single Point of Failure         │
│  • Easy debugging       │  • Cannot scale independently     │
└──────────────────────────────────────────────────────────────┘
```

```js
function buildArchitectureSlide(pres) {
  const slide = initSlide(pres);
  addSlideHeader(slide, pres, {
    title: "Starting Point: The Simplest Deployment",
    partLabel: "PART 1",
    accentColor: COLORS.backend,
    complexity: 1,
  });

  // Zone label
  slide.addText("SINGLE HOST DEPLOYMENT", {
    x: 0.3, y: 0.62, w: 3.0, h: 0.2,
    fontSize: 8, color: COLORS.textMuted, fontFace: FONTS.body, charSpacing: 1,
  });

  // Client node (outside zone)
  addNodeCard(slide, pres, {
    x: 0.2, y: 1.05, w: 1.3, h: 1.2,
    emoji: "👤", name: "Client", meta: "Browser",
    borderColor: COLORS.client,
  });

  // Arrow: client → zone
  addHArrow(slide, pres, { x: 1.57, y: 1.55, w: 0.5, label: "HTTP/80", color: COLORS.accent });

  // Zone border
  addZoneBorder(slide, pres, {
    x: 2.15, y: 0.75, w: 7.65, h: 2.6,
    color: COLORS.backend, label: "ubuntu-01",
  });

  // Nodes inside zone
  addNodeCard(slide, pres, {
    x: 2.4, y: 0.98, w: 2.0, h: 1.8,
    emoji: "🌐", name: "Frontend", meta: "Nginx :80",
    borderColor: COLORS.frontend,
  });
  addHArrow(slide, pres, { x: 4.47, y: 1.75, w: 0.56, label: "proxy", color: COLORS.frontend });

  addNodeCard(slide, pres, {
    x: 5.1, y: 0.98, w: 2.0, h: 1.8,
    emoji: "⚙️", name: "Backend", meta: "FastAPI :8080",
    borderColor: COLORS.backend,
  });
  addHArrow(slide, pres, { x: 7.17, y: 1.75, w: 0.56, label: "SQL", color: COLORS.database });

  addNodeCard(slide, pres, {
    x: 7.8, y: 0.98, w: 1.8, h: 1.8,
    emoji: "🗄️", name: "Database", meta: "Postgres :5432",
    borderColor: COLORS.database,
  });

  // Bottom panel with pros/cons
  addBottomPanel(slide, pres, [
    "Simple deployment — one machine handles everything",
    "All services colocated, easy debugging",
    "Perfect for a one-person team",
    "Upgrade machine specs when performance is insufficient",
  ], [
    "Single Point of Failure (SPOF)",
    "Any component failure takes down everything",
    "No component can be scaled independently",
  ], { y: 3.3, h: 2.2 });
}
```

---

## Pattern 4: Comparison Slide (Two-Column)

Side-by-side comparison with a vertical divider. Left = "before/bad/old", Right = "after/good/new".

**When to use:** Before vs. After, Pros vs. Cons, Scale Up vs. Scale Out, any A vs. B comparison.

**Layout:**
```
┌──────────────────────────┬───────────────────────────────┐
│  Header                  │                     PART N    │
├──────────────────────────┼───────────────────────────────┤
│  ❌ Before / Scale Up    │  ✅ After / Scale Out         │
│  [visual: growing boxes] │  [visual: LB + servers]       │
│  ✓ No code changes       │  ✓ Linear cost growth         │
│  ✗ Exponential cost      │  ✓ Scale without downtime     │
│  ✗ Physical limits       │  ! Must be stateless          │
└──────────────────────────┴───────────────────────────────┘
```

```js
function buildComparisonSlide(pres) {
  const slide = initSlide(pres);
  addSlideHeader(slide, pres, {
    title: "Scale Up vs Scale Out: Two Scaling Strategies",
    partLabel: "PART 1",
    accentColor: COLORS.accent,
  });

  // Vertical divider
  slide.addShape(pres.ShapeType.line, {
    x: 5.0, y: 0.55, w: 0.01, h: 4.85,
    line: { color: COLORS.border, width: 0.75 },
  });

  // ── Left column ────────────────────────────────────────
  addCompareHeading(slide, pres, {
    x: 0.3, y: 0.62, w: 4.4,
    label: "↑  Scale Up (Vertical Scaling)", type: "bad",
  });
  // Visual elements showing the concept
  addNodeCard(slide, pres, { x: 0.5, y: 1.15, w: 0.9, h: 0.9, emoji: "⚙️", meta: "2 vCPU", borderColor: COLORS.textMuted });
  slide.addText("→", { x: 1.45, y: 1.45, w: 0.3, h: 0.3, fontSize: 16, bold: true, color: COLORS.accent, fontFace: FONTS.body, align: "center" });
  addNodeCard(slide, pres, { x: 1.8, y: 1.05, w: 1.1, h: 1.1, emoji: "⚙️", meta: "16 vCPU", borderColor: COLORS.backend });
  // Comparison items
  addCompareItem(slide, pres, { x: 0.3, y: 2.48, w: 4.4, emoji: "✓", title: "No Code Changes", sub: "Simply upgrade machine specs", type: "good" });
  addCompareItem(slide, pres, { x: 0.3, y: 3.14, w: 4.4, emoji: "✗", title: "Exponential Cost", sub: "High-end machines are expensive", type: "bad" });
  addCompareItem(slide, pres, { x: 0.3, y: 3.82, w: 4.4, emoji: "✗", title: "Physical Limits", sub: "Has a ceiling, requires downtime", type: "bad" });

  // ── Right column ───────────────────────────────────────
  addCompareHeading(slide, pres, {
    x: 5.2, y: 0.62, w: 4.4,
    label: "↔  Scale Out (Horizontal Scaling)", type: "good",
  });
  // Visual: LB → 3 servers
  slide.addShape(pres.ShapeType.roundRect, {
    x: 5.4, y: 1.1, w: 0.9, h: 0.9, rectRadius: 0.1,
    fill: { color: COLORS.bg2 }, line: { color: COLORS.infra, width: 1.2 },
  });
  slide.addText("⚖️\nLB", {
    x: 5.4, y: 1.1, w: 0.9, h: 0.9,
    fontSize: 13, align: "center", valign: "middle", color: COLORS.infra, fontFace: FONTS.body,
  });
  [0.9, 1.5, 2.1].forEach((sy) => {
    addMiniNode(slide, pres, { x: 6.8, y: sy, w: 1.1, h: 0.45, emoji: "⚙️", label: "Server", borderColor: COLORS.backend });
    slide.addShape(pres.ShapeType.line, {
      x: 6.3, y: sy + 0.22, w: 0.5, h: 0.01,
      line: { color: COLORS.infra, width: 1.0, endArrowType: "arrow" },
    });
  });
  // Comparison items
  addCompareItem(slide, pres, { x: 5.2, y: 2.48, w: 4.4, emoji: "✓", title: "Linear Cost Growth", sub: "Fleet of small machines", type: "good" });
  addCompareItem(slide, pres, { x: 5.2, y: 3.14, w: 4.4, emoji: "✓", title: "Scale Without Downtime", sub: "Add/remove nodes dynamically", type: "good" });
  addCompareItem(slide, pres, { x: 5.2, y: 3.82, w: 4.4, emoji: "!", title: "Must Be Stateless", sub: "Session/cache/files need redesign", type: "warning" });
}
```

---

## Pattern 5: Code Walkthrough

Split view with a code card on one side and explanatory visual elements on the other.

**When to use:** Showing Dockerfiles, YAML configs, code snippets alongside diagrams.

**Layout:**
```
┌──────────────────────────┬───────────────────────────────┐
│  ❌ Before               │  ✅ After                     │
│  ┌─ Problem Card 1 ──┐  │  ┌────── code ────────┐       │
│  ├─ Problem Card 2 ──┤  │  │ FROM python:3.11   │──→ 📦 │
│  ├─ Problem Card 3 ──┤  │  │ COPY . /app        │ Image │
│  └────────────────────┘  │  └────────────────────┘       │
│  🔥 Callout              │  ✓ Runs on any host           │
└──────────────────────────┴───────────────────────────────┘
```

```js
function buildCodeSlide(pres) {
  const slide = initSlide(pres);
  addSlideHeader(slide, pres, {
    title: "What Is a Container? What Problems Does It Solve?",
    partLabel: "PART 3",
    accentColor: COLORS.container,
    complexity: 5,
  });

  // Vertical divider
  slide.addShape(pres.ShapeType.line, {
    x: 5.0, y: 0.55, w: 0.01, h: 4.85,
    line: { color: COLORS.border, width: 0.75 },
  });

  // Left column: Problems
  addCompareHeading(slide, pres, { x: 0.3, y: 0.62, w: 4.4, label: "❌  Before Containers", type: "bad" });

  // Problem cards (manually styled)
  const problems = [
    { env: "Dev (Mac)", ver: "Python 3.9 + brew", note: "Manual install", y: 1.1, fill: COLORS.cardDanger, line: COLORS.danger },
    { env: "Staging (Ubuntu)", ver: "Python 3.8 + apt", note: "Version mismatch", y: 1.98, fill: COLORS.cardWarn, line: COLORS.warning },
    { env: "Prod (CentOS)", ver: "Python 3.6 + yum", note: "Cannot upgrade", y: 2.86, fill: COLORS.cardDanger, line: COLORS.danger },
  ];
  problems.forEach((p) => {
    slide.addShape(pres.ShapeType.roundRect, {
      x: 0.4, y: p.y, w: 4.1, h: 0.78, rectRadius: 0.08,
      fill: { color: p.fill }, line: { color: p.line, width: 1.0 },
    });
    slide.addText(`🖥️ ${p.env}  —  ${p.ver}`, {
      x: 0.55, y: p.y + 0.04, w: 3.8, h: 0.28,
      fontSize: 11, bold: true, color: COLORS.text, fontFace: FONTS.body,
    });
    slide.addText(p.note, {
      x: 0.55, y: p.y + 0.32, w: 3.8, h: 0.22,
      fontSize: 9, color: COLORS.textMuted, fontFace: FONTS.body,
    });
  });

  // Right column: Solution with code
  addCompareHeading(slide, pres, { x: 5.2, y: 0.62, w: 4.4, label: "✅  With Containers", type: "good" });
  addCodeCard(slide, pres, {
    x: 5.3, y: 1.05, w: 2.0, h: 1.8,
    language: "Dockerfile",
    code: "FROM python:3.11-slim\nCOPY . /app\nRUN pip install -r req.txt\nCMD [\"uvicorn\", \"main:app\"]",
  });
  addHArrow(slide, pres, { x: 7.4, y: 1.63, label: "build", color: COLORS.container, w: 0.5 });
  addNodeCard(slide, pres, {
    x: 8.0, y: 1.05, w: 1.55, h: 1.8,
    emoji: "📦", name: "Image", meta: "myapp:v1.2.3",
    borderColor: COLORS.container,
  });
}
```

---

## Pattern 6: Summary Grid (2×2)

End-of-section recap with a grid of summary cards, each representing a key concept covered.

**When to use:** Last slide of each part/section.

**Layout:**
```
┌──────────────────────────────────────────────────────────────┐
│  Header: "Part N Summary"                          PART N   │
├──────────────────────────┬───────────────────────────────────┤
│  🖥️ Concept 1            │  🗄️ Concept 2                    │
│  • Point A               │  • Point A                       │
│  • Point B               │  • Point B                       │
├──────────────────────────┼───────────────────────────────────┤
│  🌐 Concept 3            │  ⚖️ Concept 4                    │
│  • Point A               │  • Point A                       │
│  • Point B               │  • Point B → Next Part           │
├──────────────────────────┴───────────────────────────────────┤
│         → Part N+1: What Comes Next                          │
└──────────────────────────────────────────────────────────────┘
```

```js
function buildSummarySlide(pres) {
  const slide = initSlide(pres);
  addSlideHeader(slide, pres, {
    title: "Part 1 Summary: Architecture Evolution Roadmap",
    partLabel: "PART 1",
    accentColor: COLORS.backend,
  });

  // 2×2 summary card grid
  addSummaryCard(slide, pres, {
    x: 0.3, y: 0.65, w: 4.5, h: 2.25,
    icon: "🖥️", title: "Single-Server Deployment", color: COLORS.backend,
    items: ["Simple setup, fast to start", "SPOF — Single Point of Failure", "Cannot scale"],
  });
  addSummaryCard(slide, pres, {
    x: 5.2, y: 0.65, w: 4.5, h: 2.25,
    icon: "🗄️", title: "DB Separation", color: COLORS.database,
    items: ["Data separated from application", "Independent backup", "Still 2 SPOFs"],
  });
  addSummaryCard(slide, pres, {
    x: 0.3, y: 3.1, w: 4.5, h: 2.25,
    icon: "🌐", title: "Three-Tier Architecture", color: COLORS.frontend,
    items: ["Clear separation of concerns", "Flexible tech stack", "Complexity grows"],
  });
  addSummaryCard(slide, pres, {
    x: 5.2, y: 3.1, w: 4.5, h: 2.25,
    icon: "⚖️", title: "Scale Out Readiness", color: COLORS.infra,
    items: ["Requires Load Balancer", "App must be stateless", "Ready? → Part 2"],
  });

  // Bottom CTA
  slide.addText("→ Part 2: The Challenges of Scale Out", {
    x: 0, y: 5.28, w: 10, h: 0.2,
    fontSize: 11, color: COLORS.accent, fontFace: FONTS.body, bold: true, align: "center",
  });
}
```

---

## Pattern 7: Three-Column Layout

Three equal columns with titled sections and bullet items. Great for comparing strategies, listing steps, or categorizing information.

**When to use:** 3 parallel concepts, strategies, or categories.

```js
function buildThreeColumnSlide(pres) {
  const slide = initSlide(pres);
  addSlideHeader(slide, pres, {
    title: "Three-Tier Caching Strategy: Outside to Inside",
    partLabel: "PART 2",
    accentColor: COLORS.accent,
  });

  addThreeCols(slide, pres, [
    {
      title: "① CDN", icon: "☁️", color: COLORS.cdn,
      items: [
        { text: "Static assets (JS/CSS/images)", sub: "TTL: days/weeks" },
        { text: "Tools: Cloudflare, CloudFront" },
        { text: "Cache Hit → no backend contact" },
        { text: "Hit Rate: ~60-80%" },
      ],
    },
    {
      title: "② Redis Cache", icon: "⚡", color: COLORS.infra,
      items: [
        { text: "API response caching", sub: "TTL: seconds/minutes" },
        { text: "Session storage" },
        { text: "Tools: Redis, Memcached" },
        { text: "Hit Rate: ~30-50%" },
      ],
    },
    {
      title: "③ In-Process Cache", icon: "🧠", color: COLORS.warning,
      items: [
        { text: "Hot data (config/constants)", sub: "TTL: seconds" },
        { text: "Cannot be shared across servers" },
        { text: "Requires special handling for scale out" },
        { text: "Hit Rate: ~10-20%" },
      ],
    },
  ], { y: 0.62, h: 4.38 });

  addTipBar(slide, pres, {
    y: 5.0,
    text: "Cache invalidation is one of the hardest problems — design your TTL strategy carefully",
  });
}
```

---

## Pattern 8: Metrics / KPI Slide

Big numbers at the top with detailed action cards below. Draws attention to key thresholds.

**When to use:** Showing performance thresholds, SLO targets, capacity limits, before/after metrics.

**Layout:**
```
┌──────────────────────────────────────────────────────────────┐
│  Header                                            PART N   │
├──────────────────────────────────────────────────────────────┤
│  ┌─ CPU>70% ─┐  ┌─ P99>1s ──┐  ┌─ Err>0.1% ┐              │
│  │  (big #)  │  │  (big #)  │  │  (big #)   │              │
│  └───────────┘  └───────────┘  └────────────┘              │
│  ⚠️ Warning banner                                          │
│  ┌─── ✅ Do First ──────┐  ┌─── ⚠️ Then Scale ─────┐       │
│  │ • Add indexes        │  │ • Confirm bottleneck   │       │
│  │ • Intro Redis        │  │ • Stateless first      │       │
│  │ • Fix N+1 queries    │  │ • Need Load Balancer   │       │
│  └──────────────────────┘  └─────────────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

```js
function buildMetricsSlide(pres) {
  const slide = initSlide(pres);
  addSlideHeader(slide, pres, {
    title: "When Should You Start Thinking About Scaling?",
    partLabel: "PART 1",
    accentColor: COLORS.warning,
  });

  // Metric cards row
  addMetricCard(slide, pres, { x: 0.4, y: 0.75, w: 2.9, h: 1.9, value: "CPU > 70%", label: "Sustained 5+ min", sub: "Not an occasional spike", color: COLORS.danger });
  addMetricCard(slide, pres, { x: 3.5, y: 0.75, w: 2.9, h: 1.9, value: "P99 > 1s", label: "API Tail Latency", sub: "Users are noticing lag", color: COLORS.warning });
  addMetricCard(slide, pres, { x: 6.6, y: 0.75, w: 2.9, h: 1.9, value: "Err > 0.1%", label: "Error Rate Threshold", sub: "5xx or timeouts increasing", color: COLORS.danger });

  // Warning banner
  slide.addShape(pres.ShapeType.roundRect, {
    x: 0.3, y: 2.85, w: 9.4, h: 0.55, rectRadius: 0.07,
    fill: { color: COLORS.cardWarn },
    line: { color: COLORS.warning, width: 1.0 },
  });
  slide.addText("⚠️  Start with code optimization — don't add machines at the first sign of trouble", {
    x: 0.5, y: 2.89, w: 9.0, h: 0.47,
    fontSize: 11, color: COLORS.warning, fontFace: FONTS.body, valign: "middle",
  });

  // Two action cards
  slide.addShape(pres.ShapeType.roundRect, {
    x: 0.3, y: 3.55, w: 4.5, h: 1.75, rectRadius: 0.1,
    fill: { color: COLORS.cardSuccess },
    line: { color: COLORS.success, width: 1.0 },
  });
  slide.addText("✅ Do These First", {
    x: 0.5, y: 3.65, w: 4.1, h: 0.3,
    fontSize: 12, bold: true, color: COLORS.success, fontFace: FONTS.body,
  });
  ["• Add indexes", "• Introduce Redis caching", "• Optimize N+1 queries", "• CDN for static assets"].forEach((item, i) => {
    slide.addText(item, { x: 0.5, y: 4.02 + i * 0.3, w: 4.1, h: 0.27, fontSize: 11, color: COLORS.text, fontFace: FONTS.body });
  });

  slide.addShape(pres.ShapeType.roundRect, {
    x: 5.2, y: 3.55, w: 4.5, h: 1.75, rectRadius: 0.1,
    fill: { color: COLORS.cardDanger },
    line: { color: COLORS.danger, width: 1.0 },
  });
  slide.addText("⚠️ Then Consider Scale Out", {
    x: 5.4, y: 3.65, w: 4.1, h: 0.3,
    fontSize: 12, bold: true, color: COLORS.danger, fontFace: FONTS.body,
  });
  ["• Confirm bottleneck is capacity", "• Application must be stateless", "• Requires a Load Balancer"].forEach((item, i) => {
    slide.addText(item, { x: 5.4, y: 4.02 + i * 0.3, w: 4.1, h: 0.27, fontSize: 11, color: COLORS.text, fontFace: FONTS.body });
  });
}
```

---

## Combining Patterns

Most real slides combine elements from multiple patterns. Common combos:

| Combo | Example |
|-------|---------|
| Architecture + Tip | Diagram at top, `addTipBar` at bottom |
| Architecture + Knowledge Cards | Diagram at top, `addKnowledgeCards` at bottom |
| Code + Arrow + Node | `addCodeCard` → `addHArrow` → `addNodeCard` |
| Comparison + Code | Left: problem cards, Right: code solution |
| Metrics + Action Cards | Big numbers at top, recommendation cards below |
| Three-Column + Tip | `addThreeCols` for content, `addTipBar` at Y=5.0 |
