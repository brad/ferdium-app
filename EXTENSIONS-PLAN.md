# Chrome Extensions Integration Plan for Ferdium

This document provides a comprehensive technical blueprint and step-by-step implementation guide for integrating Chrome extension support into Ferdium service webviews using `electron-chrome-extensions` (from `electron-browser-shell`).

---

## 1. Overview & Objectives

### Goal
Enable users to load unpacked Chrome extensions (e.g. password managers like Bitwarden or 1Password, ad blockers, productivity tools) and apply them to Ferdium service webviews with per-service toggles and UI controls for extension actions and context menus.

### Key Requirements
1. **Extension Management UI**: A dedicated **Extensions** section in Ferdium Settings to load, enable, disable, and remove unpacked extensions.
2. **Per-Service Control**: Ability to enable or disable installed extensions on a per-service basis.
3. **Extension Action UI**: Display extension action icons (browser actions) in the main app header/top bar for the currently active service, opening extension popups upon click.
4. **Context Menu Integration**: Integrate extension context menu items into service webview right-click menus.
5. **Security**: Require explicit user interaction (file picker) to load unpacked extensions; no automatic folder watching or arbitrary execution.

---

## 2. Architecture & Tech Stack

- **Core Library**: `electron-chrome-extensions` (and related utilities from `electron-browser-shell`).
- **Main Process**:
  - Initializes `ElectronChromeExtensions` instance.
  - Registers `crx://` protocol handler on app sessions (`ElectronChromeExtensions.handleCRXProtocol`).
  - Listens to extension lifecycle IPC events (load, unload, toggle per service).
  - Persists extension metadata in `userData/extensions.json` or MobX state store.
- **Renderer Process / Stores**:
  - `ExtensionsStore`: MobX store for managing loaded extension metadata, per-service bindings, and active action states.
  - `ExtensionsScreen`: Settings component for managing unpacked extensions.
  - `ExtensionActionList`: Header/Titlebar component rendering extension action icons using `<browser-action-list>` or custom extension button popups.
- **Service Webviews**:
  - Extension logic attached to partition sessions (`persist:service-<serviceId>`).

---

## 3. Data Models & State Management

### 3.1 Extension Model (`src/models/Extension.ts`)
```typescript
export interface ExtensionManifest {
  name: string;
  version: string;
  description?: string;
  icons?: Record<string, string>;
  browser_action?: object;
  action?: object;
}

export interface Extension {
  id: string;
  name: string;
  version: string;
  description: string;
  path: string;
  iconUrl?: string;
  enabled: boolean;
  enabledServices: string[]; // List of service IDs where this extension is enabled (empty array = enabled globally if configured)
}
```

### 3.2 MobX Store (`src/stores/ExtensionsStore.ts`)
- Manages state for installed extensions.
- Persists data to local storage or setting files (`config/extensions.json`).
- Exposes actions:
  - `loadUnpackedExtension(path: string)`
  - `removeExtension(extensionId: string)`
  - `toggleExtension(extensionId: string, enabled: boolean)`
  - `toggleExtensionForService(extensionId: string, serviceId: string, enabled: boolean)`
  - `getExtensionsForService(serviceId: string)`

---

## 4. Settings UI (`Settings -> Extensions`)

### 4.1 Component Structure
- **Location**: `src/components/settings/extensions/`
  - `ExtensionsDashboard.tsx`: Main view listing installed extensions.
  - `ExtensionItem.tsx`: Card component representing a single loaded extension with controls (toggle, per-service selector, delete).
- **Container**: `src/containers/settings/ExtensionsScreen.tsx`
- **Navigation**: Add "Extensions" item to `src/components/settings/navigation/SettingsNavigation.tsx`.

### 4.2 Features in UI
- **"Load Unpacked Extension" Button**: Triggers Electron's `dialog.showOpenDialog` for directory selection.
- **Extension List**:
  - Displays icon, name, version, and description.
  - Toggle switch for global enable/disable.
  - Per-service assignment dropdown / multi-select checkboxes.
  - "Remove" button with confirmation modal.

---

## 5. Main Process & Session Setup (`src/index.ts` & `src/electron/ipc-api/extensions.ts`)

### 5.1 Extension Manager Initialization
```typescript
import { ElectronChromeExtensions } from 'electron-chrome-extensions';

let extensionsHandler: ElectronChromeExtensions | null = null;

export function setupExtensions(appSession: Electron.Session) {
  // Handle CRX protocol for extension assets & browser action popups
  ElectronChromeExtensions.handleCRXProtocol(appSession);

  extensionsHandler = new ElectronChromeExtensions({
    session: appSession,
    // Add custom tab details assignment if required
  });
}
```

### 5.2 Binding Extensions to Service Partition Sessions
In `src/index.ts` where service sessions (`persist:service-${serviceId}`) are created:
1. Handle CRX protocol on `serviceSession`.
2. Load active extensions assigned to `serviceId` into `serviceSession.extensions`.
3. Handle extension IPC requests to reload or unload extensions dynamically when settings change.

---

## 6. Extension Action UI (Main App Top Bar)

### 6.1 UI Placement
Place extension action buttons in the main app header bar (`src/components/ui/Header.tsx` or title bar component).

### 6.2 Implementation
- When an active service tab is selected in Ferdium, query the enabled extensions for that service.
- Render extension action icons using `<browser-action-list>` web component (injected via preload) or custom popup triggers.
- Clicking an icon opens the extension's popup window anchored below the icon.

---

## 7. Context Menu Integration (`src/webview/contextMenuBuilder.ts`)

- `electron-chrome-extensions` provides context menu item handlers via `chrome.contextMenus`.
- In `contextMenuBuilder.ts`:
  - Fetch context menu items registered by extensions for the current session/tab.
  - Append extension-provided context menu items into Ferdium's standard right-click context menu.

---

## 8. Step-by-Step Implementation Roadmap

### Phase 1: Dependencies & Store Setup
1. Add `electron-chrome-extensions` to `dependencies` in `package.json`.
2. Create `src/models/Extension.ts` model interface.
3. Create `src/stores/ExtensionsStore.ts` MobX store and register it in `src/stores/index.ts`.

### Phase 2: Main Process & IPC Integration
1. Create `src/electron/ipc-api/extensions.ts` to handle file dialogs and extension loading in Electron.
2. Update `src/index.ts` to initialize `ElectronChromeExtensions` and handle `crx://` protocols for main & service sessions.

### Phase 3: Settings UI Component
1. Add `ExtensionsScreen` container and `ExtensionsDashboard` component.
2. Add "Extensions" to `SettingsNavigation.tsx`.
3. Implement extension loading, per-service toggle, and deletion UI.

### Phase 4: Top Bar Extension Action Icons & Context Menus
1. Add extension action buttons to the main header bar.
2. Integrate extension context menu items in `src/webview/contextMenuBuilder.ts`.

### Phase 5: Testing & Verification
1. Test loading popular unpacked Chrome extensions (e.g. Bitwarden, uBlock Origin).
2. Verify extension state persists across app restarts.
3. Verify per-service isolation (extensions enabled on Service A do not affect Service B if disabled for B).

---

## 9. Security Considerations

- **Explicit User Action Required**: Extensions can only be loaded via explicit directory selection in Settings.
- **No Automatic Execution**: Arbitrary filesystem watching is disabled to prevent unauthorized extension injection.
- **Permissions Audit**: Prompt users with extension permission warnings when loading new unpacked extensions.
