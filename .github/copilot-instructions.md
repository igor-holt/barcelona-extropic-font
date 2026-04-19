# Barcelona-Extropic Font System

This repository contains the Barcelona-Extropic font delivery system: a Rust WASM font engine + Cloudflare Workers deployment pipeline that serves brand-consistent typography across web, mobile (iOS WKWebView), and Google Workspace environments.

## Build, Test, and Deploy

### Prerequisites

```bash
# Install Rust + WASM toolchain
rustup target add wasm32-unknown-unknown
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh

# Install Cloudflare CLI
npm i -g wrangler
wrangler login
```

### Run Tests

```bash
# Run all tests
cargo test --features ambigraph

# Run a single test
cargo test --features ambigraph proof_of_render_contains_hash

# Run tests matching a pattern
cargo test --features ambigraph proof
```

**Critical:** All 34 tests must pass before deployment. The test suite validates:
- Contrast ratio invariants (12:1 stem:serif)
- QoS budget enforcement (free tier vs premium)
- Proof-of-render hash determinism
- Chirality transform correctness

### Build WASM Binary

```bash
wasm-pack build \
  --target web \
  --release \
  --features ambigraph \
  --out-dir pkg \
  -- --no-default-features
```

Output: `pkg/be_ambigraph_bg.wasm` (the font engine binary) and `pkg/be_ambigraph.js` (JS bindings).

### Deploy

The deployment is idempotent and resumable via `.deploy_state` tracking:

```bash
# Set your gateway source directory and URL
export GATEWAY_DIR=/path/to/coalition-gateway
export GATEWAY_URL=https://coalition-gateway.YOUR_SUBDOMAIN.workers.dev

./deploy.sh
```

To re-run a specific step after failure, delete the corresponding line from `.deploy_state`.

### Smoke Test

```bash
GATEWAY_URL=https://your-gateway.workers.dev ./smoke_test.sh
```

Verifies: manifest delivery, CSS endpoint, workspace JSON, iOS injection script, CSS custom properties.

## Architecture

### Font Engine (lib.rs)

The core Rust implementation (~800 lines) contains:

- **FontArchitect**: WASM-exported struct that renders glyphs as SVG with proof-of-render SHA-256 hashing
- **GlyphIndex**: O(1) character → render function dispatch
- **Chirality transforms**: SVG matrix transforms for horizontal/vertical flipping and 180° rotation (zero texture re-upload)
- **QoS budget system**: Free tier gets basic glyphs (A, C, O); premium tier unlocks all glyphs
- **Ambigraph module** (`--features ambigraph`): Glyph pair interpolation for morphing animations

Key constants (locked in "Seismic Baseline v1.0"):
- `CONTRAST_RATIO: f32 = 12.0` — Didone-class stem:serif ratio
- `STEM_WIDTH: f32 = 12.0` — Base stem width in SVG units
- `GRID_SNAP: f32 = 4.0` — All coordinates snap to 4px grid

The font engine is **stateless** and **deterministic**: same params → same SVG → same hash.

### Gateway Integration (font_route.js)

Cloudflare Workers handler that serves font assets via `/fonts/be/*`:

- `/fonts/be/barcelona-extropic.css` — Universal CSS with `@font-face`, custom properties (`--be-stem`, `--be-tracking`, etc.), and utility classes (`.be-display`, `.be-kinetic`, `.be-chiral-h`)
- `/fonts/be/workspace.json` — Inline styles for Google Workspace Add-ons (no `@font-face` support)
- `/fonts/be/ios-inject.js` — WKUserScript for iOS WKWebView injection
- `/fonts/be/` — JSON manifest with endpoints and integration snippets

The route handler is **injected** into an existing `coalition-gateway` worker via `gateway.patch`:
1. Add `import { handleFontRoute } from './font_route';` at top of `src/index.ts`
2. Add `if (path.startsWith('/fonts/')) return handleFontRoute(request, env);` before the 404 handler

### R2 Storage (Optional)

For WOFF2 binary delivery (Tier 1), assets are uploaded to R2 bucket `be-font-assets` with versioned paths (`v1.0.0/barcelona-extropic.wasm`). The CSS and JS are also stored in R2 with `immutable` cache headers.

### Multi-Environment CSS Strategy

The system serves **identical brand presentation** across incompatible environments:

| Environment | Solution | File |
|-------------|----------|------|
| Web browsers | `@font-face` + WOFF2 from R2 | `barcelona-extropic.css` |
| Google Workspace Add-ons | Inline `style=""` attributes with Google Fonts fallbacks | `workspace.json` |
| iOS WKWebView | `WKUserScript` injected at `documentStart` | `ios-inject.js` |
| Electron, Claude artifacts | Standard `<link>` tag | `barcelona-extropic.css` |

All environments fall back to **Playfair Display** (Google Fonts CDN) when WOFF2 is unavailable. This ensures zero-config deployment to restricted environments like Google Workspace.

## Key Conventions

### Immutability & Hashing

The font engine uses **proof-of-render** hashing:
```rust
pub fn render_string_with_proof(&self, chars: &str) -> String
```

Output format:
```json
{
  "svg": "<svg>...</svg>",
  "sha256": "64-char hex hash",
  "params": {"baseline":"barcelona_extropic", "stem_width":12.0, ...}
}
```

The hash covers SVG content + params. This enables:
- Verifiable rendering (client can recompute and verify)
- Cache invalidation (hash changes when params change)
- Audit trail (hash proves specific params were used)

### Idempotent Deployment

`deploy.sh` tracks state in `.deploy_state` (one `DONE:step_name` per line). Each step:
1. Checks `state_done()` — skip if already completed
2. Executes (idempotent operations: `cargo test` reruns, `wrangler r2 bucket create` fails silently if exists)
3. Marks `state_mark()` on success

**To re-run a step:** Delete its line from `.deploy_state`.

### Chirality (Ambigraph Feature)

Chirality transforms are **pure SVG operations**:
```rust
pub enum ChiralAxis {
    Horizontal,      // matrix(-1,0,0,1, 2*cx, 0)
    Vertical,        // matrix(1,0,0,-1, 0, 2*cy)
    Rotation180,     // matrix(-1,0,0,-1, 2*cx, 2*cy)
    None,
}
```

GPU drivers optimize these to sign flips (not matrix multiplies) because coefficients are `±1`. No texture re-upload required.

### CSS Custom Properties

All brand values are defined as CSS variables in `font_route.js`:
```css
--be-stem: 12px
--be-tracking: 0.25em
--be-weight: 700
--be-ink: #f0ede8
--be-ground: #1a1e24
```

**Convention:** Never hardcode brand values in consuming code. Always reference `var(--be-stem)`, etc. This allows global updates via a single CSS variable change.

### Gateway Patching

The system **does not create a new Worker**. It patches an existing `coalition-gateway` worker by:
1. Copying `font_route.js` → `coalition-gateway/src/font_route.js`
2. Applying `gateway.patch` to `coalition-gateway/src/index.ts`

This keeps all routes under one domain and avoids CORS issues.

## File Inventory

| File | Purpose |
|------|---------|
| `lib.rs` | Rust WASM font engine (34 tests, 791 lines) |
| `wrangler.toml` | Gateway config patch (R2 bucket binding template) |
| `deploy.sh` | Master deployment script (idempotent, 9 steps) |
| `smoke_test.sh` | Post-deploy verification (9 checks) |
| `font_route.js` | Cloudflare Workers route handler for `/fonts/be/*` |
| `barcelona-extropic.css` | Universal CSS (not in repo root — generated by `font_route.js`) |
| `gateway.patch` | Diff to apply to `coalition-gateway/src/index.ts` |
| `bio-profile-hub.html` | Example integration (Igor Holt profile page) |

## Known State (Seismic Log)

**Current deployment status:**
- Rust source: **Verified** — 34/34 tests pass
- WASM binary: **Not built** — run `wasm-pack build` on deployment target
- R2 bucket: **Not created** — step 5 of `deploy.sh`
- Gateway patch: **Not applied** — requires `GATEWAY_DIR` env var
- Cloudflare account: `04c59c95ce8d0a0be98099b7f7e39d18`
- Gateway worker: `coalition-gateway` (16 workers total, confirmed live)

**After deployment, integration snippet:**
```html
<link rel="stylesheet" href="https://YOUR_GATEWAY/fonts/be/barcelona-extropic.css">
```

Or for Google Workspace:
```javascript
const styles = JSON.parse(UrlFetchApp.fetch('https://YOUR_GATEWAY/fonts/be/workspace.json').getContentText());
```

## Integration Examples

See `bio-profile-hub.html` for a real-world example showing:
- CSS custom properties usage
- `.be-display`, `.be-subtitle`, `.be-kinetic` utility classes
- Grid background (`.be-grid-bg`)
- Fallback font stack handling
