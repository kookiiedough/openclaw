# Implementation Details: UI Theme, Nav Rail, Sub-app Loader, and React Mounting

This document details the minimum set of files and exact modifications required to implement the requested UI enhancements.

---

## 1. Replace the Entire UI Theme

**Primary File:** `ui/src/styles/base.css`
**Modification:** Redefine the CSS variables in the `:root` and `:root[data-theme="light"]` selectors.

```css
/* Example: Replacing with a "Midnight Gold" theme */
:root {
  --bg: #0a0b10;
  --text: #e0e0e0;
  --accent: #d4af37; /* Gold */
  --border: #1c1d26;
  /* ... repeat for all variables in base.css ... */
}
```

**Secondary File:** `ui/src/ui/theme.ts`
**Modification:** (Optional) Update `resolveTheme` if adding more than just light/dark modes (e.g., "high-contrast").

---

## 2. Introduce a Persistent Left Navigation Rail

**Primary File:** `ui/src/styles/layout.css`
**Modification:** Update the `.shell` grid and `.nav` styles.

```css
.shell {
  /* Change nav width to rail width */
  --shell-nav-width: 64px;
  grid-template-columns: var(--shell-nav-width) 1fr;
  grid-template-areas: "nav content"; /* Move nav to the far left, span full height */
}

.nav {
  display: flex;
  flex-direction: column;
  align-items: center; /* Center icons */
  padding: 12px 0;
}

.nav-item__text {
  display: none; /* Hide text in the rail */
}
```

**Secondary File:** `ui/src/ui/app-render.ts`
**Modification:** Adjust the `renderApp` HTML structure to match the new grid areas (move `header.topbar` or integrate it into `main.content`).

---

## 3. Create a Dynamic Sub-app Loader

**Primary File:** `ui/src/ui/navigation.ts`
**Modification:** Replace static constants with a registry.

```typescript
export interface SubApp {
  id: string;
  label: string;
  icon: IconName;
  path: string;
  render: (state: AppViewState) => TemplateResult;
}

const registry = new Map<string, SubApp>();
export function registerSubApp(app: SubApp) { registry.set(app.id, app); }
```

**Secondary File:** `ui/src/ui/app-render.ts`
**Modification:** Refactor the main content block to use the registry.

```typescript
/* Inside renderApp */
const activeApp = registry.get(state.tab);
return html`
  <main class="content">
    ${activeApp ? activeApp.render(state) : nothing}
  </main>
`;
```

---

## 4. Mount Independent React Apps

**New File:** `ui/src/ui/components/ReactBridge.ts`
**Role:** A Lit wrapper that manages the React lifecycle.

```typescript
import { LitElement, html } from 'lit';
import { createRoot } from 'react-dom/client';

@customElement('react-bridge')
export class ReactBridge extends LitElement {
  private root: any;
  @property({ type: Object }) app: any; // The React Component

  firstUpdated() {
    this.root = createRoot(this.renderRoot.querySelector('#react-root')!);
    this.root.render(React.createElement(this.app));
  }

  render() { return html`<div id="react-root"></div>`; }
}
```

**Modification in `ui/src/ui/app-render.ts`:**
Use the `ReactBridge` within the sub-app loader's `render` function for any React-based sub-apps.

```typescript
registerSubApp({
  id: 'my-react-app',
  render: (state) => html`<react-bridge .app=${MyReactComponent}></react-bridge>`
});
```
