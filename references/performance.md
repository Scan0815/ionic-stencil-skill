# Performance — Stencil bundling & Ionic icon inlining

Two production-grade optimizations that cut cold-load request count
dramatically on Stencil + Ionic apps. Both are config-only (no
component code changes), both are reversible with a one-line revert,
and both compound well.

When you'll need this: as soon as a deployed Stencil + Ionic app's
DevTools Network tab on cold load shows hundreds of requests on first
paint, especially `time-outline.svg`, `mail-outline.svg`, … and
`p-XYZ.entry.js`, `p-ABC.entry.js`, … patterns. Both indicate the
default lazy-load is over-fragmenting.

---

## 1. Inline `<ion-icon>` SVGs via `addIcons()`

### Problem

`<ion-icon name="time-outline">` defaults to **fetching** the SVG via
`/svg/<name>.svg` over HTTP the first time each icon renders. With ~70
unique icons in a typical admin app, that's ~70 separate cold-load
requests. Each ~120–170 ms via a normal CDN route.

### Solution

Register every icon used in the app once at boot via `addIcons()` from
`ionicons`. The icons are inlined as `data:image/svg+xml;…` URIs in the
JS bundle, so `<ion-icon name="…">` resolves synchronously in-process
with **zero network**.

### Setup

```ts
// src/global/icons.ts
import { addIcons } from 'ionicons';
import {
  // Alphabetical (so the diff is small when a teammate adds one)
  addOutline,
  alertCircleOutline,
  // ... import every icon you use
  trashOutline,
  warningOutline,
} from 'ionicons/icons';

export function registerIcons(): void {
  addIcons({
    // kebab-case key (the value used in <ion-icon name="...">)
    // → camelCase value (the named export from ionicons/icons)
    'add-outline': addOutline,
    'alert-circle-outline': alertCircleOutline,
    // ...
    'trash-outline': trashOutline,
    'warning-outline': warningOutline,
  });
}
```

```ts
// src/global/app.ts (the file pointed to by `globalScript:` in stencil.config.ts)
import '@ionic/core';
import { registerIcons } from './icons';

export default async () => {
  // Inline ion-icon SVGs so they resolve from the bundle, not the network.
  registerIcons();

  // … your other globalScript setup
};
```

The Stencil compiler runs `globalScript` exactly once at app boot,
before any component mounts. `addIcons()` is idempotent and safe to
call any number of times.

### Trade-off

- **Adds** ~1.6 kB raw per icon to the bundle (~120 kB raw / ~50–70 kB
  gzipped for ~75 icons)
- **Removes** ~75 separate HTTP requests on every cold load

Below ~30 icons the saving is marginal. Above ~50 the saving is huge.

### Drift audit (catch unregistered icons)

`<ion-icon name="missing">` will silently fall back to fetching
`/svg/missing.svg` if the icon is not registered — no build error, no
runtime error. Run this periodically:

```bash
# Tags used but not registered → still fetched per-icon
comm -23 \
  <(grep -rEho '<ion-icon[^>]*name=["'\''][^"'\'']+["'\'']' src \
      --include='*.tsx' --include='*.ts' --include='*.html' \
      | grep -oE 'name=["'\''][^"'\'']+["'\'']' \
      | sed -E 's/name=["'\'']([^"'\'']*)["'\'']/\1/' | sort -u) \
  <(grep -oE "'[a-z][a-z0-9-]+'" src/global/icons.ts | tr -d "'" | sort -u)
```

Also handles dynamic name attributes — `<ion-icon name={cond ? 'a-outline' : 'b-outline'}>`,
`icon: 'kebab-name'` in route/menu data, and helper functions returning
icon names — by adapting the first command:

```bash
# Add ternary literals + data-driven icon: 'name' patterns
{
  grep -rEho '<ion-icon[^>]*name=["'\''][^"'\'']+["'\'']' src --include='*.tsx' \
    | grep -oE 'name=["'\''][^"'\'']+["'\'']' \
    | sed -E 's/name=["'\'']([^"'\'']*)["'\'']/\1/'
  grep -rEh '<ion-icon[^>]*name=\{[^}]+\}' src --include='*.tsx' \
    | grep -oE "'[a-z][a-z0-9-]+(-outline|-sharp)?'" | tr -d "'"
  grep -rEho "icon:\s*['\"][a-z][a-z0-9-]+(-outline|-sharp)?['\"]" src --include='*.tsx' \
    | grep -oE "['\"][a-z][a-z0-9-]+['\"]" | tr -d "'\""
} | sort -u
```

---

## 2. Manual chunking via `bundles:` in `stencil.config.ts`

### Problem

Stencil 4 lazy-loads each `@Component`-decorated tag as its own JS
chunk by default. With Ionic in the mix (also Stencil-built), every
`<ion-button>`, `<ion-input>`, `<ion-modal>` etc. is **also** a
separate lazy chunk. A console-style app with ~60 first-party
components and ~50 used Ionic components will emit **~150
`p-*.entry.js` chunks** plus ~70 shared chunks — every chunk is one
HTTP request on first render.

### Solution

The `bundles:` array in `stencil.config.ts` forces named tags to share
one entry chunk. Stencil's automatic lazy-loading still applies;
`bundles:` only **groups** tags within their resulting chunks.

Stencil 4 accepts third-party tags (`ion-*`) too because the loader
registers everything reachable through `globalScript` imports.

### Pattern

```ts
// stencil.config.ts
export const config: Config = {
  // … other config
  bundles: [
    // ── First-party (your @Component tags) ───────────────────

    // App shell — ships eager with the entry
    { components: ['app-root', 'app-login', 'page-header', 'page-content'] },

    // Per feature directory: one bundle if the feature has ≥3 components.
    // Smaller features (≤2 components) merge with adjacent admin workflows.
    { components: ['user-list', 'user-modal', 'user-edit', 'user-create'] },
    { components: ['chat-list', 'chat-detail', 'chat-bubble', 'chat-input'] },

    // ── Ionic primitives + routing infra (every page needs them) ───
    { components: [
      'ion-app', 'ion-content', 'ion-header', 'ion-footer', 'ion-toolbar',
      'ion-title', 'ion-buttons', 'ion-button', 'ion-icon', 'ion-label',
      'ion-text', 'ion-note', 'ion-spinner', 'ion-progress-bar',
      // routing infra — used by app-root on first paint
      'ion-router', 'ion-route', 'ion-router-outlet', 'ion-nav',
    ]},

    // ── Ionic forms (modals + edit screens) ────────────────────
    { components: [
      'ion-input', 'ion-input-password-toggle', 'ion-textarea',
      'ion-checkbox', 'ion-toggle', 'ion-radio', 'ion-radio-group',
      'ion-select', 'ion-select-option', 'ion-searchbar',
      'ion-segment', 'ion-segment-button',
    ]},

    // ── Ionic layout / list ────────────────────────────────────
    { components: [
      'ion-list', 'ion-item', 'ion-item-divider', 'ion-grid', 'ion-row',
      'ion-col', 'ion-card', 'ion-card-header', 'ion-card-title',
      'ion-card-subtitle', 'ion-card-content', 'ion-chip', 'ion-badge',
      'ion-avatar',
    ]},

    // ── Ionic overlay (modals, popovers, accordions, menus) ────
    { components: [
      'ion-menu', 'ion-menu-button', 'ion-menu-toggle', 'ion-split-pane',
      'ion-back-button', 'ion-modal', 'ion-popover',
      'ion-accordion', 'ion-accordion-group',
    ]},
  ],
  // … other config
};
```

### Grouping rules

**First-party (your tags):**
- One bundle per feature directory if the feature has **≥3** components
- Sub-3 features **merge with an adjacent admin workflow** (e.g.
  `hosting + storage`, `members + users`, `monitoring + audit`)
- The **app shell** (`app-root`, login page, top-level frames) gets
  its own small bundle — ships eager
- **Don't merge unrelated features** to save a chunk — modal code
  loading with list code is fine within a feature, but loading
  `chat-bubble` while rendering `user-list` is wasted bytes

**Ionic (`ion-*`) — group by usage cluster:**
- **Primitives + routing infra** — needed at first paint (router,
  toolbar, button, icon, spinner, …)
- **Forms** — input, checkbox, radio, select, segment, toggle
- **Layout / list** — list, item, card, grid, chip, badge, avatar
- **Overlay** — modal, popover, menu, accordion

This 4-cluster grouping covers ~50 used Ionic components into 4
chunks. The trade-off: even if a route only renders `<ion-button>`,
the whole "primitives" bundle loads. But every page needs primitives
— it's a no-cost grouping.

### Verification

```bash
# Before
ls dist/<your-app>/www/build/p-*.entry.js | wc -l

# Apply config, rebuild
npx stencil build  # or your build command

# After — should drop dramatically
ls dist/<your-app>/www/build/p-*.entry.js | wc -l
```

In DevTools Network on the running app:
- Reload `/` → app shell + primitives + layout chunks load
- Navigate to any feature route → exactly one new entry chunk loads
- Total `*.js` request count is in the **10s, not 100s**
- No `console.error` about unknown custom elements (would mean a tag
  was misspelled in `bundles:`)

### Drift audit (catch unbundled tags)

```bash
# All Ionic tags used in the app
USED=$(grep -rEho '<ion-[a-z][a-z-]+' src --include='*.tsx' \
  | sed 's/<//' | sort -u)

# All Ionic tags in the bundles config
BUNDLED=$(grep -oE "'ion-[a-z][a-z-]+'" stencil.config.ts | tr -d "'" | sort -u)

# Diff: tags used but not bundled (will fall through to per-component chunk)
comm -23 <(echo "$USED") <(echo "$BUNDLED")
```

Same shape works for first-party tags by adapting the first command:

```bash
grep -rEho "tag:\s*['\"][a-z][a-z0-9-]+['\"]" src --include='*.tsx' \
  | grep -oE "['\"][a-z][a-z0-9-]+['\"]" | tr -d "'\""
```

A misspelled tag in `bundles:` is **not a build error** — Stencil
silently falls through to default per-component chunking for that tag.
Run the drift audit after adding components or before each release.

### Risks + mitigations

| Risk | Mitigation |
|---|---|
| Misspelled tag → falls through to default chunk (one extra request) | Drift-audit grep + manual smoke-test each route |
| Tree-shaking weakens within a bundle (modal code loads with list code) | Acceptable — modals follow list views in workflow |
| One bundle becomes too heavy for first paint | Split that one bundle (e.g. project-shell with 16 tags → list-bundle vs detail-pages-bundle), don't revert all |
| Service worker caching old chunk filenames | Stencil writes content-hashed filenames, but if `serviceWorker:` is enabled you may need a cache-bust strategy on deploy |

### When NOT to use `bundles:`

- App has **<20 components total** — Stencil's default chunking is
  fine; manual grouping adds maintenance for negligible benefit
- App is shipped as a **published library** (consumer drops `dist/`
  into their app) — use `dist-custom-elements` outputTarget instead;
  manual `bundles:` is for SPA deployment scenarios

### Rollback

Single revert: delete the `bundles: [...]` block. Stencil reverts to
default per-component chunking. `git checkout -- stencil.config.ts`.

---

## Combined impact (real numbers)

From a production console with 60 first-party + 53 used Ionic tags:

| Metric | Default | After both |
|---|---:|---:|
| `*.svg` icon requests on cold load | ~75 | **0** |
| `p-*.entry.js` chunks | 150 | **57** (−62 %) |
| `p-*.js` chunks total | 224 | **116** (−48 %) |
| Bundle size on disk (raw) | unchanged | +~120 kB icons |
| Bundle size gzipped | unchanged | +~50–70 kB |
| Total HTTP requests on cold load | 947 | ~700 |

The win is **request count**, not byte count. HTTP/2 multiplexing
already helps, but each request still has connection-pool, parse, and
eval overhead — and CDN edge caches can't always serve sequential
deeply-nested chunk requests in parallel.

For DOMContentLoaded on real cold loads, expect **30–50 % faster**.
