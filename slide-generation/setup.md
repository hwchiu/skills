# Setup Guide — Prerequisites & Installation

> Everything you need to install before generating slides. Follow this guide on a fresh machine to get a working environment.

## System Requirements

| Tool | Minimum Version | Check Command | Purpose |
|------|----------------|---------------|---------|
| **Node.js** | `v18+` (recommended `v20+`) | `node -v` | Runs all slide generation scripts |
| **npm** | `v9+` | `npm -v` | Installs JavaScript dependencies |
| **Python 3** | `v3.8+` (recommended `v3.10+`) | `python3 --version` | Merge script (ZIP-level PPTX merge via `lxml`) |
| **pip** | `v21+` | `pip3 --version` | Installs Python dependencies |

### Platform Notes

- **macOS:** Install Node.js via [Homebrew](https://brew.sh/) — `brew install node`
- **Linux (Ubuntu/Debian):** `sudo apt install nodejs npm python3 python3-pip`
- **Windows:** Use [nvm-windows](https://github.com/coreybutler/nvm-windows) for Node.js; Python from [python.org](https://www.python.org/downloads/)

## Step 1: Clone & Install Node Dependencies

```bash
git clone <your-repo-url>
cd moslides
npm install
```

This installs the following packages (defined in `package.json`):

| Package | Version | Purpose |
|---------|---------|---------|
| `pptxgenjs` | `^4.0.1` | Core PPTX generation library — creates slides programmatically |
| `react` | `^19.2.4` | Required by `react-icons` for rendering icon components |
| `react-dom` | `^19.2.4` | Server-side rendering of react-icons to SVG strings |
| `react-icons` | `^5.6.0` | Icon library — provides `FaServer`, `FaDocker`, etc. |
| `sharp` | `^0.34.5` | Converts SVG icon strings to PNG base64 for embedding in PPTX |

### Troubleshooting `sharp`

`sharp` uses native C++ bindings. If installation fails:

```bash
# macOS: ensure build tools are available
xcode-select --install

# Linux: install build dependencies
sudo apt install build-essential libvips-dev

# Force rebuild
npm rebuild sharp
```

## Step 2: Install Python Dependencies

The merge script (`src/merge.js`) shells out to Python for ZIP-level PPTX merging:

```bash
pip3 install lxml
```

| Package | Purpose |
|---------|---------|
| `lxml` | XML parsing for PPTX slide XML manipulation during merge |

> **Note:** `python-pptx` is NOT required. The merge script uses direct ZIP/XML manipulation via `lxml` for reliability.

## Step 3: Create Output Directory

```bash
mkdir -p output
```

## Step 4: Verify Installation

```bash
# Test a single part build
node src/part1.js
# Expected: "✅ output/part1.pptx written (X slides)"

# Test the merge (requires all parts to be built first)
node src/merge.js
# Expected: "✅ output/cloud_native_slides_final.pptx (N slides)"
```

## Project File Structure

After setup, your project should look like:

```
moslides/
├── package.json          # Node.js dependencies
├── package-lock.json     # Locked dependency versions
├── node_modules/         # Installed packages (auto-created by npm install)
├── skills/               # This documentation
│   ├── README.md
│   ├── setup.md          # ← You are here
│   ├── design-tokens.md
│   ├── layout-guide.md
│   ├── cookbook.md
│   ├── helpers-api.md
│   ├── decision-trees.md
│   ├── anti-patterns.md
│   └── template-workflow.md
├── src/
│   ├── design-system.js  # Color/font tokens + theme system
│   ├── helpers.js        # Shared slide-building helpers
│   ├── icon-helper.js    # react-icons → PNG base64 converter
│   ├── part1.js          # Slide scripts (one per section)
│   ├── part2.js
│   ├── ...
│   └── merge.js          # Merges all part .pptx into final deck
├── output/               # Generated .pptx files (gitignored)
└── templates/            # (Optional) .pptx master templates
```

## Starting a New Presentation

To create a brand new slide deck using this system:

```bash
# 1. Copy the infrastructure
mkdir my-new-deck && cd my-new-deck
npm init -y
npm install pptxgenjs react react-dom react-icons sharp
pip3 install lxml

# 2. Copy the core files from this project
cp <moslides>/src/design-system.js src/
cp <moslides>/src/helpers.js src/
cp <moslides>/src/icon-helper.js src/
cp <moslides>/src/merge.js src/

# 3. Create your first part file
cat > src/part1.js << 'EOF'
const pptxgen = require("pptxgenjs");
const { COLORS, FONTS, setTheme } = require("./design-system");
setTheme("light");
const { initSlide, addSlideHeader } = require("./helpers");

async function main() {
  const pres = new pptxgen();
  const slide = initSlide(pres);
  addSlideHeader(slide, pres, { title: "My First Slide", partLabel: "PART 1" });
  await pres.writeFile({ fileName: "output/part1.pptx" });
  console.log("✅ output/part1.pptx written");
}
main();
EOF

# 4. Build
mkdir -p output
node src/part1.js
```

## Version Compatibility Matrix

Tested and confirmed working with:

| Component | Tested Version | Notes |
|-----------|---------------|-------|
| Node.js | v24.13.0 | Also works with v18, v20, v22 |
| npm | v11.6.2 | Any npm that ships with Node 18+ |
| Python | 3.11.6 | Also works with 3.8, 3.9, 3.10 |
| pptxgenjs | 4.0.1 | Major API: `addText`, `addShape`, `addImage` |
| sharp | 0.34.5 | Requires Node 18+ |
| react | 19.2.4 | Only used for SSR of icons, not UI |
| react-icons | 5.6.0 | `react-icons/fa` (FontAwesome 5) |
| lxml | latest | Python XML library for merge |

## Common Issues

### `Error: Cannot find module 'pptxgenjs'`
→ Run `npm install` in the project root.

### `sharp: Installation error`
→ See [sharp installation docs](https://sharp.pixelplumbing.com/install). On Apple Silicon: `npm install --arch=arm64 sharp`.

### `python3: command not found`
→ Install Python 3. On macOS: `brew install python3`. On Ubuntu: `sudo apt install python3`.

### `ModuleNotFoundError: No module named 'lxml'`
→ Run `pip3 install lxml`.

### `Error: ENOENT: no such file or directory, open 'output/part1.pptx'`
→ Create the output directory: `mkdir -p output`.
