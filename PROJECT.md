# Obsidian Marp Slides - Technical Documentation

> For user documentation and getting started guides, see [README.md](README.md) and the [online documentation](https://samuele-cozzi.github.io/obsidian-marp-slides/).

## Overview

**Obsidian Marp Slides** is a plugin that integrates [Marp](https://marp.app/) (Markdown Presentation Ecosystem) into [Obsidian](https://obsidian.md/), enabling users to create, preview, and export slide presentations directly from Markdown files.

| Property | Value |
|----------|-------|
| Plugin ID | `marp-slides` |
| Version | 0.45.6 |
| Author | Samuele Cozzi |
| License | MIT |
| Min Obsidian Version | 0.15.0 |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Obsidian App                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌──────────────────┐    ┌───────────────┐  │
│  │  MarpSlides │───▶│  MarpPreviewView │───▶│   Marp Core   │  │
│  │   (Plugin)  │    │    (ItemView)    │    │   (Renderer)  │  │
│  └──────┬──────┘    └──────────────────┘    └───────────────┘  │
│         │                                                       │
│         │           ┌──────────────────┐    ┌───────────────┐  │
│         └──────────▶│    MarpExport    │───▶│   Marp CLI    │  │
│                     │   (Exporter)     │    │   (Export)    │  │
│                     └────────┬─────────┘    └───────────────┘  │
│                              │                                  │
│                     ┌────────▼─────────┐                       │
│                     │     FilePath     │                       │
│                     │   (Utilities)    │                       │
│                     └──────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

**Preview Pipeline:**
1. User opens Markdown file and triggers "Slide Preview" command
2. `MarpSlides` retrieves active `MarkdownView` and creates `MarpPreviewView`
3. `MarpPreviewView` uses Marp Core to render Markdown → HTML/CSS
4. Rendered slides displayed in split pane with file base path for assets
5. File change events trigger re-render automatically

**Export Pipeline:**
1. User triggers export command (PDF/HTML/PPTX/PNG)
2. `MarpSlides` creates `MarpExport` instance with current settings
3. `FilePath` resolves file paths, themes, and resources
4. `MarpExport` constructs CLI arguments and invokes Marp CLI
5. Marp CLI uses Chrome/Chromium for PDF/PPTX rendering

---

## Project Structure

```
obsidian-marp-slides/
├── src/
│   ├── main.ts                      # Plugin entry point, commands, settings
│   ├── config/
│   │   └── marp.config.js           # Marp engine configuration for markdown-it plugins
│   ├── utilities/
│   │   ├── settings.ts              # Settings interface and defaults
│   │   ├── marpExport.ts            # Export functionality (PDF, HTML, PPTX, PNG)
│   │   ├── filePath.ts              # File/path resolution utilities
│   │   ├── libs.ts                  # External library management
│   │   └── icons.ts                 # SVG icon definitions
│   └── views/
│       └── marpPreviewView.ts       # Slide preview rendering
├── tests/
│   ├── filePath.test.ts             # Path utility tests
│   └── __mocks__/
│       └── obsidian.ts              # Obsidian API mocks
├── docs/                            # User documentation
├── vault/samples/                   # Sample presentations
├── .github/workflows/
│   └── release-please.yml           # CI/CD pipeline
├── esbuild.config.mjs               # Build configuration
├── tsconfig.json                    # TypeScript configuration
├── jest.config.js                   # Test configuration
├── package.json                     # Dependencies and scripts
├── manifest.json                    # Obsidian plugin metadata
├── styles.css                       # Plugin styling
└── version-bump.mjs                 # Version management script
```

---

## Core Components

### MarpSlides (`src/main.ts:10-147`)

Main plugin class extending Obsidian's `Plugin`.

**Responsibilities:**
- Plugin lifecycle management (`onload`, `onunload`)
- Command registration (preview, export)
- Settings management
- Event listeners (file changes, cursor position)

**Key Methods:**
| Method | Line | Description |
|--------|------|-------------|
| `onload()` | 16 | Initializes plugin, registers views/commands |
| `loadSettings()` | 90 | Loads persisted settings |
| `saveSettings()` | 94 | Persists settings to disk |
| `showPreviewSlide()` | 112 | Opens preview pane |
| `exportFile()` | 104 | Triggers export operation |
| `onChange()` | 98 | Handles file modification events |

**Registered Commands:**
- `marp-slides:preview` - Slide Preview
- `marp-slides:export-pdf` - Export PDF
- `marp-slides:export-pdf-notes` - Export PDF with Notes
- `marp-slides:export-html` - Export HTML
- `marp-slides:export-pptx` - Export PPTX
- `marp-slides:export-png` - Export PNG

---

### MarpPreviewView (`src/views/marpPreviewView.ts:16-166`)

Custom view for rendering slides, extending Obsidian's `ItemView`.

**Responsibilities:**
- Marp Core initialization and configuration
- Theme loading from vault
- Slide rendering (Markdown → HTML)
- Cursor-to-slide synchronization
- Export action buttons

**Key Methods:**
| Method | Line | Description |
|--------|------|-------------|
| `onOpen()` | 58 | Initializes container, loads themes |
| `displaySlides()` | 131 | Renders Markdown to HTML slides |
| `onLineChanged()` | 89 | Scrolls to slide based on cursor |
| `addActions()` | 97 | Adds export buttons to view header |

**Marp Configuration** (lines 29-40):
```typescript
new Marp({
    container: { tag: 'div', id: '__marp-vscode' },
    slideContainer: { tag: 'div', 'data-marp-vscode-slide-wrapper': '' },
    html: this.settings.EnableHTML,
    inlineSVG: { enabled: true, backdropSelector: false },
    math: this.settings.MathTypesettings,
    minifyCSS: true,
    script: false
});
```

---

### MarpExport (`src/utilities/marpExport.ts:8-158`)

Handles exporting presentations to various formats.

**Responsibilities:**
- Building Marp CLI argument arrays
- Managing export types and options
- Browser path resolution
- Error handling for missing Chrome

**Key Methods:**
| Method | Line | Description |
|--------|------|-------------|
| `export()` | 16 | Main export orchestrator |
| `run()` | 101 | Sets up environment and executes CLI |
| `runMarpCli()` | 136 | Executes Marp CLI with arguments |

**Supported Export Types:**
| Type | CLI Flags | Output |
|------|-----------|--------|
| `pdf` | `--pdf` | PDF file |
| `pdf-with-notes` | `--pdf --pdf-notes --pdf-outlines` | PDF with speaker notes |
| `pptx` | `--pptx` | PowerPoint file |
| `png` | `--images --png` | PNG images |
| `html` | `--html --template [mode]` | HTML file |
| `preview` | `--html --preview` | Live preview server |

---

### FilePath (`src/utilities/filePath.ts:4-112`)

Utility class for file and path resolution.

**Responsibilities:**
- Vault base path resolution
- Absolute vs relative link format handling
- Theme directory resolution
- Plugin directory management

**Key Methods:**
| Method | Line | Description |
|--------|------|-------------|
| `getCompleteFileBasePath()` | 41 | Gets resource base path for assets |
| `getCompleteFilePath()` | 56 | Gets full file path for export |
| `getThemePath()` | 80 | Resolves custom theme directory |
| `getLibDirectory()` | 99 | Gets markdown-it plugins directory |
| `getMarpEngine()` | 106 | Gets Marp engine config path |

---

### MarpSlidesSettings (`src/utilities/settings.ts:1-21`)

Settings interface and defaults.

```typescript
interface MarpSlidesSettings {
    CHROME_PATH: string;           // Custom browser path for export
    ThemePath: string;             // Custom theme CSS directory
    EnableHTML: boolean;           // Allow HTML in Markdown
    MathTypesettings: string;      // 'mathjax' or 'katex'
    HTMLExportMode: string;        // 'bare' or 'bespoke'
    EXPORT_PATH: string;           // Custom export output directory
    EnableSyncPreview: boolean;    // Sync preview with cursor
    EnableMarkdownItPlugins: boolean; // Enable markdown-it extensions
}
```

**Default Values:**
| Setting | Default |
|---------|---------|
| CHROME_PATH | `''` (auto-detect) |
| ThemePath | `''` (none) |
| EnableHTML | `false` |
| MathTypesettings | `'mathjax'` |
| HTMLExportMode | `'bare'` |
| EXPORT_PATH | `''` (same as source) |
| EnableSyncPreview | `true` |
| EnableMarkdownItPlugins | `false` |

---

### Libs (`src/utilities/libs.ts:8-61`)

Manages external markdown-it plugin libraries.

**Responsibilities:**
- Check if libraries exist locally
- Download compiled plugins from GitHub releases
- Extract ZIP archive and cache plugins

**Library Source:** `https://github.com/samuele-cozzi/obsidian-marp-slides/releases/download/lib-v3/lib.zip`

**Included Plugins:**
- `markdown-it-container` - Custom containers
- `markdown-it-mark` - Text highlighting
- `markdown-it-kroki` - Diagram rendering via Kroki.io

---

### LineSelectionListener (`src/main.ts:255-300`)

Experimental feature for cursor-to-slide synchronization.

**Implementation:** Extends `EditorSuggest` (non-intrusive approach to track cursor)

**How it works:**
1. Listens to cursor position changes
2. Counts slide separators (`---`) before cursor
3. Parses YAML frontmatter to adjust slide count
4. Scrolls preview to corresponding slide

---

## Technology Stack

### Runtime Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@marp-team/marp-core` | ^3.9.0 | Core slide rendering engine |
| `@marp-team/marp-cli` | ^2.5.0 | Export engine (PDF, PPTX, HTML, PNG) |
| `@marp-team/marpit` | ^2.6.1 | Markdown presentation framework |
| `gray-matter` | ^4.0.3 | YAML frontmatter parsing |
| `fs-extra` | ^11.2.0 | Extended file system operations |
| `jszip` | ^3.10.1 | ZIP handling for library distribution |
| `request` | ^2.88.2 | HTTP requests for library download |

### Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `typescript` | ^4.9.5 | Type-safe development |
| `esbuild` | 0.17.3 | Fast bundler |
| `jest` | ^29.7.0 | Testing framework |
| `ts-jest` | ^29.1.2 | TypeScript support for Jest |
| `obsidian` | ^1.5.7-1 | Obsidian API types |
| `@typescript-eslint/*` | 5.29.0 | Linting |

### External Requirements

- **Chrome/Chromium/Edge** - Required for PDF, PPTX, and PNG export
- **Node.js** - Development and build environment

---

## Development Setup

### Prerequisites

- Node.js (v16+)
- npm
- Obsidian (for testing)

### Installation

```bash
# Clone the repository
git clone https://github.com/samuele-cozzi/obsidian-marp-slides.git
cd obsidian-marp-slides

# Install dependencies
npm install
```

### Build Commands

| Command | Description |
|---------|-------------|
| `npm run dev` | Watch mode with inline sourcemaps |
| `npm run build` | TypeScript check + production build |
| `npm run test` | Run tests with coverage |
| `npm run test:watch` | Run tests in watch mode |
| `npm run version` | Bump version in manifest |

### Development Workflow

1. **Start watch mode:**
   ```bash
   npm run dev
   ```

2. **Link to Obsidian vault:**
   - Copy or symlink the project directory to your vault's `.obsidian/plugins/marp-slides/`
   - Or set up the vault's plugin directory to point to your development folder

3. **Enable plugin:**
   - Open Obsidian Settings → Community Plugins
   - Enable "Marp Slides"
   - Use "Reload app without saving" (Ctrl/Cmd+R) after changes

4. **Debug:**
   - Open Developer Tools (Ctrl/Cmd+Shift+I)
   - Check Console for logs and errors

### Build Configuration (`esbuild.config.mjs`)

- **Entry:** `main.ts`
- **Output:** `main.js`
- **Format:** CommonJS
- **Target:** ES2018
- **External:** `obsidian`, `electron`, `@codemirror/*`
- **Production:** Minified, no sourcemap
- **Development:** Inline sourcemap

---

## Testing

### Framework

- **Jest** with **ts-jest** preset
- Coverage reporting via **lcov**
- Mocks for Obsidian API

### Running Tests

```bash
# Run all tests with coverage
npm run test

# Watch mode
npm run test:watch
```

### Test Structure

```
tests/
├── filePath.test.ts      # Path resolution tests
├── coverage/             # Coverage reports (generated)
└── __mocks__/
    └── obsidian.ts       # Obsidian API mocks
```

### Coverage

Coverage reports are generated in `tests/coverage/` and uploaded to CodeClimate during CI.

---

## CI/CD Pipeline

### GitHub Actions Workflow (`.github/workflows/release-please.yml`)

**Trigger:** Push to `main` branch

### Jobs

#### 1. release-please
- Uses `google-github-actions/release-please-action@v3`
- Analyzes commits for version bump
- Creates release PR if warranted
- Generates changelog in `docs/CHANGELOG.md`

#### 2. release-plugin (if release created)
1. Updates `manifest.json` version
2. Commits version update
3. Builds plugin (`npm install && npm run build`)
4. Runs tests with CodeClimate coverage upload
5. Packages artifacts:
   - `main.js`
   - `manifest.json`
   - `styles.css`
   - `obsidian-marp-slides-{version}.zip`
6. Uploads to GitHub release

### Release Artifacts

| File | Description |
|------|-------------|
| `main.js` | Compiled plugin code |
| `manifest.json` | Plugin metadata |
| `styles.css` | Plugin styling |
| `obsidian-marp-slides-{version}.zip` | Complete plugin package |

---

## Configuration Options Reference

### CHROME_PATH
**Type:** `string` | **Default:** `''`

Custom path to Chrome, Chromium, or Edge browser for PDF/PPTX/PNG export. If empty, Marp CLI auto-detects installed browsers.

### ThemePath
**Type:** `string` | **Default:** `''`

Vault-relative path to directory containing custom Marp theme CSS files. Themes are loaded on preview open.

### EXPORT_PATH
**Type:** `string` | **Default:** `''`

Custom output directory for exports. If empty, exports to same directory as source file. Does not affect HTML export.

### EnableHTML
**Type:** `boolean` | **Default:** `false`

Allow HTML elements in Marp Markdown. Use with caution.

### MathTypesettings
**Type:** `'mathjax' | 'katex'` | **Default:** `'mathjax'`

Math rendering library. Can be overridden per-slide via frontmatter.

### HTMLExportMode
**Type:** `'bare' | 'bespoke'` | **Default:** `'bare'`

HTML export template. `bespoke` is experimental and provides interactive features.

### EnableSyncPreview
**Type:** `boolean` | **Default:** `true`

(Experimental) Synchronize slide preview with editor cursor position.

### EnableMarkdownItPlugins
**Type:** `boolean` | **Default:** `false`

(Experimental) Enable markdown-it plugins for containers, marks, and Kroki diagrams.

---

## Obsidian API Integration

### Used APIs

| API | Usage |
|-----|-------|
| `Plugin` | Base class for plugin |
| `ItemView` | Custom preview view |
| `MarkdownView` | Access editor content |
| `PluginSettingTab` | Settings UI |
| `EditorSuggest` | Cursor position tracking |
| `Vault` | File operations |
| `FileSystemAdapter` | Path resolution |
| `WorkspaceLeaf` | View management |

### Registered Entities

| Type | ID/Name |
|------|---------|
| View | `marp-preview-view` |
| Icons | `slides-preview-marp`, `slides-marp-export-pdf`, `slides-marp-export-pptx`, `slides-marp-slide-present` |
| Commands | 6 commands (see MarpSlides section) |
| Ribbon | Preview button |

---

## Known Limitations

- **Wiki Links** not supported in slides
- **Mobile App** plugin is in alpha state
- **Export** (except HTML) requires Chrome/Chromium/Edge installed
- **Sync Preview** is experimental and may have edge cases

---

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make changes with tests
4. Submit a pull request

For bug reports and feature requests, use [GitHub Issues](https://github.com/samuele-cozzi/obsidian-marp-slides/issues).
