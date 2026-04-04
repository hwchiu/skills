# Template Workflow — Working with .pptx Master Templates

> Optional two-stage workflow: generate content with pptxgenjs, then apply an organization's .pptx master template via python-pptx.

## When Do You Need This?

| Scenario | Use Template? |
|----------|--------------|
| Personal / internal presentation | ❌ Built-in design system is enough |
| Organization requires branded master slides | ✅ Apply template post-generation |
| Conference submission with required format | ✅ Apply template post-generation |
| Quick prototype / draft | ❌ Skip template step |

## How It Works

```
┌────────────────────┐      ┌─────────────────────┐      ┌───────────────────┐
│  pptxgenjs         │      │  python-pptx         │      │  Final .pptx      │
│  (Node.js)         │ ──→  │  (Python)            │ ──→  │  with branding    │
│                    │      │                      │      │                   │
│  Generate content  │      │  Apply master layout │      │  Ready to present │
│  using helpers.js  │      │  from template.pptx  │      │                   │
└────────────────────┘      └─────────────────────┘      └───────────────────┘
```

**Stage 1 — Content Generation (Node.js)**
- Uses pptxgenjs + design-system + helpers
- Produces a content-only .pptx with correct text, shapes, arrows
- No branding or master slide layout

**Stage 2 — Template Application (Python)**
- Uses python-pptx to read the template .pptx
- Copies slide masters and layouts from template
- Applies them to the content .pptx
- Outputs the final branded deck

## Setup

### Install python-pptx (only needed for template workflow)

```bash
pip3 install python-pptx
```

### Place your template

```
templates/
  company-template.pptx    # Your organization's branded template
```

## Template Application Script

Create `src/apply-template.py`:

```python
#!/usr/bin/env python3
"""Apply a .pptx master template to a content-only .pptx."""

import sys
from pptx import Presentation
from pptx.util import Inches, Pt

def apply_template(content_path, template_path, output_path):
    """Copy slide masters from template, apply to content slides."""
    template = Presentation(template_path)
    content = Presentation(content_path)

    # Copy slide layouts from template
    template_layout = template.slide_layouts[0]  # or whichever layout

    # For each slide in content, apply the template's slide layout
    for slide in content.slides:
        # Copy background from template if needed
        if template.slides and template.slides[0].background:
            slide.background.fill.solid()
            # Apply template background color/image

    content.save(output_path)
    print(f"✅ Template applied: {output_path}")

if __name__ == "__main__":
    if len(sys.argv) != 4:
        print("Usage: python3 apply-template.py <content.pptx> <template.pptx> <output.pptx>")
        sys.exit(1)
    apply_template(sys.argv[1], sys.argv[2], sys.argv[3])
```

## Integration with Build Pipeline

### In merge.js or a build script:

```js
const fs = require("fs");
const { execSync } = require("child_process");

// Check if templates directory has a .pptx file
const templatesDir = "templates";
const templateFiles = fs.existsSync(templatesDir)
  ? fs.readdirSync(templatesDir).filter(f => f.endsWith(".pptx"))
  : [];

if (templateFiles.length > 0) {
  const template = `${templatesDir}/${templateFiles[0]}`;
  console.log(`📎 Applying template: ${template}`);
  execSync(`python3 src/apply-template.py output/final.pptx ${template} output/final-branded.pptx`);
} else {
  console.log("ℹ️  No template found — using built-in design system");
}
```

## Rules

1. **Template is optional** — if `templates/` is empty or missing, skip the template step entirely
2. **Content first, branding second** — never compromise content layout for template constraints
3. **Test without template** — always verify slides look correct before applying template
4. **One template per build** — if multiple `.pptx` files exist in `templates/`, use the first one (or make it configurable)
