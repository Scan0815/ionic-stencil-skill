---
name: ionic-stencil
description: |
  Use this skill whenever the user is building, debugging, or extending a project that uses Ionic Framework components (@ionic/core) together with StencilJS (@stencil/core). Triggers include: creating custom web components with StencilJS, using ion-* components in Stencil-built apps, styling Ionic components (Shadow DOM, CSS variables, ::part()), handling Ionic lifecycle hooks, setting up an Ionic + Stencil project, wiring up events between components, or asking about @Component, @Prop, @State, @Event, @Listen, @Watch, @Element, or @Method decorators. Also trigger when the user asks about Shadow DOM encapsulation, scoped vs. shadow components, slot patterns, or theming with CSS custom properties in Ionic. Performance triggers — bundle-size or cold-load optimization for Stencil/Ionic apps, ion-icon SVG fetches, manual chunk grouping via `bundles:`, `addIcons()`, or "too many p-*.entry.js chunks" type symptoms. If the user is building any kind of app or component library with these two frameworks — even if they don't say "Ionic" or "Stencil" explicitly — use this skill.
---

# Ionic + StencilJS Development Skill

## Core Concept

**Stencil** is the compiler behind Ionic's components. When you build with `@stencil/core`, you're using the same toolchain Ionic itself is built on. Ionic components (`ion-*`) are just Stencil-built Web Components, so they integrate naturally into custom Stencil projects.

---

## Project Setup

```bash
# New Stencil component library (recommended for custom component libs)
npm init stencil

# With Ionic core as a dependency
npm install @ionic/core

# SCSS support via @stencil/sass (preferred over plain CSS)
npm install --save-dev @stencil/sass@3.2.3

# Testing — jest-stencil-runner for Jest v30+
npm install --save-dev jest-stencil-runner
```

**stencil.config.ts** minimal setup with SCSS:
```ts
import { Config } from '@stencil/core';
import { sass } from '@stencil/sass';

export const config: Config = {
  namespace: 'my-app',
  plugins: [
    sass(), // enables .scss for all styleUrl entries
  ],
  globalStyle: 'src/theme/space/index.scss', // theme folder: name reflects the color palette
  outputTargets: [
    { type: 'dist' },
    { type: 'dist-custom-elements' },
    { type: 'www', serviceWorker: null }
  ]
};
```

**jest.config.js:**
```js
const { createJestStencilPreset } = require('jest-stencil-runner/preset');

module.exports = createJestStencilPreset({
  rootDir: __dirname,
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
  ],
  testMatch: [
    '**/tests/**/*.spec.(ts|tsx)',
  ],
});
```

**tsconfig.json** — add types:
```json
{
  "compilerOptions": {
    "types": ["jest", "jest-stencil-runner"]
  }
}
```

**package.json** scripts:
```json
{
  "scripts": {
    "test": "jest",
    "test.watch": "jest --watchAll",
    "test.coverage": "jest --coverage"
  }
}
```

To load Ionic in an app/HTML entry point:
```ts
import { defineCustomElements } from '@ionic/core/loader';
defineCustomElements(window);
```

---

## Folder Structure

Components are grouped **by topic** — within each topic by type. Everything for a feature lives in one place.

```
src/
├── chat/
│   ├── pages/
│   │   ├── chat-overview/
│   │   │   ├── chat-overview.tsx
│   │   │   └── chat-overview.scss
│   │   └── chat-conversation/
│   │       ├── chat-conversation.tsx
│   │       └── chat-conversation.scss
│   ├── components/
│   │   ├── chat-bubble/
│   │   │   ├── chat-bubble.tsx
│   │   │   └── chat-bubble.scss
│   │   └── chat-input/
│   │       ├── chat-input.tsx
│   │       └── chat-input.scss
│   ├── modals/
│   │   └── chat-detail-modal/
│   │       ├── chat-detail-modal.tsx
│   │       └── chat-detail-modal.scss
│   ├── services/
│   │   └── chat.service.ts     ← chat domain only
│   ├── types/
│   │   ├── chat.types.ts
│   │   └── chat.enums.ts
│   └── tests/
│       ├── chat-bubble.spec.ts
│       └── chat.service.spec.ts
│
├── user/
│   ├── pages/
│   │   └── user-profile/
│   │       ├── user-profile.tsx
│   │       └── user-profile.scss
│   ├── components/
│   │   ├── user-avatar/
│   │   │   ├── user-avatar.tsx
│   │   │   └── user-avatar.scss
│   │   └── user-badge/
│   │       ├── user-badge.tsx
│   │       └── user-badge.scss
│   ├── modals/
│   │   └── user-edit-modal/
│   │       ├── user-edit-modal.tsx
│   │       └── user-edit-modal.scss
│   ├── services/
│   │   └── user.service.ts
│   ├── types/
│   │   ├── user.types.ts
│   │   └── user.enums.ts
│   └── tests/
│       ├── user-avatar.spec.ts
│       └── user.service.spec.ts
│
├── shared/              ← components without a clear topic
│   └── components/
│       └── loading-spinner/
│           ├── loading-spinner.tsx
│           └── loading-spinner.scss
│
├── guards/              ← global route guards
│   ├── auth.guard.ts
│   └── admin.guard.ts
│
├── services/            ← only cross-cutting global services
│   ├── auth.service.ts  ← tokens, roles, refresh — used by guards & all topics
│   └── api.service.ts   ← base HTTP client — used by all topic services
│
└── theme/
    └── space/
        ├── variables.scss
        ├── components.scss
        └── index.scss
```

### Rules

- **Topic first** — everything for a feature lives in one place (`chat/`, `user/`, ...)
- **Type as subfolder** — `pages/`, `components/`, `modals/`, `services/`, `types/` within the topic
- **Each component gets its own folder** — tag name = folder name = file name
- **`pages/`** — components referenced directly in `ion-router`
- **`modals/`** — components opened exclusively via `modalController.create()`
- **`components/`** — reusable building blocks without a direct route
- **`services/`** in topic — responsible for that topic only, uses global services as base
- **`types/`** — interfaces & types in `xxx.types.ts`, enums separately in `xxx.enums.ts`
- **`shared/`** — for anything without a clear topic (spinners, generic UI elements)
- **`tests/`** in topic — `.spec.ts` files for components and services of that topic
- **Never define interfaces directly in component files** — always extract to `types/`



---



> **Rule:** New components start with no custom styles. Use standard Ionic components first — layout and appearance are controlled centrally through a theme file.

### Why?

- Ionic components already ship with a consistent design — no need to reinvent the wheel
- CSS variables cascade through the entire Shadow DOM → one central value changes everything
- An empty `styleUrl` keeps the component free of overrides that conflict with the theme later
- New components stay lean and easy to maintain

### New component structure — create `styleUrl` but leave it empty

The style file is **always** created (clean structure, clear place for future adjustments), but starts empty. All visual defaults come from the theme.

```ts
import { Component, ComponentInterface, Prop, h } from '@stencil/core';

// ✅ styleUrl present but empty — styling comes from the theme
@Component({
  tag: 'my-user-card',
  styleUrl: 'my-user-card.scss', // create file, leave empty for now
  shadow: false, // Light DOM so theme variables apply
})
export class MyUserCard implements ComponentInterface {
  @Prop() name: string;
  @Prop() subtitle: string;

  render() {
    return (
      <ion-card>
        <ion-card-header>
          <ion-card-title>{this.name}</ion-card-title>
          <ion-card-subtitle>{this.subtitle}</ion-card-subtitle>
        </ion-card-header>
        <ion-card-content>
          <slot />
        </ion-card-content>
      </ion-card>
    );
  }
}
```

### Theme file — central place for all visuals

Themes live in a named subfolder — the name reflects the color palette (e.g. `space` for dark purple, `sand` for warm earth tones). Each theme has two files:

```
src/theme/
└── space/
    ├── variables.scss    ← CSS custom properties & brand colors
    └── components.scss   ← all ion-* overrides
```

**`src/theme/space/variables.scss`** — variables only, no selectors:
```scss
// ============================================
//   src/theme/space/variables.scss
//   Brand colors & Ionic CSS custom properties
// ============================================

// SCSS variables for reuse within the theme
$primary:     #7c4dff;
$primary-rgb: 124, 77, 255;
$bg:          #0d0d1a;
$surface:     #1a1a35;
$surface-2:   #12122a;
$text:        #f0f0f0;

:root {
  --ion-color-primary:          #{$primary};
  --ion-color-primary-rgb:      #{$primary-rgb};
  --ion-color-primary-contrast: #ffffff;
  --ion-color-primary-shade:    #6d44e0;
  --ion-color-primary-tint:     #895fff;

  --ion-background-color:       #{$bg};
  --ion-text-color:             #{$text};
  --ion-toolbar-background:     #{$surface-2};
  --ion-toolbar-color:          #{$text};
  --ion-item-background:        #{$surface};
  --ion-card-background:        #{$surface};
}
```

**`src/theme/space/components.scss`** — all ion-* overrides in one place:
```scss
// ============================================
//   src/theme/space/components.scss
//   Ionic component overrides for the space theme
// ============================================
@import './variables';

ion-button {
  --border-radius: 8px;
}

ion-card {
  border-radius: 12px;
}

ion-input, ion-textarea {
  --background: rgba(255, 255, 255, 0.05);
  --padding-start: 12px;
}

ion-item {
  --border-color: rgba(255, 255, 255, 0.08);
}
```

Include both files in `stencil.config.ts` as `globalStyle` — via an index file:

```
src/theme/
└── space/
    ├── variables.scss
    ├── components.scss
    └── index.scss        ← imports both
```

```scss
// src/theme/space/index.scss
@import './variables';
@import './components';
```

```ts
// stencil.config.ts
globalStyle: 'src/theme/space/index.scss',
```

### Workflow for new components

```
1. Write new component
   → Standard ion-* components only
   → create styleUrl (leave empty — room for later)

2. Adjust layout & appearance?
   → Always via src/theme/variables.scss
   → Set CSS variable — applies globally to ALL instances of that tag
   → Override Ionic components directly (ion-button, ion-card, ...)

3. Only when a truly component-specific layout is needed
   → fill the component's styleUrl and style only what's necessary
```

> ❌ No `style={{'--background': '...'}}` directly in JSX — hard to maintain and bypasses the central theme.

---

## StencilJS Decorators Reference

### @Component — Component Metadata
```ts
import { Component, ComponentInterface, h } from '@stencil/core';

@Component({
  tag: 'my-card',           // required — must contain a dash
  styleUrl: 'my-card.scss', // or styleUrls: { ios: '....ios.scss', md: '....md.scss' }
  shadow: false,            // true = Shadow DOM | false = Light DOM (default for this project)
  // scoped: true,          // alternative to shadow: Scoped encapsulation (auto-prefixed CSS)
                             // NOTE: shadow and scoped are mutually exclusive — pick one
})
export class MyCard implements ComponentInterface {
  render() {
    return <ion-card><slot /></ion-card>;
  }
}
```
> **CSS wrapper rule:**
> - `shadow: false` (Light DOM) → use tag name as wrapper in `.scss`: `my-card { ... }`
> - `scoped: true` or `shadow: true` → use `:host { ... }`
>
> → Details & examples in `references/shadow-dom.md`

---

### @Prop — Public API / Attributes
```ts
@Prop() label: string = 'Click me';
@Prop({ mutable: true }) isOpen: boolean = false;
@Prop({ reflect: true }) color: string; // reflects to DOM attribute
```
- Props are **immutable by default** — set `mutable: true` to change from inside
- Use `reflect: true` when the attribute must stay in sync with the DOM (e.g. for CSS selectors)
- camelCase prop → kebab-case HTML attribute automatically (`myProp` → `my-prop`)

---

### @State — Internal Reactive State
```ts
@State() count: number = 0;
@State() items: string[] = [];

// ✅ Triggers re-render (new reference)
this.items = [...this.items, 'new item'];

// ❌ Does NOT trigger re-render (mutation in place)
this.items.push('new item');
```

---

### @Watch — Reactivity on Prop/State Changes
```ts
@Prop() value: string;

@Watch('value')
onValueChange(newVal: string, oldVal: string) {
  // fires on every change EXCEPT initial render
  // use { immediate: true } to also fire on first load:
}

@Watch('value', { immediate: true })
onValueChangeImmediate(newVal: string) { ... }
```

---

### @Event + EventEmitter — Emit Custom Events
```ts
import { Event, EventEmitter } from '@stencil/core';

@Event() myChange: EventEmitter<string>;

handleClick() {
  this.myChange.emit('hello');
}
```
Options: `@Event({ eventName: 'customName', bubbles: true, composed: true, cancelable: false })`
> `composed: false` = event stays within Shadow DOM, doesn't reach outer DOM

---

### @Listen — Listen to DOM Events
```ts
import { Listen } from '@stencil/core';

@Listen('click')
handleClick(e: MouseEvent) { ... }

// Listen on window/document/body:
@Listen('scroll', { target: 'window', passive: true })
onScroll() { ... }

// Listen for a custom event from a child:
@Listen('myChange')
onMyChange(e: CustomEvent<string>) { ... }
```

---

### @Element — Reference to Host Element
```ts
import { Element } from '@stencil/core';

@Element() el: HTMLElement;

componentDidLoad() {
  const input = this.el.querySelector('ion-input');
}
```

---

### @Method — Public Methods
```ts
import { Method } from '@stencil/core';

@Method()
async open(): Promise<void> {
  this.isOpen = true;
}
```
> All `@Method` functions **must** return a Promise. Use `async` or wrap manually.

---

## Lifecycle Hooks

| Hook | When |
|------|------|
| `connectedCallback()` | Component added to DOM |
| `componentWillLoad()` | Before first render — best place for async data fetch |
| `componentDidLoad()` | After first render — DOM is available |
| `componentWillUpdate()` | Before re-render (not called on first render) |
| `componentDidUpdate()` | After re-render |
| `componentDidRender()` | After every render (first + updates) |
| `disconnectedCallback()` | Component removed from DOM |

```ts
async componentWillLoad() {
  this.data = await fetchSomething();
}

componentDidLoad() {
  // safe to query this.el here
}
```

---

## Using Ionic Components in Stencil

> **No manual `<ion-page>`** — `ion-router` automatically wraps every routed component in an `ion-page` element. An additional `<ion-page>` in `render()` causes double wrapping and breaks layout (scrolling, safe-area insets).

```tsx
import { Component, ComponentInterface, h } from '@stencil/core';

@Component({ tag: 'my-page', styleUrl: 'my-page.scss', shadow: false })
export class MyPage implements ComponentInterface {
  render() {
    return [
      <ion-header>
        <ion-toolbar>
          <ion-title>My Page</ion-title>
        </ion-toolbar>
      </ion-header>,
      <ion-content>
        <ion-button color="primary" onClick={() => this.doSomething()}>
          Click
        </ion-button>
      </ion-content>
    ];
  }
}
```

> **Tip:** Routed pages return an array (fragment) — `ion-header` + `ion-content` side by side, without a wrapper element. `ion-router` handles the rest.

**Important:** If your custom component uses `shadow: true`, Ionic components inside it won't receive global CSS variables unless you pass them through. Prefer `shadow: false` or `scoped: true` for page-level components.

---

## Styling Ionic Components

Read `references/shadow-dom.md` for full details. Quick rules:

| Component type | How to style internal elements |
|---|---|
| **Light DOM** (no badge) | Direct CSS selectors |
| **Scoped** (orange badge) | CSS vars + global styles; controller `cssClass` (Alert, Toast, Loading, Action Sheet) |
| **Shadow** (blue badge) | CSS variables + `::part()` only — incl. **Modal** & **Popover** |

> **Modal/Popover note:** `ion-modal` is Shadow DOM. Style via `::part()` + CSS vars. For an inline
> `<ion-modal>` use `class="…"` (not `cssClass="…"` — that's an `@internal` JSX attribute that can
> break on an `@ionic/core` bump); `cssClass` is only a `modalController.create()` option.
> See `references/shadow-dom.md` → "Shadow Overlays".

```css
/* CSS variable (works for all types) */
ion-button {
  --background: hotpink;
  --color: white;
}

/* ::part() for Shadow components */
ion-button::part(native) {
  border-radius: 12px;
  border: 2px solid white;
}
```

---

## Recommended Class Structure (Ionic Style Guide)

```ts
import { Component, ComponentInterface, Element, Event, EventEmitter, Listen, Method, Prop, State, Watch, h } from '@stencil/core';

@Component({ tag: 'my-component', styleUrl: 'my-component.scss', shadow: false })
export class MyComponent implements ComponentInterface {
  // 1. Own (private) properties
  private timer: ReturnType<typeof setTimeout>;

  // 2. @Element
  @Element() el: HTMLElement;

  // 3. @State
  @State() isActive: boolean = false;

  // 4. @Prop (+ @Watch directly after the Prop it watches)
  @Prop() label: string;
  @Watch('label')
  onLabelChange(newVal: string) { ... }

  // 5. @Event
  @Event() myEvent: EventEmitter<void>;

  // 6. Lifecycle hooks (in natural order)
  componentWillLoad() { ... }
  componentDidLoad() { ... }

  // 7. @Listen
  @Listen('click')
  onClick() { ... }

  // 8. @Method
  @Method()
  async doSomething(): Promise<void> { ... }

  // 9. Private helpers
  private helperMethod() { ... }

  // 10. render()
  render() { return <div />; }
}
```

---

## Common Patterns

> **Stencil JSX vs. React JSX:** Always use `class`, never `className` — Stencil follows the HTML standard.
> ```tsx
> // ✅ correct
> <div class="wrapper">
> // ❌ wrong — this is React
> <div className="wrapper">
> ```

### Slots (Light DOM content projection)
```tsx
// Definition
render() {
  return (
    <div class="wrapper">
      <slot name="header" />   {/* named slot */}
      <slot />                  {/* default slot */}
    </div>
  );
}

// Usage
<my-component>
  <span slot="header">Title</span>
  <p>Body content</p>
</my-component>
```

### Two-way Binding Pattern
```ts
@Prop({ mutable: true }) value: string;
@Event() valueChange: EventEmitter<string>;

onInput(e: Event) {
  const newVal = (e.target as HTMLInputElement).value;
  this.value = newVal;
  this.valueChange.emit(newVal);
}
```

### Conditional Rendering
```tsx
render() {
  return (
    <div>
      {this.isLoading ? <ion-spinner /> : <ion-list>...</ion-list>}
    </div>
  );
}
```

---

## Production Performance

Two cold-load optimizations that compound and are config-only:

- **Inline `<ion-icon>` SVGs via `addIcons()`** — eliminates ~50–80 per-icon `/svg/<name>.svg` HTTP fetches on first paint. Add to your `globalScript` entry. Adds ~50–70 kB gzipped to the bundle.
- **Manual `bundles:` config in `stencil.config.ts`** — groups your `@Component` tags (and the Ionic `ion-*` tags you use) into ~10 feature chunks instead of ~150 per-component lazy chunks. Stencil's auto lazy-load still applies; bundles only forces the listed tags into one entry.

Use both as soon as the app's DevTools Network tab on cold load shows hundreds of requests with `time-outline.svg`, `mail-outline.svg`, … or `p-XYZ.entry.js`, `p-ABC.entry.js`, … patterns.

→ Full setup, drift-audit greps, grouping rules + rationale, real-world numbers in `references/performance.md`.

---

## Further Reference

- `references/shadow-dom.md` — Shadow DOM types, CSS variables, ::part() styling patterns
- `references/ionic-components.md` — Common Ionic component usage patterns & gotchas
- `references/services.md` — service pattern, singleton factory, AuthService method reference
- `references/testing.md` — jest-stencil-runner setup, newSpecPage(), events, mocking
- `references/performance.md` — `addIcons()` inlining + `bundles:` config; cold-load chunk-count optimization
