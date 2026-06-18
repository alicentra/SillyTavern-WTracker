# External Integrations

**Analysis Date:** 2026-06-18

## APIs & External Services

**SillyTavern Host Extension APIs:**
- SillyTavern browser extension host - Loads the extension, exposes chat context, events, popups, templates, metadata persistence, and message persistence.
  - SDK/Client: Global `SillyTavern` object in `src/index.tsx` and `src/components/Settings.tsx`; helper APIs from `sillytavern-utils-lib` in `src/index.tsx` and `src/components/Settings.tsx`.
  - Auth: Not applicable in this repo; the extension inherits the active SillyTavern browser session.
  - Key files: `manifest.json`, `src/index.tsx`, `src/components/Settings.tsx`.

**SillyTavern Model Connection Profiles:**
- SillyTavern connection manager - Selects the model/provider profile used for tracker generation.
  - SDK/Client: `STConnectionProfileSelect` in `src/components/Settings.tsx`, `extensionSettings.connectionManager`, `CONNECT_API_MAP`, `buildPrompt`, and `Generator.generateRequest` in `src/index.tsx`.
  - Auth: Provider credentials are managed by SillyTavern connection profiles outside this repo; no env var or provider credential file is read by this extension.
  - Runtime behavior: `src/index.tsx` resolves the selected profile from `settings.profileId`, builds messages with `buildPrompt`, and sends generation through `Generator.generateRequest`.

**Model Provider APIs:**
- Indirect provider access through SillyTavern - The extension does not import OpenAI, Anthropic, Google, Kobold, OpenRouter, or other provider SDKs directly.
  - SDK/Client: `sillytavern-utils-lib` `Generator` in `src/index.tsx`.
  - Auth: Not configured in `package.json`, `src/index.tsx`, or `.env`; credentials remain in SillyTavern host settings.
  - Supported modes: Native structured output, prompt-engineered JSON, and prompt-engineered XML are selected through `PromptEngineeringMode` in `src/config.ts` and `src/components/Settings.tsx`.

**Package Registry:**
- npm registry - Used for development dependencies listed in `package.json` and resolved in `package-lock.json`.
  - SDK/Client: npm via `package-lock.json`.
  - Auth: No `.npmrc` file is present in the repo, and no package-manager credentials were read.

## Data Storage

**Databases:**
- Not detected.
  - Connection: Not applicable.
  - Client: Not applicable.
  - Key files checked: `package.json`, `src/index.tsx`, `src/components/Settings.tsx`.

**Host-Persisted Settings and Metadata:**
- SillyTavern extension settings - `ExtensionSettingsManager<ExtensionSettings>` persists settings keyed by `EXTENSION_KEY`.
  - Connection: SillyTavern host storage through `sillytavern-utils-lib`.
  - Client: `ExtensionSettingsManager` in `src/components/Settings.tsx`.
  - Stored data: profile id, max response tokens, auto mode, schema presets, prompts, prompt engineering mode, message inclusion counts from `src/config.ts`.
- SillyTavern chat metadata - Stores chat-level schema preset and ignored group-member avatars.
  - Connection: `SillyTavern.getContext().chatMetadata` in `src/index.tsx`.
  - Client: `saveMetadataDebounced()` in `src/index.tsx`.
  - Stored data: `schemaKey` and `wtracker_ignore` values under `EXTENSION_KEY` in `src/index.tsx`.
- SillyTavern message extras - Stores generated tracker payloads on individual chat messages.
  - Connection: `message.extra[EXTENSION_KEY]` in `src/index.tsx`.
  - Client: `saveChat()` from SillyTavern context in `src/index.tsx`.
  - Stored data: tracker JSON value, tracker HTML, relationship and behavior values, and group-character scoped tracker entries.

**File Storage:**
- Local extension assets only.
  - Source templates: `templates/buttons.html` and `templates/modify_schema_popup.html`.
  - Images: `images/overview.png`, `images/modify_for_this_chat.png`, and `images/settings.gif`.
  - Built artifacts: `dist/index.js` and `dist/style.css`.
  - Runtime file uploads or external object storage: Not detected.

**Caching:**
- In-memory request tracking only.
  - Client: `pendingRequests = new Map<number, string>()` in `src/index.tsx`.
  - Purpose: Tracks active generation requests by message id so a second click can abort the request.
  - External cache: Not detected.

## Authentication & Identity

**Auth Provider:**
- Not implemented by this extension.
  - Implementation: User identity and provider credentials are owned by the SillyTavern host; this repo does not define login, session, OAuth, JWT, or API-key handling.
  - Key files checked: `package.json`, `manifest.json`, `src/index.tsx`, `src/components/Settings.tsx`.

**Provider Credentials:**
- Managed by SillyTavern connection profiles.
  - Implementation: `settings.profileId` in `src/config.ts` and `src/components/Settings.tsx` selects a profile; `src/index.tsx` sends the profile id to `Generator.generateRequest`.
  - Secrets in repo: Not detected. No `.env*` files are present; `.gitignore` excludes `.env`.

## Monitoring & Observability

**Error Tracking:**
- None detected.
  - There is no Sentry, Datadog, OpenTelemetry, LogRocket, or similar dependency in `package.json`.

**Logs:**
- Browser console and SillyTavern UI notifications.
  - `console.error` is used in `src/index.tsx` and `src/parser.ts`.
  - `st_echo` from `sillytavern-utils-lib/config` is used in `src/index.tsx` for success, info, and error notifications.
  - Parser failures log both parse errors and raw model content in `src/parser.ts`; new integrations should avoid logging secrets or provider credentials.

## CI/CD & Deployment

**Hosting:**
- GitHub repository install through the SillyTavern extension installer.
  - Repository URL is documented in `readme.md`.
  - Extension metadata and hosted artifact paths are declared in `manifest.json`.

**CI Pipeline:**
- None detected.
  - No GitHub Actions, GitLab CI, CircleCI, or other CI config files were found in the scanned repo file list.
  - Build and test commands are local npm scripts in `package.json`.

## Environment Configuration

**Required env vars:**
- None detected for runtime.
- `NODE_ENV` is set by `cross-env` inside `package.json` scripts for development and production builds.
- Model provider credentials are configured in SillyTavern connection profiles, not through repo env vars.

**Secrets location:**
- No repo-local secret files detected.
- `.gitignore` excludes `.env`.
- Provider API keys and model settings live outside this extension in SillyTavern host configuration.

## Webhooks & Callbacks

**Incoming:**
- No HTTP endpoints or webhooks are defined.
- SillyTavern event callbacks are registered in `src/index.tsx`:
  - `EventNames.CHARACTER_MESSAGE_RENDERED` triggers auto-generation for response messages when auto mode allows it.
  - `EventNames.USER_MESSAGE_RENDERED` triggers auto-generation for input messages when auto mode allows it.
  - `EventNames.CHAT_CHANGED` re-renders trackers and cleans invalid tracker data.
  - `EventNames.MESSAGE_SWIPED` re-renders tracker output after swipe changes.
- DOM callbacks are registered in `src/index.tsx` for message tracker buttons, group-member ignore toggles, and extension menu actions.
- Global generation interceptor:
  - `manifest.json` declares `generate_interceptor: "wtrackerGenerateInterceptor"`.
  - `src/index.tsx` assigns `globalThis.wtrackerGenerateInterceptor` to inject previous tracker messages into generation context.

**Outgoing:**
- Model generation requests are sent indirectly through SillyTavern.
  - Client: `Generator.generateRequest` in `src/index.tsx`.
  - Destination: The selected SillyTavern connection profile and provider; this repo does not construct provider URLs directly.
- Prompt construction uses SillyTavern context and profile metadata.
  - Client: `buildPrompt` in `src/index.tsx`.
  - Inputs: profile preset, context, instruct, sysprompt, target character id, message range, and group-name inclusion settings.
- External webhook callbacks: Not detected.

---

*Integration audit: 2026-06-18*
