# OpenClaw Modularization Plan

This plan outlines the transformation of OpenClaw from its current monolithic structure into a decoupled, pluggable architecture.

---

## 1. Modular Architecture Target

### **A. Core Engine (`@openclaw/core`)**
- **Responsibility:** AI reasoning, agent loop orchestration, session/memory persistence, and WebSocket protocol handling.
- **Independence:** Must be able to run without a UI or a specific messaging channel.
- **Boundaries:** Exposes the Gateway RPC protocol as a stable API.

### **B. UI Shell (`@openclaw/ui-shell`)**
- **Responsibility:** Main application layout, theme management, navigation routing, and WebSocket connection management.
- **Independence:** A thin frame that knows how to "mount" sub-apps but doesn't contain feature-specific logic.

### **C. Pluggable Sub-app Modules (`@openclaw/feature-*`)**
- **Examples:** `feature-chat`, `feature-agents`, `feature-usage`, `feature-config`.
- **Responsibility:** Self-contained UI views and their associated business logic (controllers).
- **Communication:** Interact with the Core Engine via the WebSocket SDK and with the UI Shell via a mounting API.

### **D. Independent Feature Packages (`@openclaw/*`)**
- **`@openclaw/protocol`:** Shared RPC frame definitions and Ajv schemas.
- **`@openclaw/ui-components`:** Design system (CSS variables), icons, and shared Lit components.
- **`@openclaw/plugin-sdk`:** Backend interfaces for third-party channel/tool development.

---

## 2. Folder Restructuring Proposal

```text
/
  apps/
    gateway/                 # Runnable Gateway server (bootstraps Core Engine)
    control-ui/              # Runnable Vite app (bootstraps UI Shell + Sub-apps)
    macos/                   # Native macOS companion app
    ios/                     # iOS node app
    android/                 # Android node app

  packages/
    core-engine/             # Agent loop, Pi integration, persistence
    protocol/                # Shared RPC schemas and types
    ui-shell/                # Navigation, Shell layout, WS Client SDK
    ui-components/           # CSS Design System, shared Lit elements
    plugin-sdk/              # Backend plugin interfaces (channel/tool)
    feature-chat/            # Chat sub-app (Lit components + controllers)
    feature-agents/          # Agents sub-app
    feature-usage/           # Usage/Analytics sub-app
    feature-nodes/           # Device/Node management sub-app
    utils/                   # Generic JS/TS utilities

  extensions/                # Built-in and 3rd party plugins (channels/tools)
```

---

## 3. Refactor Priority and Migration Steps

### **Phase 1: Foundation (High Priority)**
1.  **Extract Protocol:** Move `src/gateway/protocol` to `packages/protocol`. Ensure it has no dependencies on the Gateway implementation.
2.  **Extract UI Components:** Move `ui/src/styles/` and `ui/src/ui/components/` to `packages/ui-components`. Standardize the design system using CSS variables.
3.  **Monorepo Setup:** Configure `pnpm-workspace.yaml` to recognize the new package paths.

### **Phase 2: Core Decoupling**
1.  **Core Engine Extraction:** Move `src/agents`, `src/memory`, and session logic from `src/gateway` to `packages/core-engine`.
2.  **Backend Dependency Injection:** Refactor `src/gateway/server.impl.ts` to use a service registry. Replace direct imports with interface-based injection.

### **Phase 3: UI Shell Refactor**
1.  **Mounting API:** Define a standard interface for sub-apps in `ui/src/ui/app-view-state.ts`.
2.  **State Migration:** Move global connection state from `OpenClawApp` to a dedicated `ConnectionManager` in the UI Shell.

### **Phase 4: Feature Slicing (Incremental)**
1.  **Feature Extraction:** Move one view at a time (e.g., `ui/src/ui/views/usage.ts`) into its own package (`packages/feature-usage`).
2.  **Test Isolation:** Ensure each feature package has its own unit/browser tests that run independently of the main app.

---

## 4. Code Examples: Decoupling Strategy

### **Example 1: Decoupling UI State (Using Lit Context)**

**Before (Monolithic `app.ts`):**
```typescript
@customElement("openclaw-app")
export class OpenClawApp extends LitElement {
  @state() chatMessages: unknown[] = []; // In the God object
  // ... 100 other state properties
}
```

**After (Modular Context):**
```typescript
// packages/feature-chat/chat-context.ts
export const chatContext = createContext<ChatState>("chat-state");

// packages/feature-chat/chat-view.ts
@customElement("chat-view")
export class ChatView extends LitElement {
  @consume({ context: chatContext, subscribe: true })
  @state() state?: ChatState;

  render() {
    return html`<chat-list .messages=${this.state?.messages}></chat-list>`;
  }
}
```

### **Example 2: Decoupling Backend Services**

**Before (Sequential Bootstrapping):**
```typescript
// src/gateway/server.impl.ts
export async function startGatewayServer() {
  const cron = buildGatewayCronService(...);
  const channels = createChannelManager(...);
  // ... tight coupling ...
}
```

**After (Registry Pattern):**
```typescript
// packages/core-engine/registry.ts
export interface GatewayService {
  id: string;
  start(ctx: ServiceContext): Promise<void>;
  stop(): Promise<void>;
}

// apps/gateway/main.ts
const registry = new ServiceRegistry();
registry.register(new CronService());
registry.register(new ChannelService());
await registry.startAll();
```

---

## 5. Migration Checklist

- [ ] Initialize `packages/` directory with `package.json` for each module.
- [ ] Update `tsconfig.json` paths and `pnpm-workspace.yaml`.
- [ ] Extract `@openclaw/protocol` to break cyclic dependencies.
- [ ] Implement `SubApp` interface for UI views.
- [ ] Migrate `OpenClawApp` to a layout-only component.
- [ ] Extract one feature view per sprint until the legacy `ui/src` is empty.
