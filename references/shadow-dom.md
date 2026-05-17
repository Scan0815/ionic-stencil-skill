# Shadow DOM, CSS Variables & ::part() in Ionic

## Component Encapsulation Types

Ionic components fall into three categories. You can identify which one a component is by its badge in the Ionic docs.

| Type | Badge | Style access |
|------|-------|--------------|
| **Light DOM** | None | Direct CSS, any selector works |
| **Scoped** | 🟠 Orange | CSS vars + global styles with `cssClass` |
| **Shadow** | 🔵 Blue | CSS vars + `::part()` only |

> Never try `ion-button .button-native { ... }` — Shadow DOM blocks it completely.

---

## CSS Custom Properties (Variables)

All Ionic components expose CSS variables. This is the **primary** way to customize them.

```css
/* Set on the component itself */
ion-button {
  --background: #7c4dff;
  --background-hover: #651fff;
  --background-activated: #6200ea;
  --color: white;
  --border-radius: 8px;
  --padding-start: 20px;
  --padding-end: 20px;
}

ion-input {
  --background: transparent;
  --color: var(--ion-color-primary);
  --placeholder-color: #999;
  --placeholder-opacity: 0.8;
  --padding-start: 12px;
}

ion-item {
  --background: #1a1a2e;
  --color: white;
  --border-color: rgba(255,255,255,0.1);
  --inner-padding-end: 0;
}
```

**Tip:** Open DevTools → inspect the Ionic element → filter styles by `--` to see all variables in effect.

---

## CSS Shadow Parts (::part())

For Shadow DOM components, `::part()` lets you reach inside the shadow root.

```css
/* ion-button exposes "native" (the underlying <button> tag) */
ion-button::part(native) {
  border: 2px solid currentColor;
  border-radius: 50px;
}

/* ion-select exposes "placeholder", "icon", "text" */
ion-select::part(placeholder) {
  color: rgba(255,255,255,0.5);
  font-style: italic;
}

/* Combine with pseudo-elements */
ion-select::part(placeholder)::first-letter {
  font-size: 1.2em;
}

/* Combine with state classes on the host */
ion-toggle.toggle-checked::part(track) {
  background: rgba(var(--ion-color-success-rgb), 0.5);
}
ion-toggle::part(handle) {
  background: white;
}
```

Find all parts for a component under "CSS Shadow Parts" in the Ionic API docs for that component.

---

## Scoped Components — use cssClass

For overlay/input components (Alert, Modal, Popover, etc.) that are **Scoped**, use `cssClass`:

```ts
// In your component code
const alert = await alertController.create({
  cssClass: 'my-custom-alert',
  header: 'Warning',
  message: 'Are you sure?',
  buttons: ['OK']
});
```

```css
/* In global.css (NOT in a component's scoped styles) */
.my-custom-alert .alert-wrapper {
  background: #1a1a2e;
  border-radius: 16px;
  border: 1px solid rgba(255,255,255,0.1);
}
.my-custom-alert .alert-title {
  color: white;
}
```

> Overlay components are appended to `<ion-app>`, outside your page's scoped styles. Always put these overrides in **global CSS**.

---

## CSS Scoping by Encapsulation Type

This is one of the most important rules — which selector to use as wrapper depends directly on the `@Component` decorator:

| Encapsulation | Wrapper in `.scss` file |
|---|---|
| `shadow: false` (Light DOM) | **Component tag name** e.g. `my-user-card { ... }` |
| `scoped: true` | **`:host`** |
| `shadow: true` | **`:host`** |

### Light DOM — tag name as wrapper (`shadow: false`)

```scss
// my-user-card.scss
// shadow: false → no :host, use the tag name as wrapper
my-user-card {
  display: block;

  ion-card {
    margin: 0;
  }

  .card-footer {
    padding: 8px;
  }
}
```

> `:host` has **no effect** here — Light DOM has no Shadow Root.

### Scoped — `:host` as wrapper (`scoped: true`)

```scss
// my-user-card.scss
// scoped: true → use :host
:host {
  display: block;

  ion-card {
    margin: 0;
  }
}

:host(.active) {
  border: 2px solid var(--ion-color-primary);
}
```

### Shadow DOM — `:host` as wrapper (`shadow: true`)

```scss
// my-user-card.scss
// shadow: true → use :host
:host {
  display: block;
  background: var(--card-bg, transparent);
}

:host(.active) {
  border: 2px solid var(--ion-color-primary);
}

:host-context(ion-content) {
  padding: 16px;
}
```

---

## Passing CSS Variables through Shadow DOM

CSS variables pierce the Shadow DOM — this is the only way to style inside a `shadow: true` component from outside:

```scss
// from outside (theme/space/components.scss)
my-card {
  --card-bg: #1a1a2e;
  --card-color: white;
}
```
```scss
// inside the component (shadow: true → use :host)
:host {
  background: var(--card-bg, white);
  color: var(--card-color, black);
}
```

---

## Styling Inside Your Own Shadow Component

### Option 1: prefer `shadow: false` for page-level components
```ts
@Component({ tag: 'my-page', shadow: false }) // Light DOM — global theme applies directly
@Component({ tag: 'my-card', scoped: true })  // Scoped — isolated but :host available
```

---

## Global Ionic CSS Variables

Core Ionic theme variables (set in `variables.scss`):

```css
:root {
  /* Brand colors */
  --ion-color-primary: #7c4dff;
  --ion-color-primary-rgb: 124, 77, 255;
  --ion-color-primary-contrast: #ffffff;
  --ion-color-primary-contrast-rgb: 255, 255, 255;
  --ion-color-primary-shade: #6d44e0;
  --ion-color-primary-tint: #895fff;

  /* App background + text */
  --ion-background-color: #0d0d1a;
  --ion-background-color-rgb: 13, 13, 26;
  --ion-text-color: #f0f0f0;
  --ion-text-color-rgb: 240, 240, 240;

  /* Toolbar */
  --ion-toolbar-background: #12122a;
  --ion-toolbar-color: #ffffff;

  /* Item */
  --ion-item-background: #1a1a35;
  --ion-item-color: #f0f0f0;
}
```

Use the **Ionic Color Generator** at `ionicframework.com/docs/theming/colors` to auto-generate all shades.
