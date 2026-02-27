# OpenClaw Internal Sub-application Architecture

This document proposes a modular architecture for the OpenClaw UI to support independent feature development, code splitting, and state isolation.

---

## 1. Module Interface Contract

All sub-applications must implement the `SubApp` interface.

```typescript
import { TemplateResult } from 'lit';
import { IconName } from './icons.js';
import { AppViewState } from './app-view-state.ts';

export interface SubApp {
  id: string;
  label: string;
  icon: IconName;
  path: string;
  group?: 'Chat' | 'Control' | 'Agent' | 'Settings';
  render: (state: AppViewState) => TemplateResult;
  onMount?: () => void;
  onUnmount?: () => void;
}

export interface LazySubApp extends Omit<SubApp, 'render'> {
  load: () => Promise<{ default: SubApp }>;
}
```

---

## 2. Registration and Lazy Loading System

The `SubAppRegistry` manages the lifecycle and loading of sub-apps.

```typescript
export class SubAppRegistry {
  private apps = new Map<string, SubApp | LazySubApp>();
  private loadedApps = new Map<string, SubApp>();

  register(app: SubApp | LazySubApp) {
    this.apps.set(app.id, app);
  }

  async getOrLoad(id: string): Promise<SubApp | undefined> {
    if (this.loadedApps.has(id)) return this.loadedApps.get(id);
    const entry = this.apps.get(id);
    if (!entry) return undefined;

    if ('render' in entry) {
      this.loadedApps.set(id, entry);
      return entry;
    } else {
      const module = await entry.load();
      this.loadedApps.set(id, module.default);
      return module.default;
    }
  }

  getAllMetadata() {
    return Array.from(this.apps.values()).map(({ render: _, load: __, ...meta }) => meta);
  }
}

export const subAppRegistry = new SubAppRegistry();
```

---

## 3. Example Implementation: Settings Sub-app

### Step 1: Define the Sub-app (`packages/feature-settings/index.ts`)

```typescript
import { html } from 'lit';
import { SubApp } from '../../ui/src/ui/navigation.ts';

const SettingsApp: SubApp = {
  id: 'settings',
  label: 'Settings',
  icon: 'settings',
  path: '/settings',
  group: 'Settings',
  render: (state) => html`<settings-view .globalState=${state}></settings-view>`,
};

export default SettingsApp;
```

### Step 2: Register in the Shell (`ui/src/ui/app.ts`)

```typescript
import { subAppRegistry } from './navigation.ts';

subAppRegistry.register({
  id: 'settings',
  label: 'Settings',
  icon: 'settings',
  path: '/settings',
  group: 'Settings',
  load: () => import('@openclaw/feature-settings')
});
```

### Step 3: Handle Routing in Shell (`ui/src/ui/app-render.ts`)

```typescript
// Inside renderApp
const activeSubApp = await subAppRegistry.getOrLoad(state.tab);

return html`
  <main class="content">
    ${activeSubApp ? activeSubApp.render(state) : nothing}
  </main>
`;
```

---

## 4. State Isolation

Sub-apps should use Lit Context to manage internal state without polluting the global `AppViewState`.

```typescript
// packages/feature-settings/state.ts
import { createContext } from '@lit/context';

export interface SettingsState {
  advancedMode: boolean;
}

export const settingsContext = createContext<SettingsState>('settings-state');
```

```typescript
// packages/feature-settings/settings-view.ts
@customElement('settings-view')
export class SettingsView extends LitElement {
  @provide({ context: settingsContext })
  @state() private state: SettingsState = { advancedMode: false };

  render() {
    return html`
      <settings-toggle .enabled=${this.state.advancedMode}></settings-toggle>
    `;
  }
}
```
