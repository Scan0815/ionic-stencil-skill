# Common Ionic Components — Patterns & Gotchas

## Navigation Structure

```tsx
// Standard page layout
render() {
  return (
    <ion-page>
      <ion-header>
        <ion-toolbar>
          <ion-buttons slot="start">
            <ion-back-button defaultHref="/home" />
          </ion-buttons>
          <ion-title>Page Title</ion-title>
          <ion-buttons slot="end">
            <ion-button onClick={() => this.openMenu()}>
              <ion-icon slot="icon-only" name="menu" />
            </ion-button>
          </ion-buttons>
        </ion-toolbar>
      </ion-header>
      <ion-content fullscreen>
        <ion-refresher slot="fixed" onIonRefresh={e => this.doRefresh(e)}>
          <ion-refresher-content />
        </ion-refresher>
        {/* content here */}
      </ion-content>
    </ion-page>
  );
}
```

---

## Lists & Items

```tsx
<ion-list>
  <ion-item>
    <ion-label>Default item</ion-label>
  </ion-item>

  <ion-item button onClick={() => this.navigate()}>
    <ion-icon name="arrow-forward" slot="end" />
    <ion-label>Clickable item</ion-label>
  </ion-item>

  <ion-item lines="none">
    <ion-thumbnail slot="start">
      <img src="avatar.png" />
    </ion-thumbnail>
    <ion-label>
      <h2>Title</h2>
      <p>Subtitle</p>
    </ion-label>
  </ion-item>
</ion-list>
```

**Gotcha:** `ion-item` is a **Shadow** component — use CSS vars or `::part(native)` to style the inner `<div>`.

---

## Radio Buttons

### Standard radio list — label as child, radio on the left
```tsx
@State() selectedReason: string;

private reasons = [
  { code: 'a', key: 'reason.a' },
  { code: 'b', key: 'reason.b' },
  { code: 'c', key: 'reason.c' },
];

render() {
  return (
    <ion-list>
      <ion-radio-group
        value={this.selectedReason}
        onIonChange={(ev: CustomEvent) => this.selectedReason = ev.detail.value}
      >
        {this.reasons.map(reason => (
          <ion-item lines="full" key={reason.code}>
            <ion-radio justify="start" label-placement="end" value={reason.code}>
              {reason.label}
            </ion-radio>
          </ion-item>
        ))}
      </ion-radio-group>
    </ion-list>
  );
}
```

> - Label goes as **direct child of `ion-radio`** — no separate `ion-label`
> - `justify="start"` → radio button on the left
> - `label-placement="end"` → label to the right of the button
> - `onIonChange` is on the **group**, not on the individual `ion-radio`
> - `lines="full"` on `ion-item` for a full-width divider

### Simple static list
```tsx
<ion-list>
  <ion-radio-group
    value={this.plan}
    onIonChange={(ev: CustomEvent) => this.plan = ev.detail.value}
  >
    <ion-item lines="full">
      <ion-radio justify="start" label-placement="end" value="free">Free</ion-radio>
    </ion-item>
    <ion-item lines="full">
      <ion-radio justify="start" label-placement="end" value="pro">Pro</ion-radio>
    </ion-item>
    <ion-item lines="full">
      <ion-radio justify="start" label-placement="end" value="enterprise" disabled>Enterprise</ion-radio>
    </ion-item>
  </ion-radio-group>
</ion-list>
```

### Styling via theme (`components.scss`)
```scss
// ion-radio is a Shadow component → CSS variables & ::part()
ion-radio {
  --color: rgba(255, 255, 255, 0.4);
  --color-checked: var(--ion-color-primary);
}
```

---

## Checkboxes

### Standard checkbox list — label as child, checkbox on the left
```tsx
<ion-list>
  <ion-item>
    <ion-checkbox justify="start" label-placement="end">
      Option A
    </ion-checkbox>
  </ion-item>
  <ion-item>
    <ion-checkbox justify="start" label-placement="end">
      Option B
    </ion-checkbox>
  </ion-item>
</ion-list>
```

### Dynamic list with state
```tsx
@State() selected: string[] = [];

private options = [
  { code: 'a', label: 'Option A' },
  { code: 'b', label: 'Option B' },
  { code: 'c', label: 'Option C' },
];

toggleOption(code: string, checked: boolean) {
  if (checked) {
    this.selected = [...this.selected, code];
  } else {
    this.selected = this.selected.filter(s => s !== code);
  }
}

render() {
  return (
    <ion-list>
      {this.options.map(option => (
        <ion-item lines="full" key={option.code}>
          <ion-checkbox
            justify="start"
            label-placement="end"
            checked={this.selected.includes(option.code)}
            onIonChange={(ev: CustomEvent) => this.toggleOption(option.code, ev.detail.checked)}
          >
            {option.label}
          </ion-checkbox>
        </ion-item>
      ))}
    </ion-list>
  );
}
```

> - Label goes as **direct child of `ion-checkbox`** — no separate `ion-label`
> - `justify="start"` → checkbox on the left
> - `label-placement="end"` → label to the right of the checkbox
> - `ev.detail.checked` (boolean) instead of `ev.detail.value`
> - state as array — always assign a new array, never `.push()`

### Styling via theme (`components.scss`)
```scss
ion-checkbox {
  --checkbox-background-checked: var(--ion-color-primary);
  --border-color: rgba(255, 255, 255, 0.4);
  --border-color-checked: var(--ion-color-primary);
  --checkmark-color: white;
}
```

---

## Forms & Inputs

```tsx
<ion-list>
  <ion-item>
    <ion-label position="floating">Email</ion-label>
    <ion-input
      type="email"
      value={this.email}
      onIonChange={e => this.email = e.detail.value}
    />
  </ion-item>

  <ion-item>
    <ion-label>Notifications</ion-label>
    <ion-toggle
      checked={this.notifications}
      onIonChange={e => this.notifications = e.detail.checked}
    />
  </ion-item>

  <ion-item>
    <ion-label>Plan</ion-label>
    <ion-select
      value={this.plan}
      onIonChange={e => this.plan = e.detail.value}
    >
      <ion-select-option value="free">Free</ion-select-option>
      <ion-select-option value="pro">Pro</ion-select-option>
    </ion-select>
  </ion-item>
</ion-list>
```

**Gotcha:** Use `onIonChange` (not `onChange`) for Ionic form components. The event detail is in `e.detail.value`.

---

## Modals

Preferred pattern: `modalController` — opened from outside, modal component is self-contained.

### Caller component
```tsx
import { modalController } from '@ionic/core';

async openModal() {
  const modal = await modalController.create({
    component: 'my-detail-modal',
    componentProps: {
      title: 'Titel',
      itemId: '123',
    },
  });
  await modal.present();

  // wait for return value
  const { data, role } = await modal.onWillDismiss();
  if (role === 'confirm') {
    console.log('Returned:', data);
  }
}
```

### Modal component (`my-detail-modal.tsx`)

Standard structure: close button top right, footer with action buttons.

```tsx
import { Component, ComponentInterface, Element, Prop, h } from '@stencil/core';
import { modalController } from '@ionic/core';

@Component({
  tag: 'my-detail-modal',
  styleUrl: 'my-detail-modal.scss',
  shadow: false,
})
export class MyDetailModal implements ComponentInterface {

  @Element() el: HTMLElement;

  @Prop() title: string;
  @Prop() itemId: string;

  private dismiss(data?: any, role?: string) {
    modalController.dismiss(data, role);
  }

  render() {
    return (
      <ion-page>

        {/* Header with close button top right */}
        <ion-header>
          <ion-toolbar>
            <ion-title>{this.title}</ion-title>
            <ion-buttons slot="end">
              <ion-button onClick={() => this.dismiss(null, 'cancel')}>
                <ion-icon slot="icon-only" name="close" />
              </ion-button>
            </ion-buttons>
          </ion-toolbar>
        </ion-header>

        {/* Main content */}
        <ion-content>
          <p>Item ID: {this.itemId}</p>
        </ion-content>

        {/* Footer with action buttons */}
        <ion-footer>
          <ion-toolbar>
            <ion-buttons slot="start">
              <ion-button onClick={() => this.dismiss(null, 'cancel')}>
                Cancel
              </ion-button>
            </ion-buttons>
            <ion-buttons slot="end">
              <ion-button
                fill="solid"
                color="primary"
                onClick={() => this.dismiss({ itemId: this.itemId }, 'confirm')}
              >
                Confirm
              </ion-button>
            </ion-buttons>
          </ion-toolbar>
        </ion-footer>

      </ion-page>
    );
  }
}
```

> - always wrap modal component in `ion-page`
> - close button: `ion-buttons slot="end"` → `ion-button` → `ion-icon slot="icon-only" name="close"`
> - footer: `ion-footer` → `ion-toolbar` → `ion-buttons slot="start/end"`
> - `modalController.dismiss(data, role)` — `role` to distinguish confirm/cancel at the caller

> **Styling the modal:** `ion-modal` is Shadow DOM. Use CSS variables + `::part()` in global CSS —
> e.g. `ion-modal.my-modal { --border-radius: 16px; }` and `ion-modal.my-modal::part(content) { … }`.
> Pass the hook class via the controller (`modalController.create({ cssClass: 'my-modal' })`) or, for
> an inline `<ion-modal>`, via plain `class="my-modal"` — **not** `cssClass=` as a JSX attribute
> (that's an `@internal` Ionic attribute that can break on an `@ionic/core` bump).
> Full details + the pre-update build smoke test in `references/shadow-dom.md` → "Shadow Overlays".



---

## Alerts & Toast

```ts
import { alertController, toastController } from '@ionic/core';

// Alert
async showAlert() {
  const alert = await alertController.create({
    header: 'Confirm',
    message: 'Are you sure?',
    cssClass: 'my-alert',
    buttons: [
      { text: 'Cancel', role: 'cancel' },
      {
        text: 'Confirm',
        role: 'confirm',
        handler: () => this.doAction()
      }
    ]
  });
  await alert.present();
}

// Toast
async showToast(msg: string) {
  const toast = await toastController.create({
    message: msg,
    duration: 2000,
    position: 'bottom',
    color: 'success',
    cssClass: 'my-toast',
  });
  await toast.present();
}
```

---

## Loading Spinner

```ts
import { loadingController } from '@ionic/core';

async withLoading(fn: () => Promise<void>) {
  const loading = await loadingController.create({
    message: 'Loading...',
    spinner: 'crescent',
  });
  await loading.present();
  try {
    await fn();
  } finally {
    await loading.dismiss();
  }
}
```

---

## Icons

```tsx
// Built-in Ionicons (included with @ionic/core)
<ion-icon name="heart" />
<ion-icon name="heart-outline" />
<ion-icon name="heart-sharp" />  // Android
<ion-icon name="heart" ios="heart-outline" md="heart" /> // platform-specific

// Custom SVG icon
<ion-icon src="/assets/icons/my-icon.svg" />

// Sizes
<ion-icon name="star" size="small" />
<ion-icon name="star" size="large" />
```

---

## Ionic Router (in Stencil apps)

Ionic comes with its own router (`ion-router`) — use this instead of `@stencil/router`. It integrates natively with `ion-nav`, handles back-button behavior, and supports iOS swipe-back gestures automatically.

### Declarative setup (root component)
```tsx
// In root component render()
<ion-app>
  <ion-router useHash={false}>
    <ion-route url="/" component="app-home" />
    <ion-route url="/detail/:id" component="app-detail" />
    <ion-route url="/settings" component="app-settings" />

    {/* Redirect */}
    <ion-route-redirect from="/" to="/home" />

    {/* Nested routes */}
    <ion-route url="/tabs" component="app-tabs">
      <ion-route url="/tabs/feed" component="tab-feed" />
      <ion-route url="/tabs/profile" component="tab-profile" />
    </ion-route>
  </ion-router>
  <ion-nav />
</ion-app>
```

### Programmatic navigation — `ion-router-link` / `href`
```tsx
// Declarative link (preferred for simple navigation)
<ion-button href="/detail/42" routerDirection="forward">
  Open Detail
</ion-button>

<ion-item href="/settings" routerDirection="back">
  Settings
</ion-item>
```

### Programmatic Navigation — `ion-router` via DOM ref
```ts
import { Element } from '@stencil/core';

@Element() el: HTMLElement;

async navigate(id: string) {
  const router = document.querySelector('ion-router');
  await router.push(`/detail/${id}`, 'forward');
  // Directions: 'forward' | 'back' | 'root'
}
```

### Reading URL params in the target component
```ts
// ion-router passes params as @Prop() directly on the component
@Prop() match: { id: string }; // for route url="/detail/:id"

componentWillLoad() {
  console.log(this.match.id); // the :id from the URL
}
```

#### Tab-based Navigation
```tsx
// app-tabs.tsx
render() {
  return (
    <ion-tabs>
      <ion-tab tab="feed">
        <ion-nav />
      </ion-tab>
      <ion-tab tab="profile">
        <ion-nav />
      </ion-tab>

      <ion-tab-bar slot="bottom">
        <ion-tab-button tab="feed" href="/tabs/feed">
          <ion-icon name="home" />
          <ion-label>Feed</ion-label>
        </ion-tab-button>
        <ion-tab-button tab="profile" href="/tabs/profile">
          <ion-icon name="person" />
          <ion-label>Profile</ion-label>
        </ion-tab-button>
      </ion-tab-bar>
    </ion-tabs>
  );
}
```

**Gotcha:** Never use `@stencil/router` alongside `ion-router` — they conflict. Stick to `ion-router` for all navigation in Ionic projects.

---

## Platform Detection & Modes

```ts
import { isPlatform, getMode } from '@ionic/core';

// Platform checks
isPlatform('ios')     // true on iOS devices
isPlatform('android') // true on Android
isPlatform('mobile')  // true on any mobile
isPlatform('desktop') // true on desktop
isPlatform('pwa')     // true if running as PWA

// Current mode (ios | md)
const mode = getMode(); // 'ios' or 'md'
```

```tsx
// Force mode on a component
<ion-button mode="md">Always Material Design</ion-button>
```

---

## Guards

Guards live globally under `src/guards/` and are wired into `ion-route` via `beforeEnter`. Guards are **async** and return `true` (continue) or `{ redirect: '/path' }`. Services are retrieved via a `getXxxService()` factory.

### Example — auth guard (`src/guards/auth.guard.ts`)
```ts
import { getAuthService } from '../services/auth.service';

/**
 * Auth guard — checks if the user is logged in
 * Automatically tries to refresh the token if expired
 */
export const authGuard = async (): Promise<boolean | { redirect: string }> => {
  const authService = getAuthService();

  // check token — auto-refreshes if needed
  const token = await authService.ensureValidToken();

  if (!token) {
    return { redirect: '/login' };
  }

  return true;
};
```

### Example — admin guard (`src/guards/admin.guard.ts`)
```ts
import { getAuthService } from '../services/auth.service';

/**
 * Admin guard — checks login + admin role
 */
export const adminGuard = async (): Promise<boolean | { redirect: string }> => {
  const authService = getAuthService();

  const token = await authService.ensureValidToken();

  if (!token) {
    return { redirect: '/login' };
  }

  if (!authService.isAdmin()) {
    return { redirect: '/login' };
  }

  return true;
};
```

### Wire guard into route
```tsx
import { authGuard } from '../../guards/auth.guard';
import { adminGuard } from '../../guards/admin.guard';

<ion-router useHash={false}>
  <ion-route url="/login" component="login-page" />

  <ion-route url="/chat" component="chat-overview" beforeEnter={authGuard} />
  <ion-route url="/admin" component="admin-page" beforeEnter={adminGuard} />
</ion-router>
```

> - return `true` → route loads
> - return `{ redirect: '/path' }` → redirect
> - always get services via `getXxxService()` factory, never instantiate directly
> - JSDoc comment per guard — what is checked, what happens on failure



---

## Common Gotchas

| Issue | Fix |
|-------|-----|
| Ionic events not firing | Use `onIonChange` / `onIonInput`, not `onChange` |
| Modal content not rendering | Ensure component is registered in stencil.config.ts |
| Shadow DOM CSS not applying | Use `--css-variable` or `::part()`, never internal class selectors |
| Overlay styles only work globally | Put `.my-alert` styles in `global.css`, not page CSS |
| `this.el` is undefined | Access inside `componentDidLoad()`, not `componentWillLoad()` |
| State array change not re-rendering | Assign new array: `this.items = [...this.items, newItem]` |
| `@Method` fails with error | Must return a Promise — use `async` |
