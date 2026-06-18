# Technology Stack

**Analysis Date:** 2026-06-18

## Languages

**Primary:**
- TypeScript 5.7.3 - Main extension source in `src/index.tsx`, `src/config.ts`, `src/parser.ts`, `src/schema-to-example.ts`, `src/components/Settings.tsx`, and `src/hooks/useForceUpdate.ts`.
- TSX / React JSX - Settings UI in `src/components/Settings.tsx` and React mount code in `src/index.tsx`.
- SCSS - Extension styles in `src/styles/main.scss`, compiled to `dist/style.css` by the `build` and `dev` scripts in `package.json`.
- JSON - Extension, compiler, formatter, Jest, Babel, and package metadata in `manifest.json`, `tsconfig.json`, `.prettierrc.json`, `jest.config.mjs`, `babel.config.json`, `package.json`, and `package-lock.json`.

**Secondary:**
- HTML / Handlebars templates - SillyTavern-rendered templates in `templates/buttons.html` and `templates/modify_schema_popup.html`; runtime tracker templates are stored in `src/config.ts`.
- CommonJS JavaScript - Build configuration in `webpack.config.cjs`.
- Generated JavaScript and CSS - Built extension artifacts in `dist/index.js` and `dist/style.css`.

## Runtime

**Environment:**
- Browser runtime inside the SillyTavern extension host. `manifest.json` loads `dist/index.js` and `dist/style.css`; `src/index.tsx` uses browser DOM APIs, the global `SillyTavern` object, global jQuery `$`, and SillyTavern event hooks.
- Node.js for build and tests. No `.nvmrc` or explicit `engines` entry is present in `package.json`; `@types/node` is pinned at `^22.13.1` and `babel.config.json` targets the current Node runtime for tests.

**Package Manager:**
- npm - `package-lock.json` is present with `lockfileVersion: 3`.
- Lockfile: present at `package-lock.json`.

## Frameworks

**Core:**
- SillyTavern extension system - `manifest.json` declares `js`, `css`, `loading_order`, and `generate_interceptor`; `src/index.tsx` calls `SillyTavern.getContext()` and uses host context APIs.
- React `^19.1.1` and React DOM `^19.1.1` - Declared as peer dependencies in `package.json`; `src/index.tsx` uses `createRoot`, and `src/components/Settings.tsx` defines the settings component.
- `sillytavern-utils-lib` `^1.0.64` - Core integration dependency in `package.json`; used in `src/index.tsx` for `buildPrompt`, `Generator`, event/type imports, config helpers, and in `src/components/Settings.tsx` for SillyTavern React components and `ExtensionSettingsManager`.
- Handlebars `^4.7.8` - Runtime template compilation in `src/index.tsx` for tracker rendering and prompt construction; default templates live in `src/config.ts`.
- `fast-xml-parser` `^5.2.5` - XML parsing for model responses in `src/parser.ts`.

**Testing:**
- Jest `^29.7.0` - Configured in `jest.config.mjs` and run through the `test` script in `package.json`.
- ts-jest `^29.2.5` - Configured as the Jest preset and transformer in `jest.config.mjs`.
- Babel Jest `^29.7.0` and Babel presets - Available in `package.json`; `babel.config.json` configures `@babel/preset-env` and `@babel/preset-typescript`.
- Test files: Not detected under the repo with `*.test.*` or `*.spec.*` globs.

**Build/Dev:**
- Webpack 5 - `webpack.config.cjs` builds `src/index.tsx` into `dist/index.js` as an output module. `package-lock.json` includes `node_modules/webpack` version `5.100.2`; `package.json` invokes `webpack` through the `build` and `dev` scripts.
- `webpack-cli` `^5.1.4` - Direct dev dependency in `package.json`.
- `babel-loader` `^9.1.3` and `ts-loader` `^9.5.2` - Chained in `webpack.config.cjs` for TypeScript, TSX, JavaScript, and JSX files.
- Sass `^1.83.4` - Compiles `src/styles/main.scss` into `dist/style.css` in `package.json` scripts.
- Terser Webpack Plugin `^5.3.10` - Production minimizer configured in `webpack.config.cjs`.
- Prettier `^3.4.2` - Formatter configured by `.prettierrc.json`; `package.json` includes the `prettify` script.
- `cross-env` `^7.0.3` - Sets `NODE_ENV` in the `dev` and `build` scripts in `package.json`.

## Key Dependencies

**Critical:**
- `sillytavern-utils-lib` `^1.0.64` - Provides the SillyTavern bridge used by `src/index.tsx` and `src/components/Settings.tsx`; new code should prefer this library for host-facing prompts, generation requests, settings, and UI controls.
- `react` `^19.1.1` and `react-dom` `^19.1.1` - Peer dependencies supplied by the SillyTavern host/runtime; do not bundle a separate React runtime unless the extension packaging model changes.
- `handlebars` `^4.7.8` - Renders user-editable tracker HTML and prompt templates in `src/index.tsx` and `src/config.ts`; invalid templates can affect message rendering.
- `fast-xml-parser` `^5.2.5` - Parses XML-mode model output in `src/parser.ts`; JSON mode uses native `JSON.parse`.

**Infrastructure:**
- `typescript` `^5.7.3` - Type-checking and transpilation configured by `tsconfig.json` and `webpack.config.cjs`.
- `webpack-cli` `^5.1.4` and Webpack 5.100.2 from `package-lock.json` - Bundles the extension entry point.
- `sass` `^1.83.4` - Builds extension styles.
- `jest` `^29.7.0`, `ts-jest` `^29.2.5`, `@types/jest` `^29.5.14` - Test stack.
- `@types/jquery` `^3.5.32` - Provides types for the global jQuery usage in `src/index.tsx`; jQuery is not imported by source modules.
- `@types/react` `^19.1.11` and `@types/react-dom` `^19.1.8` - React type packages for TSX in `src/components/Settings.tsx` and `src/index.tsx`.

## Configuration

**Environment:**
- Runtime configuration is host-driven through SillyTavern connection profiles selected in `src/components/Settings.tsx` via `STConnectionProfileSelect`.
- Extension defaults are defined in `src/config.ts` through `defaultSettings`, `DEFAULT_PROMPT`, `DEFAULT_SCHEMA_VALUE`, `DEFAULT_SCHEMA_HTML`, `DEFAULT_PROMPT_JSON`, and `DEFAULT_PROMPT_XML`.
- Per-extension settings are keyed by `EXTENSION_KEY` in `src/config.ts` and persisted through `ExtensionSettingsManager` in `src/components/Settings.tsx`.
- Per-chat and per-message data is stored through SillyTavern context objects in `src/index.tsx`, using `chatMetadata`, `message.extra`, `saveChat()`, and `saveMetadataDebounced()`.
- No `.env*` files are present in the repo root, and `.gitignore` excludes `.env`.
- Repo-local project skills are not detected; `.codex/skills` and `.agents/skills` are absent.

**Build:**
- `package.json` scripts:
  - `npm run dev` compiles SCSS with source maps and runs Webpack watch mode.
  - `npm run build` compiles SCSS without source maps and runs Webpack production mode.
  - `npm test` runs Jest through `node --experimental-vm-modules`.
  - `npm run prettify` runs Prettier over `src/**/*.ts`, `src/**/*.tsx`, and `templates/**/*.html`.
- `webpack.config.cjs`:
  - Entry: `src/index.tsx`.
  - Output: `dist/index.js` as an ES module library.
  - Externals: module requests containing `../../..` are externalized for SillyTavern host imports.
  - Source maps: enabled outside production.
  - Minification: enabled in production via `TerserPlugin`.
- `tsconfig.json`:
  - `target: ESNext`, `module: NodeNext`, `moduleResolution: nodenext`, `strict: true`, `jsx: react-jsx`.
  - `rootDir: ./src`, `outDir: ./dist`, includes `src/**/*` and `types/**/*`, excludes `node_modules` and `dist`.
- `jest.config.mjs` configures `ts-jest`, ESM TypeScript handling, and Node test environment.
- `babel.config.json` configures Babel for tests and TypeScript transforms.
- `.prettierrc.json` uses semicolons, single quotes, `tabWidth: 2`, `printWidth: 120`, and an HTML override with `tabWidth: 4` and double quotes.

## Platform Requirements

**Development:**
- Install dependencies with npm using `package-lock.json`.
- Build with Node.js capable of running modern ESM, Jest 29 with `--experimental-vm-modules`, TypeScript 5.7, Webpack 5, and Sass.
- The extension source assumes SillyTavern globals and host modules; browser runtime behavior cannot be fully exercised as a standalone web app from `src/index.tsx`.

**Production:**
- Deployment target is a SillyTavern third-party extension installed from the repository URL documented in `readme.md`.
- `manifest.json` expects committed or generated artifacts at `dist/index.js` and `dist/style.css`.
- The extension relies on the user's SillyTavern connection profiles for actual model/provider access; provider SDKs and API keys are not part of this repo.

---

*Stack analysis: 2026-06-18*
