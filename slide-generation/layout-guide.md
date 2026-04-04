# Layout Guide

> Spatial rules for the 10" × 5.5" pptxgenjs widescreen canvas. All values in inches.

## Canvas Dimensions

```
┌──────────────────────────────────────────────────────────────────────────┐
│  10.0" wide                                                             │
│                                                                          │
│  5.5" tall                                                               │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

- `pres.defineLayout({ name: "WIDE", width: 10, height: 5.5 })`
- Coordinates: `x=0` is left edge, `y=0` is top edge
- All position/size values are **inches as floats**

## Spatial Zones

Every slide follows this vertical zone system:

```
┌──────────────────────────────────────────────────────────────┐
│  HEADER ZONE   (y: 0 → 0.75)                                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ Slide title  |  Subtitle / description    | PART N    │   │
│  └────────────────────────────────────────────────────────┘   │
├──────────────────────────────────────────────────────────────┤
│  CONTENT ZONE  (y: 0.75 → 3.5)                              │
│                                                              │
│    Architecture diagrams, comparison columns, code cards,    │
│    concept illustrations, node cards + arrows                │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  BOTTOM ZONE   (y: 3.5 → 5.5)                               │
│                                                              │
│    Bottom panel (tip bars, alert bars, pros/cons cards,      │
│    summary cards, knowledge cards, metric cards)             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Zone Y-Coordinates

| Zone | Y Start | Y End | Height | Purpose |
|------|---------|-------|--------|---------|
| Header | `0.0` | `0.75` | `0.75"` | Title, subtitle, part label |
| Content | `0.75` | `3.5` | `2.75"` | Main visual content |
| Bottom | `3.5` | `5.5` | `2.0"` | Supporting info, cards |

### Horizontal Margins

| Dimension | Value | Notes |
|-----------|-------|-------|
| Left margin | `0.5"` | Standard content start |
| Right margin | `0.5"` | Content should not exceed `x + w = 9.5` |
| Usable width | `9.0"` | `10.0 - 0.5 - 0.5` |
| Card gap | `0.15"` - `0.3"` | Between adjacent cards |
| Column gap | `0.2"` | Between comparison columns |

## Header Zone Detail

`addSlideHeader(slide, pres, opts)` creates the full header:

```
┌──────────────────────────────────────────────────────────────┐
│  0.5"  Title (16pt bold)          Part Label ──→  │  0.15"  │
│        Subtitle (8.5pt muted)                     │         │
│  ─────────────────────── divider line (y≈0.65) ──────────── │
└──────────────────────────────────────────────────────────────┘
```

- Title: `x=0.5, y=0.18, fontSize=16`
- Subtitle: `x=0.5, y=0.42, fontSize=8.5`
- Part label: `x=8.8, y=0.2, fontSize=8, align=right`
- Divider line: `y=0.65, x=0.5 → 9.5`

## Content Zone Patterns

### Architecture Diagram (most common)

```
y=0.85  ┌─────────┐          ┌─────────┐          ┌─────────┐
        │  Node   │──arrow──→│  Node   │──arrow──→│  Node   │
        │  Card   │          │  Card   │          │  Card   │
y=1.85  └─────────┘          └─────────┘          └─────────┘
```

- Node cards: `w=1.5~2.2, h=0.8~1.1`
- Horizontal spacing between nodes: `0.3"~0.6"` gap for arrows
- Arrow labels sit centered between nodes
- **3-node row** example: `x=0.5, x=3.8, x=7.0` (with arrows between)
- **4-node row** example: `x=0.3, x=2.7, x=5.1, x=7.5`

### Two-Column Comparison

```
        ┌────── 4.25" ──────┐  0.2"  ┌────── 4.25" ──────┐
        │  ❌ Before/Bad     │       │  ✅ After/Good      │
y=0.85  │  heading           │       │  heading            │
        │  ┌─ item ─────┐   │       │  ┌─ item ─────┐    │
        │  ├─ item ─────┤   │       │  ├─ item ─────┤    │
        │  └─ item ─────┘   │       │  └─ item ─────┘    │
        └────────────────────┘       └─────────────────────┘
```

- Left column: `x=0.5, w=4.25`
- Right column: `x=5.0, w=4.25` (gap = `0.25"`)
- Use `addCompareHeading` at `y=0.85`, then `addCompareItem` rows below

### Three-Column Layout

```
        ┌──── 2.8" ────┐  ┌──── 2.8" ────┐  ┌──── 2.8" ────┐
y=0.85  │  Column 1     │  │  Column 2     │  │  Column 3     │
        │  (items)      │  │  (items)      │  │  (items)      │
        └───────────────┘  └───────────────┘  └───────────────┘
```

- `addThreeCols` handles positioning automatically
- Each column: `w ≈ 2.8"`, gaps between columns `≈ 0.3"`

## Bottom Zone Patterns

### Full-Width Bottom Panel

`addBottomPanel(slide, pres, { y })` — draws a filled panel from y to bottom:

```
y=3.5   ══════════════════════════════════════════════════
        ┌──── card ────┐  ┌──── card ────┐  ┌──── card ────┐
        │  ✅ Pros     │  │  ⚠️ Cons     │  │  💡 Tip      │
        └──────────────┘  └──────────────┘  └──────────────┘
y=5.5   ══════════════════════════════════════════════════
```

- Default `y=3.3`
- Panel fills `x=0` to `x=10`, `y` to `5.5`
- Place cards inside the panel at `y + 0.2` or so

### Tip Bar / Alert Bar

- Tip: `addTipBar(slide, pres, text, y)` — blue accent left border
- Alert: `addAlertBar(slide, pres, text, y)` — red danger left border
- Width: `9.0"` (0.5 to 9.5), height: `≈0.5"`

### Summary Card Grid (2×3)

```
        ┌─ card ──┐  ┌─ card ──┐  ┌─ card ──┐
y=3.6   │  1      │  │  2      │  │  3      │
        └─────────┘  └─────────┘  └─────────┘
        ┌─ card ──┐  ┌─ card ──┐  ┌─ card ──┐
y=4.55  │  4      │  │  5      │  │  6      │
        └─────────┘  └─────────┘  └─────────┘
```

- Each card: `w≈2.8, h≈0.8`
- Grid starts at `x=0.5`
- Row gap: `≈0.15"`

### Metric Cards (horizontal row)

```
        ┌── metric ──┐  ┌── metric ──┐  ┌── metric ──┐  ┌── metric ──┐
y=3.6   │  99.9%     │  │  < 200ms   │  │  50x       │  │  ∞         │
        │  Uptime    │  │  Latency   │  │  Throughput │  │  Scale     │
        └────────────┘  └────────────┘  └────────────┘  └────────────┘
```

- Each: `w≈2.1, h≈1.2`
- 4 cards in a row from `x=0.5`

## Content Density Rules

### Maximums Per Slide

| Element | Maximum | Rationale |
|---------|---------|-----------|
| Architecture nodes | 6 | More = too crowded, arrows overlap |
| Arrow labels | 5 | Each needs readable space |
| Comparison items per column | 6 | Below the fold becomes unreadable |
| Bullet points | 5 | Beyond 5, font size drops too small |
| Summary cards | 6 (2×3) | Standard grid fits comfortably |
| Metric cards | 4 | Single row, each needs big number space |
| Code lines in code card | 12 | More → shrink font → unreadable |
| Nested zone borders | 2 | More = visual noise |

### Minimum Sizes (readability floor)

| Element | Min Size | Notes |
|---------|----------|-------|
| Body text | 9pt | Smaller is illegible on projector |
| Node card title | 10pt | Must be readable from back row |
| Arrow label | 7pt | Smallest readable element |
| Code text | 8.5pt | Monospace needs more space |
| Card/node minimum width | 1.3" | Below this, text overflows |
| Card/node minimum height | 0.5" | Below this, padding is crushed |

### Spacing Rules

| Between | Minimum Gap | Recommended |
|---------|-------------|-------------|
| Adjacent node cards | 0.3" | 0.5" (for arrow) |
| Stacked rows of cards | 0.15" | 0.2" |
| Text and card edge | 0.1" (padding) | 0.15" |
| Bottom panel top and first card | 0.15" | 0.25" |

## Fill-the-Slide Checklist

Before committing any slide, verify:

- [ ] No white gap > 0.5" between content elements
- [ ] Content spans at least 80% of usable width (8" of 10")
- [ ] Bottom zone is utilized (panel, cards, tip bar, or metrics)
- [ ] Header zone has both title AND subtitle
- [ ] Part label is present (top-right)
- [ ] No element extends beyond `x=9.5` or `y=5.5`
- [ ] No element starts before `x=0.3` or `y=0.1`
