# Coding Conventions

**Analysis Date:** 2026-06-18

## Naming Patterns

**Files:**
- Use feature-oriented names for source files under `src/`. Utility modules use lowercase or kebab-case names such as `src/parser.ts` and `src/schema-to-example.ts`.
- Use PascalCase filenames for React components such as `src/components/Settings.tsx`.
- Use `useX` naming for hook files and exported hook functions, as in `src/hooks/useForceUpdate.ts`.
- Use lowercase descriptive names for Sass and Handlebars templates, such as `src/styles/main.scss`, `templates/buttons.html`, and `templates/modify_schema_popup.html`.
- Treat `dist/index.js` and `dist/style.css` as build outputs from `webpack.config.cjs` and `package.json`; do not edit generated files directly.

**Functions:**
- Use camelCase for normal functions and handlers: `renderTracker`, `getTrackerEntry`, `ensureTrackerEntry`, `deleteTrackerEntry`, and `modifyChatMetadata` in `src/index.tsx`.
- Use PascalCase for React components, as in `WTrackerSettings` from `src/components/Settings.tsx`.
- Use action prefixes that communicate side effects: `get*` reads, `ensure*` creates or normalizes, `update*` mutates UI state, `delete*` removes data, and `render*` writes DOM.
- Use `handle*` names for React event handlers inside components, as in `handleSchemaPresetChange`, `handleSchemaPresetsListChange`, `handleSchemaValueChange`, and `handleSchemaHtmlChange` in `src/components/Settings.tsx`.

**Variables:**
- Use camelCase for locals and module singletons: `globalContext`, `generator`, `pendingRequests`, `incomingTypes`, `outgoingTypes`, and `settingsManager` in `src/index.tsx` and `src/components/Settings.tsx`.
- Use SCREAMING_SNAKE_CASE for stable storage keys, constants, default prompts, and default schema objects: `EXTENSION_KEY`, `DEFAULT_PROMPT`, `DEFAULT_SCHEMA_VALUE`, and `CHAT_MESSAGE_SCHEMA_VALUE_KEY` in `src/config.ts` and `src/index.tsx`.
- Use clear collection suffixes for sets, maps, and arrays: `ignoreSet`, `pendingRequests`, `schemaPresetItems`, `incomingTypes`, and `outgoingTypes`.

**Types:**
- Use PascalCase for exported interfaces, enums, and type aliases: `ExtensionSettings`, `Schema`, `PromptEngineeringMode`, `WTrackerEntry`, and `WTrackerStorage` in `src/config.ts` and `src/index.tsx`.
- Use uppercase enum members for `PromptEngineeringMode.NATIVE`, `PromptEngineeringMode.JSON`, and `PromptEngineeringMode.XML` in `src/config.ts`.
- Prefer specific TypeScript shapes for extension-owned data, as with `ExtensionSettings` in `src/config.ts`; dynamic SillyTavern and schema data currently use `any` in `src/index.tsx`, `src/parser.ts`, and `src/schema-to-example.ts`.

## Code Style

**Formatting:**
- Use Prettier from `package.json` with the repository settings in `.prettierrc.json`.
- TypeScript and TSX formatting uses semicolons, single quotes, 2-space indentation, and a 120-column print width from `.prettierrc.json`.
- HTML templates use the `.prettierrc.json` override: 4-space indentation and double quotes in `templates/buttons.html` and `templates/modify_schema_popup.html`.
- Run `npm run prettify` from `package.json` for `src/**/*.ts`, `src/**/*.tsx`, and `templates/**/*.html`.
- No formatter script covers `src/styles/main.scss`; keep Sass edits consistent with the nested selector style already in `src/styles/main.scss`.

**Linting:**
- ESLint and Biome are not detected. There is no `.eslintrc*`, `eslint.config.*`, or `biome.json` in the project root.
- TypeScript strictness is the main static quality gate. `tsconfig.json` enables `strict`, `isolatedModules`, `forceConsistentCasingInFileNames`, and `noFallthroughCasesInSwitch`.
- Project guidance lives in `copilot-instructions.md`; no repo-local skill files are present under `.codex/skills` or `.agents/skills`.

## Import Organization

**Order:**
1. Runtime and React imports first, as in `src/index.tsx` and `src/components/Settings.tsx`.
2. External package imports next, especially `sillytavern-utils-lib`, `handlebars`, `fast-xml-parser`, and React/SillyTavern component imports.
3. Local relative imports last, using the NodeNext `.js` import suffix for TypeScript files: `./config.js`, `./parser.js`, `./schema-to-example.js`, and `../hooks/useForceUpdate.js`.

**Path Aliases:**
- Not detected. `tsconfig.json` does not define `paths`; use relative imports inside `src/`.
- Preserve `.js` suffixes on local TypeScript ESM imports because `tsconfig.json` uses `module: "NodeNext"` and `webpack.config.cjs` maps `.js` requests to `.ts`, `.tsx`, `.js`, and `.jsx`.

## Error Handling

**Patterns:**
- Use guard clauses for missing DOM, message, setting, and tracker data, as in `renderTracker`, `getTrackerEntry`, `toggleWTrackerForMember`, and `generateTracker` in `src/index.tsx`.
- Use `try`/`catch` around parse and generation boundaries. `src/parser.ts` catches low-level JSON/XML failures and throws user-focused errors such as invalid JSON or invalid XML.
- Use `try`/`catch`/`finally` for async UI workflows that need cleanup. `generateTracker` in `src/index.tsx` always removes spinner classes in `finally`.
- Treat user cancellation separately from failures. `generateTracker` checks `error.name !== 'AbortError'` in `src/index.tsx`.
- Roll back persisted tracker data when rendering generated data fails. `generateTracker` deletes the tracker entry before surfacing the render error in `src/index.tsx`.
- For editable JSON text in `src/components/Settings.tsx`, invalid JSON is intentionally ignored until the textarea contains valid JSON. For saved tracker edits in `src/index.tsx`, invalid JSON is reported through `st_echo('error', ...)`.
- Current dynamic integration points use `as any` and `// @ts-ignore` in `src/index.tsx`; keep new suppressions narrowly scoped and prefer typed wrappers when SillyTavern data shapes are known.

## Logging

**Framework:** `console` plus SillyTavern `st_echo`

**Patterns:**
- Use `console.error` for developer diagnostics in `src/index.tsx` and `src/parser.ts`.
- Use `st_echo('success' | 'error' | 'info', message)` for user-visible extension feedback in `src/index.tsx`.
- Use SillyTavern confirmations before destructive or restorative actions: `globalContext.Popup.show.confirm` in `src/index.tsx` and `SillyTavern.getContext().Popup.show.confirm` in `src/components/Settings.tsx`.
- No centralized logging abstraction is detected. Keep logging local to the failing operation.

## Comments

**When to Comment:**
- Use section comments sparingly to separate large integration areas in `src/index.tsx`, such as Handlebars helpers, core logic, group member toggles, UI initialization, and the main entry point.
- Comment non-obvious behavior tied to SillyTavern or external APIs, such as skipping the current message during context insertion, preserving `<details>` state, fallback DOM lookup behavior, and Generator cancellation handling in `src/index.tsx`.
- Avoid comments that restate simple assignments. Existing useful comments explain why a fallback or cleanup exists.

**JSDoc/TSDoc:**
- Sparse JSDoc is used. `src/hooks/useForceUpdate.ts` documents the exported hook.
- Add TSDoc for reusable exported hooks and utilities when the function's behavior is not obvious from its name and types.
- Large constants such as prompt and schema templates in `src/config.ts` are self-describing through names and object structure rather than JSDoc.

## Function Design

**Size:** Prefer small pure helpers for parsing and transformations, as in `src/parser.ts`, `src/schema-to-example.ts`, and `src/hooks/useForceUpdate.ts`. Large SillyTavern orchestration currently lives in `src/index.tsx`; extract reusable logic into focused helpers when adding behavior.

**Parameters:** Use explicit TypeScript parameters for public helpers. Use an options object for optional behavior, as `getTrackerEntry(message, { characterKey, allowLegacyFallback })` does in `src/index.tsx`. Use callback updaters for settings mutations in `src/components/Settings.tsx`.

**Return Values:** Return explicit values from pure functions (`parseResponse`, `schemaToExample`, `getTrackerEntry`, `deleteTrackerEntry`). Use `void` for DOM mutation helpers such as `updateWTrackerMemberButton` and `updateAllWTrackerMemberButtons`. Use `Promise`-returning `async` functions for popup, generation, and initialization flows in `src/index.tsx`.

## Module Design

**Exports:** Use named exports. `src/config.ts` exports settings types, constants, defaults, prompts, and schemas. `src/parser.ts` exports `parseResponse`. `src/schema-to-example.ts` exports `schemaToExample`. `src/components/Settings.tsx` exports both `settingsManager` and `WTrackerSettings`. `src/index.tsx` is the side-effectful extension entry point and does not export application functions.

**Barrel Files:** Not used. Import directly from the owning module, such as `../config.js` or `../hooks/useForceUpdate.js`.

---

*Convention analysis: 2026-06-18*
