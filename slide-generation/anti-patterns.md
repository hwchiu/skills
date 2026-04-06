# Anti-Patterns

> Common mistakes when building slides with pptxgenjs + this design system. Each one includes the bad pattern, why it's bad, and the correct approach.

---

## 1. Hardcoded Colors

**❌ Bad:**
```js
slide.addText("Title", { color: "58A6FF", fontSize: 16 });
slide.addShape(pres.ShapeType.rect, { fill: { color: "0D1117" } });
```

**Why it's bad:** Breaks when you switch themes. Makes bulk updates impossible. Colors become inconsistent.

**✅ Good:**
```js
slide.addText("Title", { color: COLORS.accent, fontSize: 16 });
slide.addShape(pres.ShapeType.rect, { fill: { color: COLORS.bg } });
```

---

## 2. Forgetting `setTheme()` Call

**❌ Bad:**
```js
const { COLORS, FONTS } = require("./design-system");
// Missing setTheme("light") — COLORS still has dark theme values!
```

**Why it's bad:** The default export is the dark theme. Without `setTheme("light")`, your slides will have the wrong colors.

**✅ Good:**
```js
const { COLORS, FONTS, setTheme } = require("./design-system");
setTheme("light");
```

---

## 3. Text Overflow from Long Strings

**❌ Bad:**
```js
addNodeCard(slide, pres, {
  title: "Application Server with Load Balancing and Health Checks",
  w: 1.5, // too narrow for this text
});
```

**Why it's bad:** pptxgenjs doesn't auto-shrink text. The string overflows the card boundary. On projector, it looks broken.

**✅ Good:**
- Keep card titles ≤ 3 words for `w < 2.0`
- Use `shrinkText: true` on `addText` options if available
- Or increase card width: `w: 2.5`
- Or abbreviate: `"App Server"` instead of `"Application Server with Load Balancing and Health Checks"`

---

## 4. Element Overlap (Stacking Without Y-offset)

**❌ Bad:**
```js
addNodeCard(slide, pres, { x: 2, y: 1.0, h: 1.0, ... });
addNodeCard(slide, pres, { x: 2, y: 1.5, h: 1.0, ... });
// Second card overlaps first (1.0 + 1.0 = 2.0 > 1.5)
```

**Why it's bad:** Cards visually collide. Text becomes unreadable.

**✅ Good:**
```js
addNodeCard(slide, pres, { x: 2, y: 1.0, h: 1.0, ... });
addNodeCard(slide, pres, { x: 2, y: 2.2, h: 1.0, ... });
// Gap of 0.2" between cards (2.2 > 1.0 + 1.0)
```

**Rule:** Next element Y ≥ previous Y + previous H + 0.15"

---

## 5. Too Many Nodes on One Slide

**❌ Bad:** 8 node cards + 7 arrows on a single slide. Everything is tiny and cramped.

**Why it's bad:** Projector readability drops dramatically. Audience can't follow the flow.

**✅ Good:**
- Max 6 nodes per slide
- Max 5 arrows per slide
- If you have more, split into 2 slides: "Part A: Frontend → API" and "Part B: API → Database"
- Use a zone border to group related nodes if you must have 5-6

---

## 6. Missing Bottom Zone Content

**❌ Bad:** Slide has header + 3 node cards in the middle, but the bottom 2" is completely empty.

**Why it's bad:** Wastes 35% of slide area. Looks incomplete. Breaks the "fill the slide" principle.

**✅ Good:** Always fill the bottom zone with one of:
- `addBottomPanel` + pros/cons cards
- `addTipBar` with a key takeaway
- `addAlertBar` with a warning
- `addKnowledgeCards` with supporting facts
- `addCommentBar` with context
- Summary or metric cards

---

## 7. Inconsistent Component Colors

**❌ Bad:**
```js
// Slide 3: Database is orange
addNodeCard(slide, pres, { ..., borderColor: COLORS.database });
// Slide 7: Database is green (wrong!)
addNodeCard(slide, pres, { ..., borderColor: COLORS.backend });
```

**Why it's bad:** Viewers build mental associations. If "Database = orange" on slide 3 but "Database = green" on slide 7, they lose the visual thread.

**✅ Good:** Always use the same token for the same component type:
- Frontend → `COLORS.frontend` (always)
- Database → `COLORS.database` (always)
- See design-tokens.md for full component color map

---

## 8. Using `#` Prefix in pptxgenjs Colors

**❌ Bad:**
```js
slide.addText("Title", { color: "#4A7FB5" });
```

**Why it's bad:** pptxgenjs expects hex WITHOUT `#`. The `#` causes a silent failure or wrong color rendering.

**✅ Good:**
```js
slide.addText("Title", { color: "4A7FB5" });
// Or better:
slide.addText("Title", { color: COLORS.accent });
```

**Exception:** `iconToBase64()` DOES need `#` prefix: `iconToBase64(FaServer, "#" + COLORS.backend, 256)`

---

## 9. Arrows Without Labels

**❌ Bad:**
```js
addHArrow(slide, pres, { x1: 2.5, x2: 4.5, y: 1.5 });
// No label — what does this arrow represent?
```

**Why it's bad:** Unlabeled arrows are ambiguous. Viewers guess at the relationship.

**✅ Good:**
```js
addHArrow(slide, pres, { x1: 2.5, x2: 4.5, y: 1.5, label: "HTTP" });
```

---

## 10. Giant Text Blocks Instead of Visual Elements

**❌ Bad:** A slide with 15 lines of bullet-point text and no diagrams, cards, or visual structure.

**Why it's bad:** This is a Word document, not a slide. Audience reads faster than you speak and tunes out.

**✅ Good:**
- Max 5 bullet points per area
- Replace lists with node cards when showing components
- Replace process descriptions with arrow flows
- Use comparison columns instead of "advantages and disadvantages" paragraphs

---

## 11. Forgetting `pres` Parameter

**❌ Bad:**
```js
addNodeCard(slide, { title: "Server", ... });
// Missing pres — crash!
```

**Why it's bad:** Most helpers need `pres` for `pres.ShapeType.*` access. Without it, you get a runtime error.

**✅ Good:**
```js
addNodeCard(slide, pres, { title: "Server", ... });
```

**Rule:** The call signature is always `helperName(slide, pres, { options })`.

---

## 13. Negative Line Shape Dimensions (Triggers PowerPoint Repair)

**❌ Bad:**
```js
// Drawing a line from (x1,y1) to (x2,y2) — works only when x2>x1 and y2>y1
slide.addShape(pres.ShapeType.line, {
  x: x1, y: y1, w: x2 - x1, h: y2 - y1,  // ❌ negative if x2<x1 or y2<y1
  line: { color: COLORS.border, width: 1 },
});
```

**Why it's bad:** pptxgenjs writes `cx`/`cy` directly from `w`/`h`. A negative value produces `<a:ext cx="-2057400"/>` in the OOXML — an invalid dimension that causes PowerPoint to show a **"repair required"** dialog on open.

This is easiest to hit when drawing connection lines between diagram nodes in a loop:
```js
lines.forEach(a => {
  slide.addShape(pres.ShapeType.line, {
    x: a.x1, y: a.y1,
    w: a.x2 - a.x1,  // ❌ negative when line goes left (x2 < x1)
    h: a.y2 - a.y1,  // ❌ negative when line goes up   (y2 < y1)
  });
});
```

**✅ Good:** Always normalize to a positive bounding box + `flipH`/`flipV` for direction:
```js
lines.forEach(a => {
  const x = Math.min(a.x1, a.x2);
  const y = Math.min(a.y1, a.y2);
  const w = Math.max(Math.abs(a.x2 - a.x1), 0.01); // 0.01 avoids zero-width
  const h = Math.max(Math.abs(a.y2 - a.y1), 0.01);
  const flipH = a.x2 < a.x1;
  const flipV = a.y2 < a.y1;
  slide.addShape(pres.ShapeType.line, {
    x, y, w, h,
    line: { color: COLORS.border, width: 1 },
    ...(flipH ? { flipH: true } : {}),
    ...(flipV ? { flipV: true } : {}),
  });
});
```

**Rule:** Any time you compute `w` or `h` by subtracting two coordinates, guard with `Math.abs()`.

---

## 12. Not Awaiting `iconToBase64()`

**❌ Bad:**
```js
const icon = iconToBase64(FaServer, "#4A7FB5", 256);
slide.addImage({ data: icon, ... }); // icon is a Promise, not base64!
```

**Why it's bad:** `iconToBase64` is async. Without `await`, you pass a Promise object as image data.

**✅ Good:**
```js
const icon = await iconToBase64(FaServer, "#" + COLORS.backend, 256);
slide.addImage({ data: icon, x: 1, y: 1, w: 0.5, h: 0.5 });
```

---

## Summary Checklist

Before building each slide, verify:

- [ ] `setTheme()` called at top of file
- [ ] All colors use `COLORS.*` tokens
- [ ] No `#` prefix in pptxgenjs color values
- [ ] Components use correct semantic colors
- [ ] Max 6 nodes, 5 arrows per slide
- [ ] Bottom zone has content (not empty)
- [ ] No element overlaps (Y math checked)
- [ ] All text fits within its container
- [ ] All arrows have labels
- [ ] All async icon calls use `await`
- [ ] All helpers receive both `slide` AND `pres`
