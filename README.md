# Barcelona-Extropic Font System

A production-ready, multi-environment typography system combining Rust WASM font rendering with Cloudflare Workers edge delivery.

## Features

- 🎨 **Didone-class font rendering** — 12:1 contrast ratio, mathematically verified
- 🔐 **Proof-of-render hashing** — SHA-256 integrity verification for every SVG output
- 🌐 **Universal deployment** — Web, iOS WKWebView, Google Workspace Add-ons, Electron
- ⚡ **Edge delivery** — Cloudflare Workers + R2 with immutable caching
- 🎭 **Chirality transforms** — GPU-optimized SVG mirroring (horizontal, vertical, 180°)
- 🧪 **34 test suite** — Full invariant coverage before every deploy

## Quick Start

```bash
# Prerequisites
rustup target add wasm32-unknown-unknown
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
npm i -g wrangler

# Run tests
cargo test --features ambigraph

# Deploy
export GATEWAY_DIR=/path/to/coalition-gateway
export GATEWAY_URL=https://your-gateway.workers.dev
./deploy.sh

# Verify
./smoke_test.sh
```

## Integration

### Web / Electron / Claude Artifacts

```html
<link rel="stylesheet" href="https://YOUR_GATEWAY/fonts/be/barcelona-extropic.css">

<h1 class="be-display">Barcelona Extropic</h1>
<p class="be-subtitle">Didone · 12:1 Contrast · Seismic Baseline</p>
```

### Google Workspace Add-ons

```javascript
// Apps Script (no @font-face support)
const styles = JSON.parse(
  UrlFetchApp.fetch('https://YOUR_GATEWAY/fonts/be/workspace.json').getContentText()
);

const card = CardService.newTextParagraph()
  .setText('<span style="' + styles.display + '">Barcelona Extropic</span>');
```

### iOS WKWebView

```swift
let script = WKUserScript(
  source: try! String(contentsOf: URL(string: "https://YOUR_GATEWAY/fonts/be/ios-inject.js")!),
  injectionTime: .atDocumentStart,
  forMainFrameOnly: false
)
webView.configuration.userContentController.addUserScript(script)
```

## Architecture

```
lib.rs (791 lines)
  ├─ FontArchitect      → WASM-exported render engine
  ├─ GlyphIndex         → O(1) character dispatch
  ├─ ChiralAxis         → SVG matrix transforms
  └─ Ambigraph          → Glyph interpolation (optional feature)

font_route.js
  ├─ /fonts/be/barcelona-extropic.css  → Universal CSS
  ├─ /fonts/be/workspace.json          → Google Workspace inline styles
  ├─ /fonts/be/ios-inject.js           → iOS WKUserScript
  └─ /fonts/be/                        → JSON manifest

deploy.sh (9 steps, idempotent)
  ├─ Prerequisites check
  ├─ WASM toolchain setup
  ├─ cargo test --features ambigraph (MUST PASS)
  ├─ wasm-pack build
  ├─ R2 bucket creation
  ├─ Asset upload (WASM, JS, CSS)
  ├─ Gateway patch application
  ├─ wrangler deploy
  └─ Smoke test
```

## CSS Custom Properties

All brand values are CSS variables:

```css
--be-stem:          12px        /* Stem width */
--be-tracking:      0.25em      /* Letter spacing */
--be-weight:        700         /* Font weight */
--be-ink:           #f0ede8     /* Foreground color */
--be-ground:        #1a1e24     /* Background color */
```

Use `.be-display`, `.be-subtitle`, `.be-kinetic` utility classes for instant brand consistency.

## Design Specifications

See `barcelona-extropic.fig` for Figma design tokens, specimen sheets, and integration examples.

## Deployment Status

- ✅ Rust source (34/34 tests pass)
- ⏳ WASM binary (build on deploy)
- ⏳ R2 bucket (created by deploy.sh)
- ⏳ Gateway patch (requires GATEWAY_DIR)

## License

MIT License - See LICENSE file for details.

## Documentation

See `.github/copilot-instructions.md` for detailed architecture, conventions, and deployment procedures.
