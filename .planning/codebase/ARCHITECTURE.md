<!-- refreshed: 2026-06-18 -->
# Architecture

**Analysis Date:** 2026-06-18

## System Overview

```text
+-------------------------------------------------------------------+
| SillyTavern extension host                                         |
| `manifest.json` loads `dist/index.js` and `dist/style.css`          |
+----------------------+----------------------+---------------------+
| Entry coordinator    | Settings surface     | Runtime assets      |
| `src/index.tsx`      | `src/components/`    | `templates/`,       |
|                      |                      | `src/styles/`       |
+----------+-----------+----------+-----------+----------+----------+
           |                      |                      |
           v                      v                      v
+-------------------------------------------------------------------+
| Extension state and support modules                                |
| `src/config.ts`, `src/parser.ts`, `src/schema-to-example.ts`,       |
| `src/hooks/useForceUpdate.ts`                                       |
+-------------------------------------------------------------------+
           |
           v
+-------------------------------------------------------------------+
| SillyTavern APIs and persisted data                                |
| `SillyTavern.getContext()`, `sillytavern-utils-lib`,                |
| chat `extra.WTracker`, chat metadata, extension settings            |
+-------------------------------------------------------------------+
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| Extension manifest | Declares SillyTavern-visible bundle, CSS, load order, and `generate_interceptor` global name. | `manifest.json` |
| Entry coordinator | Boots settings, mounts React settings, wires SillyTavern DOM/event handlers, runs tracker generation, renders trackers, and mutates chat metadata/message extras. | `src/index.tsx` |
| Settings manager instance | Creates the singleton `ExtensionSettingsManager<ExtensionSettings>` used by the entry coordinator and React settings UI. | `src/components/Settings.tsx:26` |
| React settings UI | Lets users select connection profile, auto mode, prompt engineering mode, schema presets, prompt text, token limit, and context windows. | `src/components/Settings.tsx` |
| Settings and schema defaults | Defines `ExtensionSettings`, `Schema`, `PromptEngineeringMode`, default prompts, default JSON schema, default Handlebars HTML, and `EXTENSION_KEY`. | `src/config.ts` |
| Model response parser | Extracts fenced response content, parses JSON/XML, and normalizes XML array fields against the schema. | `src/parser.ts` |
| Schema example generator | Converts JSON schema into example JSON or XML for prompt-engineered requests. | `src/schema-to-example.ts` |
| Force-update hook | Provides a small React rerender trigger for settings manager mutations. | `src/hooks/useForceUpdate.ts` |
| Runtime HTML templates | Supplies extension menu button markup and per-chat schema selection popup markup. | `templates/buttons.html`, `templates/modify_schema_popup.html` |
| Styles | Provides settings layout, group member toggle state, message tracker controls, and default tracker table styling. | `src/styles/main.scss` |
| Build pipeline | Builds `src/index.tsx` to `dist/index.js` and Sass to `dist/style.css`. | `webpack.config.cjs`, `package.json`, `tsconfig.json` |

## Pattern Overview

**Overall:** Client-side SillyTavern extension with a single entry coordinator, React settings pane, and functional support modules.

**Key Characteristics:**
- Use `manifest.json` as the SillyTavern contract; it points to generated `dist/index.js`, `dist/style.css`, and the `wtrackerGenerateInterceptor` global.
- Keep runtime initialization in `src/index.tsx`: `settingsManager.initializeSettings()` resolves before `main()` mounts settings and starts global UI wiring.
- Store user-level settings through `ExtensionSettingsManager` in `src/components/Settings.tsx`; store chat-level schema and ignore-list data in `globalContext.chatMetadata` from `src/index.tsx`.
- Store rendered tracker payloads on each chat message under `message.extra[EXTENSION_KEY]` using the `WTrackerEntry` and `WTrackerStorage` shape in `src/index.tsx`.
- Render tracker HTML with Handlebars templates from settings/schema presets, not JSX components, because tracker content is inserted into SillyTavern message DOM.
- Use small pure modules for parsing and schema example generation: `src/parser.ts` and `src/schema-to-example.ts`.

## Layers

**Manifest and Build Layer:**
- Purpose: Expose the extension to SillyTavern and produce browser-loadable assets.
- Location: `manifest.json`, `webpack.config.cjs`, `package.json`, `tsconfig.json`, `babel.config.json`
- Contains: Load metadata, webpack entry/output, TypeScript compilation settings, package scripts, Babel/Jest configuration.
- Depends on: `src/index.tsx`, `src/styles/main.scss`, npm dependencies in `package.json`.
- Used by: SillyTavern extension loader through `manifest.json`.

**Bootstrap and Integration Layer:**
- Purpose: Initialize settings, mount React settings, register DOM buttons, register SillyTavern events, and install the global generate interceptor.
- Location: `src/index.tsx`
- Contains: `main()`, `renderReactSettings()`, `initializeGlobalUI()`, global Handlebars helpers, delegated click handlers, eventSource handlers.
- Depends on: `sillytavern-utils-lib`, `SillyTavern.getContext()`, `settingsManager`, `WTrackerSettings`, templates.
- Used by: Generated `dist/index.js` loaded from `manifest.json`.

**Settings Layer:**
- Purpose: Persist extension configuration and expose settings controls.
- Location: `src/components/Settings.tsx`, `src/config.ts`, `src/hooks/useForceUpdate.ts`
- Contains: Settings singleton, React component, default settings, schema/prompt defaults, rerender helper.
- Depends on: `sillytavern-utils-lib/components/react`, `ExtensionSettingsManager`, React hooks.
- Used by: `src/index.tsx` for runtime settings and `src/components/Settings.tsx` for user edits.

**Tracker Storage and Rendering Layer:**
- Purpose: Read, write, migrate, delete, and render tracker entries attached to chat messages.
- Location: `src/index.tsx`
- Contains: `getTrackerEntry()`, `ensureTrackerEntry()`, `deleteTrackerEntry()`, `renderTracker()`, `renderAllTrackers()`, group-character scoping helpers.
- Depends on: SillyTavern chat message objects, `EXTENSION_KEY`, Handlebars, DOM selectors.
- Used by: Manual regenerate/edit/delete controls, auto generation events, chat change rerendering, message swipe rerendering.

**Generation Layer:**
- Purpose: Build the prompt context, inject previous tracker summaries, call the configured connection profile, parse model output, compute relationship state, and save rendered tracker data.
- Location: `src/index.tsx`, `src/parser.ts`, `src/schema-to-example.ts`
- Contains: `generateTracker()`, `includeWTrackerMessages()`, `parseResponse()`, `schemaToExample()`.
- Depends on: `buildPrompt`, `Generator`, connection profiles, active schema preset, selected prompt engineering mode.
- Used by: Message button clicks, auto mode event handlers, regenerate controls.

**Template and Style Layer:**
- Purpose: Provide SillyTavern-rendered popups/menu items and tracker/settings CSS.
- Location: `templates/buttons.html`, `templates/modify_schema_popup.html`, `src/styles/main.scss`
- Contains: Extension menu button, chat schema selector popup, `.wtracker-*` and `.mes_wtracker` styles.
- Depends on: SillyTavern template renderer and Sass build script in `package.json`.
- Used by: `initializeGlobalUI()` and `modifyChatMetadata()` in `src/index.tsx`.

## Data Flow

### Extension Startup Path

1. SillyTavern reads `manifest.json` and loads `dist/index.js` plus `dist/style.css` (`manifest.json:6`, `manifest.json:7`).
2. The source entry side effect calls `settingsManager.initializeSettings()` (`src/index.tsx:949`).
3. `main()` renders the settings pane and starts global UI wiring (`src/index.tsx:943`).
4. `renderReactSettings()` creates `#wtracker-react-settings-root` inside `#extensions_settings` and renders `WTrackerSettings` (`src/index.tsx:921`).
5. `initializeGlobalUI()` inserts message/group/member controls, extension menu content, event listeners, and the global generate interceptor (`src/index.tsx:746`).

### Manual Tracker Generation Path

1. A delegated click on `.mes_wtracker_button` or `.wtracker-regenerate-button` calls `generateTracker(messageId)` (`src/index.tsx:772`, `src/index.tsx:781`).
2. `generateTracker()` reads the selected message, skips ignored group members, checks/cancels any pending request, and validates `settings.profileId` (`src/index.tsx:565`).
3. The function ensures chat schema metadata defaults, reads the active schema preset, resolves the selected connection profile, and captures existing `<details>` open state (`src/index.tsx:590`).
4. `buildPrompt()` creates the base prompt context for the configured profile and message range (`src/index.tsx:613`).
5. `includeWTrackerMessages()` injects prior tracker summaries into the message list when configured (`src/index.tsx:625`).
6. Native mode sends the JSON schema through `overridePayload.json_schema`; JSON/XML prompt mode builds an example with `schemaToExample()` and parses the model response with `parseResponse()` (`src/index.tsx:657`, `src/index.tsx:672`, `src/index.tsx:681`).
7. The response updates `relationshipValue`, `behaviorValue`, tracker value, and tracker HTML on `message.extra.WTracker` through `ensureTrackerEntry()` (`src/index.tsx:698`).
8. `renderTracker()` must succeed before `saveChat()` persists the message; render failure deletes the tentative tracker entry (`src/index.tsx:708`, `src/index.tsx:724`, `src/index.tsx:727`).

### Auto Tracker Generation Path

1. `initializeGlobalUI()` registers `EventNames.CHARACTER_MESSAGE_RENDERED` and `EventNames.USER_MESSAGE_RENDERED` handlers (`src/index.tsx:828`, `src/index.tsx:832`).
2. Each handler checks `settings.autoMode` against `incomingTypes` or `outgoingTypes` arrays from `src/index.tsx:28` and `src/index.tsx:29`.
3. Matching events call `generateTracker(messageId)` and follow the manual tracker generation path (`src/index.tsx:830`, `src/index.tsx:834`).

### Saved Tracker Render Path

1. `EventNames.CHAT_CHANGED` calls `renderTracker(i)` for each message in `globalContext.chat` (`src/index.tsx:836`, `src/index.tsx:842`).
2. `renderTracker()` reads the message tracker entry, compiles stored Handlebars HTML, inserts `.mes_wtracker` before `.mes_text`, and prepends edit/regenerate/delete controls (`src/index.tsx:69`).
3. If a stored template fails during chat-change rendering, `deleteTrackerEntry()` removes invalid tracker data and `saveChat()` persists cleanup (`src/index.tsx:844`, `src/index.tsx:846`, `src/index.tsx:852`).
4. `EventNames.MESSAGE_SWIPED` rerenders the swiped message or all messages (`src/index.tsx:855`).

### Settings Update Path

1. `WTrackerSettings` reads settings from `settingsManager.getSettings()` and localizes schema text in React state (`src/components/Settings.tsx:29`).
2. All setting edits flow through `updateAndRefresh()`, which mutates the manager settings, calls `settingsManager.saveSettings()`, and forces a React rerender (`src/components/Settings.tsx:34`).
3. Schema preset list edits rebuild `settings.schemaPresets`; schema JSON edits parse before saving; schema HTML edits save raw template text (`src/components/Settings.tsx:64`, `src/components/Settings.tsx:80`, `src/components/Settings.tsx:99`).

### Per-Chat Schema Selection Path

1. The extension menu button from `templates/buttons.html` is rendered by `globalContext.renderExtensionTemplateAsync()` (`src/index.tsx:817`).
2. Clicking `#wtracker_modify_schema_preset` calls `modifyChatMetadata()` (`src/index.tsx:821`).
3. `modifyChatMetadata()` ensures `chatMetadata.WTracker.schemaKey`, renders `templates/modify_schema_popup.html`, and saves a changed schema key through `context.saveMetadataDebounced()` (`src/index.tsx:872`, `src/index.tsx:895`, `src/index.tsx:910`).

**State Management:**
- Extension settings live in SillyTavern extension settings through `settingsManager` from `src/components/Settings.tsx:26`.
- Chat-level state lives in `SillyTavern.getContext().chatMetadata`, including `WTracker.schemaKey` and `wtracker_ignore` in `src/index.tsx`.
- Message-level tracker state lives in `message.extra.WTracker` with optional group-character scoping through `characterTrackers` in `src/index.tsx:31` and `src/index.tsx:41`.
- In-flight request state lives in the module-level `pendingRequests` map in `src/index.tsx:27`.
- UI-only transient state includes React `schemaText` in `src/components/Settings.tsx:31` and captured `<details>` open state in `src/index.tsx:603` and `src/index.tsx:486`.

## Key Abstractions

**Extension Settings:**
- Purpose: Typed shape for persisted extension configuration.
- Examples: `ExtensionSettings`, `Schema`, `PromptEngineeringMode`, `defaultSettings` in `src/config.ts`.
- Pattern: Central config module exports types and defaults; `ExtensionSettingsManager` in `src/components/Settings.tsx` owns persistence.

**Tracker Entry Storage:**
- Purpose: Persist generated tracker data, tracker HTML template, relationship/behavior counters, and optional group-character identity.
- Examples: `WTrackerEntry`, `WTrackerStorage`, `getTrackerEntry()`, `ensureTrackerEntry()`, `deleteTrackerEntry()` in `src/index.tsx`.
- Pattern: Store extension-owned data under `message.extra[EXTENSION_KEY]`; use nested `characterTrackers` for group-chat character scoping.

**Prompt Engineering Mode:**
- Purpose: Select between native JSON schema payloads and prompt-engineered JSON/XML output.
- Examples: `PromptEngineeringMode` in `src/config.ts:3`, `generateTracker()` branches in `src/index.tsx:657`.
- Pattern: Native mode uses `overridePayload.json_schema`; JSON/XML modes use Handlebars prompt templates plus `schemaToExample()` and `parseResponse()`.

**SillyTavern Event Hooks:**
- Purpose: Connect WTracker behavior to message rendering, chat changes, message swipes, and generation interception.
- Examples: eventSource handlers and `globalThis.wtrackerGenerateInterceptor` in `src/index.tsx:828`, `src/index.tsx:865`.
- Pattern: Register listeners once from `initializeGlobalUI()` after settings initialization.

**Handlebars Templates:**
- Purpose: Render stored tracker HTML and prompt templates from schema/prompt settings.
- Examples: helper registration in `src/index.tsx:45`, tracker rendering in `src/index.tsx:87`, prompt compilation in `src/index.tsx:673`, defaults in `src/config.ts:326`.
- Pattern: Compile with `{ noEscape: true, strict: true }`; preserve template failures as errors that prevent saving generated tracker data.

## Entry Points

**SillyTavern Manifest:**
- Location: `manifest.json`
- Triggers: SillyTavern extension loader.
- Responsibilities: Load `dist/index.js`, load `dist/style.css`, and advertise `wtrackerGenerateInterceptor`.

**Bundle Source Entry:**
- Location: `src/index.tsx`
- Triggers: Webpack entry in `webpack.config.cjs:6`.
- Responsibilities: Register helpers, initialize settings, mount settings UI, wire extension controls, and install runtime handlers.

**Settings UI Entry:**
- Location: `src/components/Settings.tsx`
- Triggers: `renderReactSettings()` in `src/index.tsx:921`.
- Responsibilities: Read/update extension settings and render settings controls.

**Build Entry:**
- Location: `package.json`
- Triggers: `npm run build` or `npm run dev`.
- Responsibilities: Compile Sass to `dist/style.css` and webpack `src/index.tsx` to `dist/index.js`.

**Template Entry:**
- Location: `templates/buttons.html`, `templates/modify_schema_popup.html`
- Triggers: `globalContext.renderExtensionTemplateAsync()` calls in `src/index.tsx:817` and `src/index.tsx:895`.
- Responsibilities: Provide SillyTavern-rendered HTML fragments for extension menu and chat schema popup.

## Architectural Constraints

- **Threading:** Browser single-threaded event loop with async `Generator.generateRequest()` callbacks in `src/index.tsx:631`; cancellation is coordinated through `AbortController` and `pendingRequests` in `src/index.tsx:27`.
- **Global state:** Module-level `globalContext`, `generator`, `pendingRequests`, `incomingTypes`, `outgoingTypes`, Handlebars helper registration, and `globalThis.wtrackerGenerateInterceptor` live in `src/index.tsx`.
- **SillyTavern globals:** Runtime code assumes `SillyTavern`, `$`, DOM nodes like `#extensions_settings`, `#message_template`, `#rm_group_members`, and SillyTavern event names are present before `src/index.tsx` runs.
- **Circular imports:** No circular source imports detected across `src/index.tsx`, `src/components/Settings.tsx`, `src/config.ts`, `src/parser.ts`, `src/schema-to-example.ts`, and `src/hooks/useForceUpdate.ts`.
- **Module specifiers:** Local imports use `.js` specifiers in TypeScript source, and `webpack.config.cjs` maps `.js` to `.ts`/`.tsx` through `resolve.extensionAlias`.
- **Generated assets:** `dist/index.js` and `dist/style.css` are generated from `src/index.tsx` and `src/styles/main.scss` but are the files loaded by `manifest.json`.
- **Project skills:** Repo-local skill directories `.codex/skills/` and `.agents/skills/` are not present, so no project-specific skill constraints apply.

## Anti-Patterns

### Coordinator Growth In The Entry File

**What happens:** `src/index.tsx` contains bootstrap code, DOM wiring, storage helpers, group-member controls, generation orchestration, rendering, editing, deletion, and per-chat schema UI.
**Why it's wrong:** Changes to independent behavior share one high-blast-radius module and make small tracker features harder to test in isolation.
**Do this instead:** Keep `src/index.tsx` as bootstrap and move new pure storage helpers into a focused module such as `src/tracker-storage.ts`, generation helpers into `src/generation.ts`, and DOM wiring into `src/ui.ts` when adding or extracting substantial behavior.

### Direct DOM HTML Injection

**What happens:** `src/index.tsx` builds tracker controls and edit popup content with inline HTML strings, and saved tracker templates are inserted through `container.innerHTML`.
**Why it's wrong:** Runtime rendering depends on caller-provided template validity and DOM selector stability; broken templates can throw during chat rendering.
**Do this instead:** Put reusable SillyTavern HTML fragments under `templates/`, keep dynamic tracker template validation near `renderTracker()` in `src/index.tsx:69`, and keep render failures inside the existing delete-and-save recovery path in `src/index.tsx:844`.

## Error Handling

**Strategy:** Surface user-facing failures through `st_echo`, log developer diagnostics to `console.error`, and avoid persisting generated tracker data unless it can render successfully.

**Patterns:**
- `settingsManager.initializeSettings()` catches migration/startup failure, logs it, and shows a SillyTavern error toast (`src/index.tsx:949`).
- `parseResponse()` catches JSON/XML parse failures, logs the format and raw content, and throws normalized errors for `generateTracker()` (`src/parser.ts:36`).
- `generateTracker()` treats aborts separately from real errors, removes spinner classes in `finally`, and emits user-facing tracker generation failure messages (`src/index.tsx:565`).
- Render validation happens before `saveChat()`; a render error deletes the tentative entry through `deleteTrackerEntry()` and rerenders the message (`src/index.tsx:708`, `src/index.tsx:727`).
- Chat-change rerendering catches broken saved templates, deletes invalid tracker data, and persists cleanup with `saveChat()` (`src/index.tsx:836`).
- Invalid settings schema JSON in `WTrackerSettings` is ignored until valid JSON parses successfully (`src/components/Settings.tsx:80`).

## Cross-Cutting Concerns

**Logging:** Use `console.error` for developer diagnostics and `st_echo` from `sillytavern-utils-lib/config` for user-visible success/error/info messages in `src/index.tsx`.
**Validation:** Use JSON parsing in `src/components/Settings.tsx`, response parsing in `src/parser.ts`, Handlebars strict rendering in `src/index.tsx`, and schema-derived XML array normalization in `src/parser.ts`.
**Authentication:** No extension-owned authentication layer exists; connection/profile credentials are delegated to SillyTavern connection profiles referenced from `src/index.tsx:590`.
**Persistence:** Use `settingsManager.saveSettings()` in `src/components/Settings.tsx`, `globalContext.saveChat()`/`saveChat()` in `src/index.tsx`, and `context.saveMetadataDebounced()` for chat metadata in `src/index.tsx`.
**Styling:** Add extension CSS in `src/styles/main.scss`; `package.json` builds it into `dist/style.css` loaded by `manifest.json`.
**Templates:** Add SillyTavern-rendered HTML fragments to `templates/` and load them with `globalContext.renderExtensionTemplateAsync()` from `src/index.tsx`.

---

*Architecture analysis: 2026-06-18*
