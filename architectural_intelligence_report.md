# Architectural Intelligence Report: OpenClaw

## SECTION 1: Architecture Map

### Project Structure Overview
- **Monorepo (pnpm):** Root contains the backend (Gateway), `ui/` contains the frontend (Control UI), `extensions/` contains channel/tool plugins, and `packages/` contains shared utilities.
- **Backend (Gateway):** Primary entrypoint is `src/entry.ts` (bootstrapped via `openclaw.mjs`). Core server implementation in `src/gateway/server.impl.ts`.
- **Frontend (Control UI):** Entrypoint is `ui/src/main.ts` which imports `ui/src/ui/app.ts`. Built with Lit and Vite.
- **Build System:**
    - Backend: `tsdown` (defined in `package.json`).
    - Frontend: `Vite` (defined in `ui/vite.config.ts`).
    - Mobile: Gradle for Android, Xcodegen/Xcode for iOS/macOS.

### Entrypoints
- **Frontend:** `ui/index.html` -> `ui/src/main.ts` -> `ui/src/ui/app.ts` (OpenClawApp custom element).
- **Backend:** `openclaw.mjs` -> `src/entry.ts` -> `src/cli/run-main.ts`.

### Key Systems
- **State Management:**
    - Frontend: Centralized in `OpenClawApp` (ui/src/ui/app.ts) using Lit `@state`. Logic is delegated to modular `app-*.ts` helpers.
    - Backend: In-memory state in `src/gateway/server-runtime-state.ts`.
- **Routing:**
    - Frontend: Centralized in `ui/src/ui/navigation.ts`, using `window.location.pathname` mapping to `Tab` names.
    - Backend: Centralized in `src/routing/resolve-route.ts` for mapping channel/peer identifiers to `agentId` and `sessionKey`.
- **API Layer:** WebSocket-based custom RPC protocol. Frames (`req`, `res`, `event`) are validated using Ajv schemas in `src/gateway/protocol/`.
- **Service Boundaries:**
    - **Gateway:** Central control plane, handles WebSocket clients, message routing, and persistence.
    - **Agent (Pi):** AI reasoning engine, integrated via RPC or local module.
    - **Nodes:** Device-local executors (macOS app, iOS/Android nodes) that connect as clients.
- **Database Access Layer:**
    - **Sessions:** JSONL files managed by `SessionManager` in `~/.openclaw/sessions/`.
    - **Memory:** SQLite with `sqlite-vec` extension for vector search (`src/agents/memory-search.ts`).
    - **Config:** JSON5 file `~/.openclaw/openclaw.json`.
- **Plugin System:** Located in `src/plugins/` and `extensions/`. The `plugin-sdk` (`src/plugin-sdk/index.ts`) provides stable interfaces for adding channels and tools.

---

## SECTION 2: Data Flow

### UI -> State -> API -> DB (Example: Sending a Chat Message)
1.  **User Action:** User submits a message in `ui/src/ui/views/chat.ts`.
2.  **UI State Update:** `handleSendChat` in `ui/src/ui/app-chat.ts` updates the `chatSending` state in `OpenClawApp`.
3.  **API Request:** `GatewayBrowserClient` (ui/src/ui/gateway.ts) sends a `chat.send` WebSocket frame.
4.  **Gateway Handling:** `src/gateway/server-methods/chat.ts` receives the request and authorizes it.
5.  **Agent Dispatch:** `dispatchInboundMessage` (src/auto-reply/dispatch.ts) invokes the Agent (Pi).
6.  **Persistence:**
    - The message is appended to the `.jsonl` session file using `SessionManager` via `appendAssistantTranscriptMessage`.
    - Tool events are broadcast back to the UI in real-time.
7.  **UI Sync:** The UI receives a `chat` event, updates `chatMessages` state, and re-renders.

### Dependency Graph (Major Modules)
- `src/cli` -> `src/config`, `src/gateway`
- `src/gateway` -> `src/agents`, `src/channels`, `src/plugins`, `src/routing`
- `src/agents` -> `src/memory`, `src/plugin-sdk`
- `ui/src/ui/app.ts` -> `ui/src/ui/controllers/*`, `ui/src/ui/views/*`

### Global Configuration
- `~/.openclaw/openclaw.json`: Main config for agents, models, channels, and gateway settings.
- `package.json`: Build and dependency configuration.
- `tsconfig.json`: TypeScript compiler options.

### Environment Variables
- `OPENCLAW_GATEWAY_TOKEN`: Token for gateway authentication.
- `OPENCLAW_GATEWAY_PASSWORD`: Password for gateway authentication.
- `OPENCLAW_PROFILE`: Set to `dev` for development mode.
- `OPENCLAW_GATEWAY_PORT`: Port for the gateway (default: 18789).
- `OPENCLAW_CLAUDE_CLI_LOG_OUTPUT`: Enables verbose logs for the Claude CLI agent.

---

## SECTION 3: UI Analysis

### Design System
- **Definition:** Defined via CSS variables in `ui/src/styles/base.css`.
- **Tailwind:** **NO**, Tailwind is not used.
- **Component Library:** Custom Lit components in `ui/src/ui/components/`. No external UI library like Material or Shoelace.
- **Modularity:** Components are modular at the file level but tightly coupled to the `OpenClawApp` "God object" for state.

### Safe Modification Points
- **Redesign Theme:** Edit CSS variables in `ui/src/styles/base.css`.
- **Add Dark Mode:** Already implemented in `ui/src/ui/theme.ts`. New themes can be added by extending `ThemeMode` and the CSS variable sets.
- **Sidebar App Switcher:** Modify the `aside.nav` element in `ui/src/ui/app-render.ts`.
- **Embed Sub-application:**
    - Register a new `Tab` in `ui/src/ui/navigation.ts`.
    - Create a new view in `ui/src/ui/views/`.
    - Add a conditional render in the `main` block of `renderApp` in `ui/src/ui/app-render.ts`.

---

## SECTION 4: Extension Opportunities

### Sub-app Mounting
- **Support:** Native "sub-app" mounting (like iframes or MFE) isn't present, but the `Tab` system in `ui/src/ui/navigation.ts` serves as a logical mounting point.
- **Layout Shell:** The `shell` class in `ui/src/ui/app-render.ts` is the primary container that can be hijacked.

### Plugin Boundaries
- **Backend:** `src/plugin-sdk/` defines clear boundaries for channel and tool plugins. Extensions live in `extensions/` and are dynamically loaded.
- **Frontend:** Less formal. No dedicated plugin system for the UI; modifications require changing core render files.

### Centralized vs Distributed
- **Routing:** Centralized in both backend (`src/routing`) and frontend (`ui/src/ui/navigation.ts`).
- **State:** Centralized (God object in `OpenClawApp`).

---

## SECTION 5: Refactor Recommendations

### Performance Audit (Static Analysis)
- **Anti-pattern:** "God object" state management in `OpenClawApp` (ui/src/ui/app.ts). Any minor state change (e.g., typing in a text box) potentially triggers a re-render check of the entire application shell.
- **Oversized Files:** Many files exceed 700-1000 LOC, violating the `AGENTS.md` guidelines. `ui/src/ui/views/usage.ts` and `src/memory/manager.ts` are primary candidates for splitting.
- **Blocking Async:** Ensure large transcript reads in `src/gateway/session-utils.fs.ts` don't block the event loop for other concurrent users.

### Specific Refactors
1.  **Decompose `OpenClawApp`:** Split the monolithic state into smaller, focused Lit Contexts or Signals (`@lit-labs/signals`).
2.  **Modularize Views:** Move all logic from `app-render.ts` into individual view components.
3.  **Execute Refactoring Strategy:** Follow the existing plan in `tmp-refactoring-strategy.md` to split oversized files.

---

## SECTION 6: Strategic Modification Plan

### Forking and Maintenance
- **Strategy:** Leverage the `extensions/` directory for all custom logic. Avoid modifying `src/` core unless necessary.
- **Merge Hell Prevention:** Use the `plugin-sdk` to isolate custom features. For UI, try to implement custom views as standalone Lit components that are only minimally referenced in `app-render.ts`.

### What to Refactor BEFORE Building
- **State Management:** Migrate to a more reactive and decoupled state system before adding complex new features to avoid worsening the "God object" problem.
- **File Splitting:** Split `src/gateway/server.impl.ts` and `ui/src/ui/app.ts` into smaller modules.

---

## SECTION 7: Red Flags

1.  **UI State Scalability:** The current state management in `ui/src/ui/app.ts` will become unmanageable as the number of features grows.
2.  **Tight Coupling:** Most UI components directly access the `state` object, making them difficult to test or reuse in isolation.
3.  **Complex Dispatcher:** `src/auto-reply/dispatch.ts` contains very dense logic with many closure variables, making it a "fragile" area where regressions are likely.
4.  **Database Scalability:** Relying on `.jsonl` files for session history may lead to performance issues as history grows, especially with the current sync/linear scan approach for some operations.
