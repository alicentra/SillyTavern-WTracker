# Codebase Concerns

**Analysis Date:** 2026-06-18

## Tech Debt

**Main extension module has too many responsibilities:**
- Issue: `src/index.tsx` owns tracker rendering, Handlebars helpers, SillyTavern DOM integration, group-member controls, chat metadata, LLM request orchestration, parsing handoff, data persistence, and global generation interception in one 824-line file.
- Files: `src/index.tsx`, `src/parser.ts`, `src/schema-to-example.ts`, `src/components/Settings.tsx`
- Impact: Changes to one workflow can regress unrelated behavior because shared globals, DOM selectors, chat mutation, and generation code are interleaved.
- Fix approach: Split `src/index.tsx` into focused modules such as `src/render-tracker.ts`, `src/tracker-storage.ts`, `src/generate-tracker.ts`, `src/group-ignore.ts`, and `src/sillytavern-events.ts`; keep `src/index.tsx` as the initializer.

**TypeScript strictness is weakened locally with broad `any` usage:**
- Issue: Important runtime contracts use `any`, `unknown as T`, and `@ts-ignore` around chat messages, schema objects, response content, and SillyTavern globals.
- Files: `src/index.tsx:47`, `src/index.tsx:187`, `src/index.tsx:395`, `src/index.tsx:667`, `src/parser.ts:11`, `src/schema-to-example.ts:29`, `src/components/Settings.tsx:83`
- Impact: Schema, chat metadata, and external library shape changes are caught at runtime instead of compile time; generated data can reach persistence without typed validation.
- Fix approach: Define local interfaces for SillyTavern message extras, tracker response data, schema presets, connection profiles, and generator responses; replace `@ts-ignore` with explicit extension fields or wrapper types.

**Build output is committed and is the runtime artifact:**
- Issue: `manifest.json` loads `dist/index.js` and `dist/style.css`, while source lives under `src/`; `git ls-files` shows both `dist/index.js` and `dist/style.css` are tracked.
- Files: `manifest.json:6`, `manifest.json:7`, `dist/index.js`, `dist/style.css`, `src/index.tsx`, `src/styles/main.scss`
- Impact: Source and runtime bundle can drift unless every source change is followed by `npm run build` and the generated files are committed.
- Fix approach: Treat `npm run build` as mandatory verification for code changes; add a CI or pre-release check that fails when `dist/` differs from rebuilt output.

**Version metadata is inconsistent:**
- Issue: Package, manifest, and runtime settings use different versions.
- Files: `package.json:3`, `manifest.json:9`, `src/config.ts:407`
- Impact: Installers, settings migrations, and support reports can disagree about the active extension version.
- Fix approach: Use one version source of truth and generate or validate `manifest.json` and `src/config.ts` from it during build.

## Known Bugs

**Chat-specific schema selection is written but not used during generation:**
- Symptoms: The "Modify WTracker schema" chat-level popup stores `chatMetadata[WTracker].schemaKey`, but `generateTracker()` reads `settings.schemaPresets[settings.schemaPreset]` instead of the chat metadata key.
- Files: `src/index.tsx:588`, `src/index.tsx:592`, `src/index.tsx:593`, `src/index.tsx:872`, `src/index.tsx:909`, `templates/modify_schema_popup.html`
- Trigger: Select a different schema preset for the active chat via the extension menu, then generate or regenerate a tracker.
- Workaround: Change the global schema preset in `src/components/Settings.tsx` settings before generation.

**Render errors can delete existing tracker data on chat load/change:**
- Symptoms: On `EventNames.CHAT_CHANGED`, any message whose tracker template throws is deleted from chat data and the chat is saved.
- Files: `src/index.tsx:836`, `src/index.tsx:843`, `src/index.tsx:846`, `src/index.tsx:852`
- Trigger: Open a chat containing tracker data whose saved HTML template no longer matches the saved data shape, or introduce a strict Handlebars error in a schema template.
- Workaround: Fix the template or schema before opening the chat; there is no local backup path in the extension.

**Prompt-mode parsing is brittle for common model formatting variations:**
- Symptoms: `parseResponse()` extracts only the first markdown code block using a narrow language regex and then parses JSON/XML directly; malformed code fence labels, multiple code blocks, leading commentary, or XML rooted differently can fail.
- Files: `src/parser.ts:38`, `src/parser.ts:45`, `src/parser.ts:55`, `src/index.tsx:679`, `src/index.tsx:681`
- Trigger: Use JSON or XML prompt-engineering mode with a model that returns prose plus code, multiple snippets, or a non-`root` XML wrapper.
- Workaround: Use Native API mode with structured output when the selected API supports it, or manually edit prompts to force exactly one compatible fenced block.

**Schema preset editing accepts invalid schema shapes silently:**
- Symptoms: `Settings.tsx` saves any parseable JSON as a schema value; downstream code assumes object schemas with `properties`, array `items`, and fields used by the default HTML template.
- Files: `src/components/Settings.tsx:80`, `src/components/Settings.tsx:83`, `src/components/Settings.tsx:88`, `src/parser.ts:19`, `src/parser.ts:26`, `src/schema-to-example.ts:42`
- Trigger: Save a preset with syntactically valid JSON that is not a JSON Schema object, or omit nested `properties`/`items` expected by XML normalization and example generation.
- Workaround: Restore the default preset from settings and edit from that known-good shape.

## Security Considerations

**Generated tracker data can become executable HTML:**
- Risk: Tracker templates are compiled with `noEscape: true`, rendered with LLM/user-controlled data, and assigned through `innerHTML`. A model response or manually edited tracker value can inject HTML and script-capable attributes into the SillyTavern page.
- Files: `src/index.tsx:85`, `src/index.tsx:95`, `src/index.tsx:416`, `src/config.ts:326`, `src/components/Settings.tsx:214`
- Current mitigation: Rendering is wrapped in a try/catch during generation, but there is no sanitization step and escaping is disabled intentionally.
- Recommendations: Sanitize rendered HTML before insertion, remove `noEscape: true` for data-bearing templates, restrict editable template capabilities, or render known fields with DOM APIs instead of raw `innerHTML`.

**Sensitive chat/model content is logged on parser failures:**
- Risk: Failed JSON/XML parsing logs the full raw model response to the browser console.
- Files: `src/parser.ts:62`, `src/parser.ts:63`
- Current mitigation: None detected beyond local browser console scope.
- Recommendations: Log a short parse error and response length; gate raw content logging behind an explicit debug setting.

**No detected secret files in the repo root scan:**
- Risk: Not detected.
- Files: `.gitignore`, `package.json`, `src/index.tsx`
- Current mitigation: `.gitignore` excludes `.env`, and no `.env*`, credential, key, or secret files were detected during the mapper scan.
- Recommendations: Keep secret files out of `.planning/`, `dist/`, and extension settings; never store API keys in presets or prompts.

## Performance Bottlenecks

**Long chats can cause expensive prompt construction and tracker context injection:**
- Problem: `includeLastXMessages` defaults to `0`, meaning prompt construction starts at the beginning of the chat; `includeWTrackerMessages()` clones the message list and scans backward for tracker entries.
- Files: `src/index.tsx:383`, `src/index.tsx:385`, `src/index.tsx:613`, `src/index.tsx:617`, `src/config.ts:426`, `src/config.ts:427`
- Cause: `structuredClone(messages)` copies the full prompt message array, and repeated backward scans add work proportional to chat length and included tracker count.
- Improvement path: Cap defaults for long chats, pass tracker context as a separate prompt fragment where possible, and avoid cloning full message objects when only content insertion is needed.

**Auto mode can start many concurrent generation requests:**
- Problem: `pendingRequests` prevents duplicate work for the same message id but does not queue or limit requests across different message ids.
- Files: `src/index.tsx:27`, `src/index.tsx:575`, `src/index.tsx:644`, `src/index.tsx:828`, `src/index.tsx:832`
- Cause: Message-rendered events call `generateTracker(messageId)` directly when auto mode is enabled.
- Improvement path: Add a bounded queue keyed by chat and message id, serialize generation per chat/profile, and coalesce render events that fire during chat loading.

**Chat-change rendering scans and touches every message:**
- Problem: `CHAT_CHANGED` loops through every message and renders trackers immediately; render failures can also trigger a save.
- Files: `src/index.tsx:110`, `src/index.tsx:836`, `src/index.tsx:840`, `src/index.tsx:852`
- Cause: The event handler uses full-chat eager rendering rather than visible-message or dirty-message rendering.
- Improvement path: Render lazily for visible messages, debounce chat-change work, and avoid saving during render-only passes.

## Fragile Areas

**SillyTavern DOM integration depends on private selectors and template structure:**
- Files: `src/index.tsx:71`, `src/index.tsx:374`, `src/index.tsx:531`, `src/index.tsx:754`, `src/index.tsx:790`, `src/index.tsx:793`, `src/styles/main.scss:28`, `templates/buttons.html`
- Why fragile: The extension injects buttons into `#rm_group_members`, `#message_template .mes_buttons .extraMesButtons`, `.mes[mesid]`, and `#extensionsMenu`; SillyTavern DOM changes can remove controls or break event routing.
- Safe modification: Keep selector logic centralized and add smoke tests or manual checks for group member buttons, message toolbar button injection, tracker controls, and extension menu popup after SillyTavern updates.
- Test coverage: No automated DOM integration tests detected.

**Group character tracking uses avatar/name heuristics:**
- Files: `src/index.tsx:149`, `src/index.tsx:191`, `src/index.tsx:230`, `src/index.tsx:332`, `src/index.tsx:347`
- Why fragile: Character identity is inferred from normalized avatar file names or display names; duplicate names, changed avatar paths, missing avatars, and legacy tracker payloads can point a tracker at the wrong group member.
- Safe modification: Prefer stable SillyTavern character ids when available, persist both id and display metadata, and keep legacy fallback read-only until migrated.
- Test coverage: No unit tests detected for `normalizeAvatarRef()`, `getGroupMessageCharacterKey()`, `getTrackerEntry()`, or legacy fallback matching.

**Global mutable extension state has no lifecycle cleanup:**
- Files: `src/index.tsx:25`, `src/index.tsx:26`, `src/index.tsx:27`, `src/index.tsx:754`, `src/index.tsx:780`, `src/index.tsx:793`, `src/index.tsx:828`, `src/index.tsx:865`
- Why fragile: The module registers jQuery handlers, document click handlers, MutationObserver callbacks, eventSource listeners, and `globalThis.wtrackerGenerateInterceptor` without teardown or duplicate-registration guards.
- Safe modification: Add an idempotent initializer, store unsubscribe/disconnect handles, and guard against multiple loads before appending DOM nodes or registering listeners.
- Test coverage: No initialization or reload tests detected.

## Scaling Limits

**Default generation settings can consume large model budgets:**
- Current capacity: Default `maxResponseToken` is `16000`, default `includeLastXMessages` is `0` for all prior messages, and default `includeLastXWTrackerMessages` is `1`.
- Files: `src/config.ts:415`, `src/config.ts:426`, `src/config.ts:427`, `src/index.tsx:613`, `src/index.tsx:617`
- Limit: Long chats and large schemas can produce high-latency and high-cost generation requests; some APIs may reject the prompt or max-token setting.
- Scaling path: Use conservative defaults, validate per-profile token limits, and show estimated prompt/response size before generation.

**Stored tracker payloads grow with every tracked message:**
- Current capacity: Each tracked chat message stores full parsed tracker data and the full HTML template used to render it.
- Files: `src/index.tsx:704`, `src/index.tsx:705`, `src/config.ts:326`
- Limit: Large custom schemas and templates increase chat file size linearly with every tracked message.
- Scaling path: Store a template preset key plus data for current formats, keep full HTML only for migrated legacy entries, and add a migration path keyed by `formatVersion`.

## Dependencies at Risk

**Runtime depends on SillyTavern and `sillytavern-utils-lib` internals:**
- Risk: Imports and globals expose volatile host-app details such as `characters`, `selected_group`, `CONNECT_API_MAP`, `eventSource`, and `SillyTavern.getContext()`.
- Impact: SillyTavern or `sillytavern-utils-lib` API changes can break generation, group matching, settings rendering, or event hooks.
- Migration plan: Wrap host APIs in a small adapter module and make all direct SillyTavern/global references go through it.

**Webpack externalization uses a broad path heuristic:**
- Risk: Any import request containing `../../..` is externalized as a module import.
- Impact: A future relative import can be accidentally excluded from the bundle and fail at runtime.
- Migration plan: Replace the heuristic in `webpack.config.cjs` with explicit externals for known SillyTavern host modules.

**React is a peer dependency while SillyTavern supplies the runtime page:**
- Risk: `src/components/Settings.tsx` assumes React and ReactDOM are compatible with the extension bundle and page.
- Impact: Host React version mismatches can affect settings rendering.
- Migration plan: Keep React peer ranges aligned with the host environment and smoke-test `renderReactSettings()` after dependency bumps.

## Missing Critical Features

**HTML sanitization for tracker rendering:**
- Problem: User-editable templates and model-generated field values are rendered as raw HTML.
- Blocks: Safe sharing of schema presets and safe rendering of untrusted model output.

**Schema validation and migration:**
- Problem: Settings contain `version` and `formatVersion`, and old default schema constants exist, but no local validation or migration path is visible.
- Blocks: Safe evolution of tracker storage, preset shape, and chat-level data across releases.

**Automated release/build verification:**
- Problem: `manifest.json` points at committed `dist/`, but no CI config or local check enforces that `dist/` matches `src/`.
- Blocks: Reliable source-to-extension release flow.

## Test Coverage Gaps

**No automated tests detected:**
- What's not tested: Parser behavior, schema-to-example generation, tracker storage mutation, render failure handling, chat-level schema selection, group character matching, generation interception, and settings updates.
- Files: `jest.config.mjs`, `package.json`, `src/index.tsx`, `src/parser.ts`, `src/schema-to-example.ts`, `src/components/Settings.tsx`
- Risk: Regressions in prompt parsing, metadata persistence, and SillyTavern event handling can ship unnoticed.
- Priority: High

**Security-sensitive rendering has no regression tests:**
- What's not tested: Escaping/sanitization behavior for LLM-provided strings rendered through Handlebars and `innerHTML`.
- Files: `src/index.tsx:85`, `src/index.tsx:95`, `src/config.ts:326`, `src/components/Settings.tsx:214`
- Risk: A malicious or malformed model response can inject HTML into the page without any test catching the behavior.
- Priority: High

**Data-loss paths have no regression tests:**
- What's not tested: The `CHAT_CHANGED` path that deletes tracker entries on render errors and the generate-time rollback path.
- Files: `src/index.tsx:725`, `src/index.tsx:727`, `src/index.tsx:843`, `src/index.tsx:846`
- Risk: User tracker data can be removed during render or generation error handling without automated coverage.
- Priority: High

---

*Concerns audit: 2026-06-18*
