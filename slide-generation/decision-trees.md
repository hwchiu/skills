# Decision Trees

> Use these flowcharts to pick the right slide pattern and helpers for your content.

## What Slide Pattern Should I Use?

```
START: What content do I have?
│
├─ A title + course overview?
│   └─→ COVER SLIDE (see cookbook: Cover Page)
│
├─ A new section/part opener?
│   └─→ SECTION OPENER (see cookbook: Section Opener)
│
├─ System architecture or data flow?
│   ├─ How many components?
│   │   ├─ 2-3 components → Single-row architecture
│   │   ├─ 4-6 components → Multi-row or tiered architecture
│   │   └─ 7+ components → Split into 2 slides!
│   └─→ ARCHITECTURE DIAGRAM (see cookbook: Architecture Diagram)
│
├─ Before vs After, or Pros vs Cons?
│   └─→ COMPARISON SLIDE (see cookbook: Comparison Slide)
│
├─ A concept to explain with bullet points?
│   ├─ 3 parallel concepts? → THREE-COLUMN LAYOUT
│   └─ 1 concept, 3-5 bullets? → CONCEPT SLIDE
│   └─→ (see cookbook: Concept / Three-Column)
│
├─ Code snippet or config example?
│   └─→ CODE WALKTHROUGH (see cookbook: Code Walkthrough)
│
├─ Key metrics or KPIs?
│   └─→ METRICS SLIDE (see cookbook: Metrics/KPI)
│
└─ End-of-section recap?
    └─→ SUMMARY SLIDE (see cookbook: Summary Grid)
```

## What Helper Do I Use?

### For Boxes/Cards

```
Need a box?
│
├─ System component (server, DB, LB)?
│   └─→ addNodeCard(slide, pres, { ... })
│       Option: Use nameColor to color-match the component name
│
├─ Smaller component inside a zone?
│   └─→ addMiniNode(slide, pres, { ... })
│
├─ Summary/recap card?
│   └─→ addSummaryCard(slide, pres, { ... })
│       Use in 2×3 grid pattern
│
├─ Metric/KPI card with big number?
│   └─→ addMetricCard(slide, pres, { ... })
│       Use in horizontal row (max 4)
│
├─ Code snippet?
│   └─→ addCodeCard(slide, pres, { ... })
│       Dark background, monospace font
│
└─ Pros/Cons comparison item?
    └─→ addCompareItem(slide, pres, { ... })
        Pair with addCompareHeading
```

### For Connectors/Flow

```
Need to show flow?
│
├─ Left-to-right data flow?
│   └─→ addHArrow(slide, pres, { ... })
│       Pill-badge style with protocol/action label
│
├─ Response/return flow (dotted)?
│   └─→ addDashedHArrow(slide, pres, { ... })
│       Dashed line with response label
│
├─ Top-to-bottom flow?
│   └─→ addVArrow(slide, pres, { ... })
│       Vertical connector
│
└─ Grouping boundary (K8s namespace, VPC)?
    └─→ addZoneBorder(slide, pres, { ... })
        Dashed border with label
```

### For Information Bars

```
Need to highlight info?
│
├─ Helpful tip? 💡
│   └─→ addTipBar(slide, pres, text, y)
│       Blue accent border
│
├─ Warning or limitation? ⚠️
│   └─→ addAlertBar(slide, pres, text, y)
│       Red danger border
│
├─ Code-comment style note? //
│   └─→ addCommentBar(slide, pres, text, opts)
│       Grey italic, looks like a code comment
│
└─ Bottom knowledge cards?
    └─→ addKnowledgeCards(slide, pres, cards, opts)
        Row of small info cards at bottom
```

### For Slide Structure

```
Need slide scaffolding?
│
├─ New blank slide with background?
│   └─→ initSlide(pres)
│       Sets background color, returns slide object
│
├─ Header with title + subtitle + divider?
│   └─→ addSlideHeader(slide, pres, { title, subtitle, partLabel })
│
├─ Colored bottom panel?
│   └─→ addBottomPanel(slide, pres, { y })
│       Fills from y to bottom edge
│
├─ Two-column comparison header?
│   └─→ addCompareHeading(slide, pres, { leftText, rightText, y })
│
└─ Three-column layout?
    └─→ addThreeCols(slide, pres, columns, opts)
        Auto-positions 3 equal columns
```

## Quick Reference: Content → Pattern Map

| Content Type | Pattern | Key Helpers |
|-------------|---------|-------------|
| Course/deck title | Cover | initSlide, addText (manual) |
| New section intro | Section Opener | addSlideHeader, addText bullets |
| Data flow (3 nodes) | Architecture | addNodeCard × 3, addHArrow × 2, addBottomPanel |
| Before/After | Comparison | addCompareHeading, addCompareItem |
| Concept + bullets | Concept | addSlideHeader, addText bullets |
| Config/Code | Code Walkthrough | addCodeCard, addTipBar |
| Numbers/KPIs | Metrics | addMetricCard × 4, addBottomPanel |
| Section recap | Summary Grid | addSummaryCard × 6 (2×3 grid) |
| 3 parallel ideas | Three-Column | addThreeCols |
