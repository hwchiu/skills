# Slide Generation Skills

> A portable design system and style guide for building professional programmatic slide decks with **pptxgenjs**. Topic-agnostic — bring this to any project and reproduce the same polished "engineer handbook" aesthetic.

## Quick Start

```js
const pptxgen = require("pptxgenjs");
const { COLORS, FONTS, setTheme } = require("./design-system");
setTheme("light");
const { initSlide, addSlideHeader, addNodeCard, addHArrow, addTipBar } = require("./helpers");

async function main() {
  const pres = new pptxgen();
  const slide = initSlide(pres);
  addSlideHeader(slide, pres, { title: "My Slide", partLabel: "PART 1" });
  // ... add content ...
  await pres.writeFile({ fileName: "output/my-deck.pptx" });
}
main();
```

## Skill Files

| File | What It Covers |
|------|---------------|
| [setup.md](skills/setup.md) | **Start here** — prerequisites, installation, version matrix, troubleshooting |
| [design-tokens.md](skills/design-tokens.md) | Colors, fonts, theme system, component color mapping |
| [layout-guide.md](skills/layout-guide.md) | Canvas dimensions, spatial zones, content density rules |
| [helpers-api.md](skills/helpers-api.md) | Complete helper function API with all parameters and defaults |
| [cookbook.md](skills/cookbook.md) | Copy-paste recipes for every slide pattern |
| [decision-trees.md](skills/decision-trees.md) | Which helper/pattern to use for which content type |
| [anti-patterns.md](skills/anti-patterns.md) | Common mistakes and how to avoid them |
| [template-workflow.md](skills/template-workflow.md) | Working with .pptx master templates (optional) |

## Project Structure

```
src/
  design-system.js   # Color/font tokens + theme switching
  helpers.js         # Shared slide-building utilities
  <topic>.js         # Standalone script per part/topic
  merge.js           # Merges multiple .pptx into one
output/              # Generated .pptx files
templates/           # (Optional) .pptx master templates
skills/              # This design system documentation
```

## Build Commands

```bash
node src/<topic>.js          # Generate one part → output/<topic>.pptx
node src/merge.js            # Merge all parts → output/final.pptx
```

## Core Principles

1. **Never hardcode colors** — always use `COLORS.*` tokens
2. **Fill the slide** — no large empty areas; content should span the canvas
3. **Use native PPT text** — all text via `slide.addText()` for easy post-editing
4. **Projector-first** — warm light theme, high contrast, readable at distance
5. **Engineer handbook aesthetic** — pill-badge arrows, code-comment bars, color-matched nodes
