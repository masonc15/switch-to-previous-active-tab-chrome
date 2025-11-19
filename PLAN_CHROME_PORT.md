# Chrome Porting Plan – Switch to Previous Active Tab

## Feasibility snapshot
- Core tab-switching, keyboard shortcuts, context menus, and the popup UI can run on Chrome using Manifest V3 (MV3) with `chrome.tabs`, `chrome.commands`, `chrome.contextMenus`, and `chrome.action`.
- Two Firefox-only features cannot be replicated exactly: Containers (`contextualIdentities`) and the browser-action middle-click signal (`OnClickData.button`). These will need no-op fallbacks or alternative UI affordances.
- MV3 service workers are ephemeral; in-memory tab history must be persisted (for example, `chrome.storage.session`) to keep behavior stable between suspensions.

## Target architecture
- Manifest V3 with a service worker (`background.service_worker`). Move toolbar definition to `action` block and update permissions/host_permissions.
- Promise-friendly utility wrapper to keep existing Promise-based style over callback APIs (`chrome.*`). Use guarded runtime detection for Firefox vs. Chrome if a dual-track is kept.
- State persistence: mirror `oTabs`, `oPrefs`, and `oRATprefs` to `chrome.storage.session` (fast, non-durable) and hydrate from `chrome.tabs.query` on worker startup; fall back to `chrome.storage.local` if session storage is unavailable.
- UI: reuse existing `popup.html/js/css`. For service-worker-triggered popups, rely on `chrome.action.setPopup` + `chrome.action.openPopup()` from visible extension pages; fall back to `chrome.windows.create` when the action is overflowed.

## Work breakdown
1) **Manifest migration**
   - Convert to MV3 fields (`manifest_version: 3`, `action`, `background.service_worker`, `permissions` → include `tabs`, `storage`, `contextMenus`, `tabGroups`; remove `contextualIdentities`).
   - Replace `browser_action` contexts with `action` in menus/commands; update icons to MV3 `action.default_icon`.
2) **API surface updates**
   - Add lightweight `promisifyChrome` helper or import the official WebExtension polyfill to preserve Promise usage (`browser.*` style).
   - Replace `browser.*` calls with wrapper (`ext.*`) that resolves to `chrome` on Chrome and `browser` on Firefox.
   - Swap `browser.menus` → `chrome.contextMenus`; adjust contexts to `"action"`.
3) **Service worker adaptation**
   - Port `background.js` to service-worker-safe code (no direct DOM access, remove `window.matchMedia` usage or emulate via stored theme pref).
   - Persist `oTabs` updates to `chrome.storage.session` after each mutating listener; rehydrate in the worker’s `runtime.onInstalled` and `runtime.onStartup`.
   - Rework popup-opening flows that relied on synchronous background state; ensure data is injected via `chrome.runtime.sendMessage` that rehydrates state on demand.
4) **Feature gap handling**
   - Containers: gate `contextualIdentities` calls; hide container badges when API is unavailable (Chrome).
   - Middle-click alternate action: since `chrome.action.onClicked` lacks button info, expose the alternate action via context menu entry and/or an optional second command shortcut.
   - Tab hiding/attention flags: guard `hidden`/`attention` properties behind feature detection to avoid Chrome errors.
5) **Tab group labels**
   - Keep group labeling using `chrome.tabGroups.query` (Chrome 89+); add existence checks and fallback styling when absent.
6) **Reload All Tabs**
   - Verify MV3 allowance for bypassCache and sequential reload logic; move any long-lived listeners (`loadMonitor`, `attnHandler`) to re-register per worker wake cycle.
7) **Options/popup messaging**
   - Update `browser.runtime.sendMessage` calls to use the wrapper; ensure `incognito` preference respects Chrome split/incognito rules.
   - Audit `getSettings`/`getWindow`/`getGlobal` to handle worker suspension by rehydrating from storage when `oTabs` is empty.
8) **Build & packaging**
   - Add npm script (optional) for lint/zip; produce `dist/chrome/` bundle with MV3 manifest and service worker.
   - Keep Firefox MV2 build separated (if needed) via dual manifests or build-time templating.

## Risk log
- **Loss of container indicators**: Chrome lacks `contextualIdentities`; UI badges will be suppressed or replaced.
- **Middle-click alt-action**: No reliable clickData on Chrome action; must re-map to menu/shortcut.
- **Service worker suspension**: If hydration lags, first command after idle could be slower; mitigated by persisting `oTabs` to session storage and rebuilding on wake.
- **Hidden/attention flags**: Feature detection required to avoid runtime errors on Chrome.

## Validation checklist
- Manual: tab flip (same window/global), popup lists (with/without favicons), shortcuts, context menus, reload-all variants, tab-group labels.
- Chrome versions ≥ 120 in normal & incognito (split mode) with persisted preferences.
- Automated sanity: lightweight Jest/AVA unit tests for the promisify wrapper and state persistence helpers; lint with `eslint` + MV3 globals.

## Deliverables
- Updated MV3 `manifest.json`, service-worker `background.js`, wrapper utility, and conditional UI logic.
- Build/readme instructions for Chrome install + any Firefox compatibility notes.
