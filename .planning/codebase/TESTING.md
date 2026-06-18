# Testing Patterns

**Analysis Date:** 2026-06-18

## Test Framework

**Runner:**
- Jest 29.7.0 is configured through `jest.config.mjs` and `package.json`.
- The `package.json` test script runs Jest through Node ESM support: `node --experimental-vm-modules node_modules/jest/bin/jest.js`.
- `jest.config.mjs` uses `preset: 'ts-jest'`, `testEnvironment: 'node'`, `extensionsToTreatAsEsm: ['.ts']`, and a `.js` import mapper for NodeNext TypeScript imports.
- `babel.config.json` provides Babel presets for current Node and TypeScript, but Jest transformation is configured through `ts-jest` in `jest.config.mjs`.
- Current Jest discovery with `npm test -- --listTests` prints no test paths.

**Assertion Library:**
- Jest `expect` is available through Jest from `package.json`.
- No existing assertions are detected because no `*.test.*` or `*.spec.*` files are present.

**Run Commands:**
```bash
npm test                    # Run all Jest tests through package.json
npm test -- --watch         # Run Jest watch mode
npm test -- --coverage      # Generate Jest coverage output
npm test -- --listTests     # Show discovered tests; currently prints no paths
```

## Test File Organization

**Location:**
- No current test files are detected by `rg --files -g '*.test.*' -g '*.spec.*'`.
- `copilot-instructions.md` says tests should mirror source structure. Use source-adjacent mirrored tests for new coverage, such as `src/parser.test.ts`, `src/schema-to-example.test.ts`, and `src/components/Settings.test.tsx`.

**Naming:**
- Use Jest default names: `*.test.ts`, `*.test.tsx`, `*.spec.ts`, or `*.spec.tsx`.
- Prefer `*.test.ts` and `*.test.tsx` for consistency with TypeScript source under `src/`.

**Structure:**
```text
src/
  parser.ts
  parser.test.ts
  schema-to-example.ts
  schema-to-example.test.ts
  components/
    Settings.tsx
    Settings.test.tsx
  hooks/
    useForceUpdate.ts
    useForceUpdate.test.ts
```

## Test Structure

**Suite Organization:**
Actual suites are not detected. Use this project-compatible Jest shape for new tests:

```typescript
import { parseResponse } from './parser.js';

describe('parseResponse', () => {
  it('parses JSON responses from fenced code blocks', () => {
    const result = parseResponse('```json\n{"name":"Ada"}\n```', 'json');

    expect(result).toEqual({ name: 'Ada' });
  });
});
```

**Patterns:**
- Keep pure utility tests focused on inputs and outputs for `src/parser.ts` and `src/schema-to-example.ts`.
- Put setup for globals and mocks inside the test file until it is reused by multiple files.
- Restore or delete modified globals in `afterEach` when testing code that touches `globalThis.SillyTavern`, jQuery, DOM APIs, or `globalThis.wtrackerGenerateInterceptor` from `src/index.tsx`.
- Use explicit assertions for thrown user-facing errors from `src/parser.ts`.

## Mocking

**Framework:** Jest

**Patterns:**
No current mocks are detected. Because the project is ESM-oriented, use Jest ESM mocks before dynamically importing modules that consume the mock:

```typescript
jest.unstable_mockModule('sillytavern-utils-lib/config', () => ({
  characters: [],
  name1: 'User',
  selected_group: null,
  st_echo: jest.fn(),
}));

const { parseResponse } = await import('./parser.js');
```

For global SillyTavern access, create the global in `beforeEach`:

```typescript
beforeEach(() => {
  (globalThis as any).SillyTavern = {
    getContext: jest.fn(() => ({
      chat: [],
      chatMetadata: {},
      eventSource: { on: jest.fn() },
      saveMetadataDebounced: jest.fn(),
    })),
  };
});

afterEach(() => {
  delete (globalThis as any).SillyTavern;
  jest.restoreAllMocks();
});
```

**What to Mock:**
- Mock `sillytavern-utils-lib` and `sillytavern-utils-lib/config` when testing `src/index.tsx` or `src/components/Settings.tsx`.
- Mock `SillyTavern.getContext()` for code that reads chat state, metadata, event sources, popups, or template rendering.
- Mock `Generator` and `buildPrompt` from `sillytavern-utils-lib` for tracker generation paths in `src/index.tsx`.
- Mock popup confirmations before testing delete, restore, or save flows in `src/index.tsx` and `src/components/Settings.tsx`.

**What NOT to Mock:**
- Do not mock `fast-xml-parser` for normal `parseResponse` coverage in `src/parser.ts`; test real XML parsing behavior.
- Do not mock `schemaToExample` internals in `src/schema-to-example.ts`; test generated JSON and XML output directly.
- Do not mock simple default settings from `src/config.ts` unless a test needs a deliberately reduced schema.

## Fixtures and Factories

**Test Data:**
No shared fixtures are detected. Start with inline fixtures for parser and schema utility tests:

```typescript
const simpleSchema = {
  type: 'object',
  properties: {
    name: { type: 'string', description: 'Character name' },
    inventoryItems: { type: 'array', items: { type: 'string' } },
  },
};
```

**Location:**
- Shared fixtures are not currently present.
- Keep one-off fixtures inside the relevant test file, such as `src/parser.test.ts`.
- If chat or SillyTavern factories become shared, add a dedicated helper under `src/test-utils/` and import it from tests.

## Coverage

**Requirements:** None enforced. `jest.config.mjs` does not define `collectCoverage`, `coverageThreshold`, or coverage reporters. `package.json` does not define a coverage script.

**View Coverage:**
```bash
npm test -- --coverage
```

## Test Types

**Unit Tests:**
- Best current fit under `testEnvironment: 'node'`.
- Prioritize pure modules first: `src/parser.ts`, `src/schema-to-example.ts`, and simple settings defaults in `src/config.ts`.
- Cover invalid JSON/XML handling in `src/parser.ts` because the module throws normalized errors after logging diagnostics.

**Integration Tests:**
- Not currently present.
- Testing `src/index.tsx` requires mocks for SillyTavern globals, jQuery-style DOM access, `sillytavern-utils-lib`, `handlebars`, and async Generator behavior.
- DOM-heavy tests for `src/index.tsx`, `src/components/Settings.tsx`, and `src/hooks/useForceUpdate.ts` need a DOM-capable Jest environment such as `jsdom`; the current `jest.config.mjs` uses `node`.

**E2E Tests:**
- Not used. No Playwright, Cypress, or browser automation config is detected in `package.json` or the project root.

## Common Patterns

**Async Testing:**
```typescript
await expect(asyncOperation()).resolves.toEqual(expectedValue);
await expect(failingOperation()).rejects.toThrow('Tracker generation failed');
```

Use fake implementations for SillyTavern popups and Generator callbacks when testing async flows from `src/index.tsx`.

**Error Testing:**
```typescript
expect(() => parseResponse('not-json', 'json')).toThrow('Model response is not valid JSON.');
expect(() => parseResponse('<root>', 'xml')).toThrow();
```

Prefer asserting the normalized error messages thrown by `src/parser.ts` instead of low-level parser messages.

---

*Testing analysis: 2026-06-18*
