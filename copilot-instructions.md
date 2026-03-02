# Copilot Instructions for SillyTavern-WTracker

## Project Overview

SillyTavern-WTracker is a **SillyTavern extension** that tracks chat statistics with LLMs using connection profiles. This extension provides a minimalistic, non-intrusive approach to scene/chat tracking without the complexity or bugs of similar extensions.

**Key Differentiators:**
- Works seamlessly without requiring connection profile switches
- No "Prompt Maker" - uses straightforward JSON schema editing
- No chat summarization features (intentionally kept simple)
- Real-time tracker updates with customizable schemas

## Technology Stack

### Core Technologies
- **TypeScript** (v5.7.3) - Primary language, strict mode enabled
- **React** (v19.1.1) - UI components using React 19 with peer dependencies
- **SCSS** - Styling with Sass compilation
- **Webpack** - Bundling with Terser for production optimization
- **Jest** - Testing framework with Babel for TypeScript/React

### Key Dependencies
- `sillytavern-utils-lib` (v1.0.64) - Core utilities, components, and SillyTavern integration
- `handlebars` (v4.7.8) - Template rendering for tracker display
- `fast-xml-parser` (v5.2.5) - XML parsing for prompt engineering modes
- `jquery` (v3.7.1) - DOM manipulation (SillyTavern compatibility)

### Build Tools
- **Babel** - Transpilation with React, TypeScript, and env presets
- **Webpack CLI** - Build orchestration
- **Sass CLI** - SCSS compilation with source maps in development
- **Prettier** - Code formatting (configured for .ts, .tsx, .html files)

## Architecture & Design Patterns

### Extension Structure
```
src/
├── index.tsx              # Main extension entry point, event handlers, core logic
├── config.ts              # Settings interfaces, defaults, prompts, schemas
├── parser.ts              # Response parsing (JSON/XML/Native API modes)
├── schema-to-example.ts   # Convert JSON schemas to example responses
├── components/
│   └── Settings.tsx       # React settings UI with ExtensionSettingsManager
├── hooks/
│   └── useForceUpdate.ts  # Custom React hook for forcing re-renders
└── styles/
    └── main.scss          # Extension styles

templates/                 # HTML templates for UI elements
dist/                     # Build output (index.js, style.css)
```

### Key Concepts

#### 1. **Connection Profiles**
SillyTavern's system for managing different API configurations (Chat Completion, Text Completion). The extension tracks stats per profile without requiring manual switches.

#### 2. **Schema Presets**
User-defined JSON schemas that define what data to track. Each preset contains:
- `name`: Display name
- `value`: JSON schema object (draft-07)
- `html`: Handlebars template for rendering tracker data

#### 3. **Prompt Engineering Modes** (`PromptEngineeringMode`)
- **NATIVE**: Uses API's native structured output support
- **JSON**: Wraps schema in JSON markdown code block instructions
- **XML**: Uses XML format for models without JSON support

#### 4. **Extension Settings** (`ExtensionSettings`)
```typescript
{
  version: string                              // Extension version
  formatVersion: string                        // Settings format version
  profileId: string                            // Active connection profile ID
  maxResponseToken: number                     // Token limit for tracker generation
  autoMode: AutoModeOptions                    // When to trigger (RESPONSES, INPUT, BOTH, MANUAL)
  schemaPreset: string                         // Active schema preset key
  schemaPresets: Record<string, Schema>        // All available schemas
  prompt: string                               // Main tracker generation prompt
  includeLastXMessages: number                 // Context window size (0 = all)
  includeLastXWTrackerMessages: number         // Include previous tracker data (0 = none)
  promptEngineeringMode: PromptEngineeringMode // How to structure requests
  promptJson: string                           // Template for JSON mode
  promptXml: string                            // Template for XML mode
}
```

### Code Organization Patterns

#### Settings Management
Use `ExtensionSettingsManager` from `sillytavern-utils-lib`:
```typescript
export const settingsManager = new ExtensionSettingsManager<ExtensionSettings>(
  EXTENSION_KEY,
  defaultSettings
);
```

#### Message Metadata Storage
Tracker data is stored in chat message extras:
```typescript
message.extra[EXTENSION_KEY] = {
  value: trackerData,        // Parsed tracker object
  html: trackerHtmlSchema    // Handlebars template for rendering
};
```

#### Event Handling
The extension hooks into SillyTavern events:
- `GENERATION_AFTER_COMMANDS` - Process outgoing messages
- `MESSAGE_RECEIVED` - Process incoming AI responses
- `CHARACTER_MESSAGE_RENDERED` - Render tracker UI in messages
- `USER_MESSAGE_RENDERED` - Render tracker UI for user messages

## Development Guidelines

### TypeScript Configuration
- **Target**: ESNext
- **Module**: NodeNext with nodenext resolution
- **Strict mode**: Enabled
- **JSX**: react-jsx (React 17+ transform)
- **Output**: dist/ directory
- **Include type definitions** for jQuery, React, Node.js

### Code Style
- Use **Prettier** for formatting: `npm run prettify`
- Prefer **functional components** with hooks over class components
- Use **TypeScript strict types** - avoid `any` when possible
- Follow **React 19 patterns** (automatic memo, new hooks, async transitions)
- Use **?. optional chaining** and **?? nullish coalescing** for safety

### Naming Conventions
- **Constants**: SCREAMING_SNAKE_CASE (`EXTENSION_KEY`, `DEFAULT_PROMPT`)
- **Interfaces/Types**: PascalCase (`ExtensionSettings`, `Schema`)
- **Enums**: PascalCase with PascalCase values (`PromptEngineeringMode.NATIVE`)
- **Functions/Variables**: camelCase (`renderTracker`, `settingsManager`)
- **React Components**: PascalCase (`WTrackerSettings`)

### Building & Testing
```bash
npm run dev      # Development build with watch mode (SCSS + Webpack)
npm run build    # Production build (minified, no source maps)
npm run test     # Run Jest tests
npm run prettify # Format code with Prettier
```

### Important Development Notes

1. **React Components**: Use React 19 features (automatic memo, transitions)
2. **SillyTavern Integration**: Always use `sillytavern-utils-lib` components and utilities
3. **Extension Metadata**: Update `manifest.json` version when releasing
4. **Handlebars Templates**: Register custom helpers if needed (e.g., `join` helper)
5. **Error Handling**: Support fallback modes (JSON/XML) when Native API fails
6. **State Management**: Use `useForceUpdate` hook for settings-based re-renders

### Common Patterns

#### Updating Settings with Re-render
```typescript
const updateAndRefresh = useCallback(
  (updater: (currentSettings: ExtensionSettings) => void) => {
    const currentSettings = settingsManager.getSettings();
    updater(currentSettings);
    settingsManager.saveSettings();
    forceUpdate();
  },
  [forceUpdate]
);
```

#### Parsing Responses
```typescript
import { parseResponse } from './parser.js';

const result = parseResponse(
  responseText,
  settings.promptEngineeringMode,
  schemaValue
);
```

#### Building Prompts
```typescript
import { buildPrompt, Generator } from 'sillytavern-utils-lib';

const generator = new Generator();
const prompt = await buildPrompt(messages, {
  // ... prompt options
});
```

## Extension-Specific Context

### SillyTavern APIs Used
- `SillyTavern.getContext()` - Access global chat context
- `eventSource.on()` - Subscribe to SillyTavern events
- `callPopup()` - Display modal dialogs
- `getThumbnailUrl()` - Get character avatars
- Connection profile API for reading profile details

### Chat Message Structure
Messages have `.extra[EXTENSION_KEY]` for storing tracker data:
```typescript
{
  [CHAT_MESSAGE_SCHEMA_VALUE_KEY]: object,  // Parsed tracker data
  [CHAT_MESSAGE_SCHEMA_HTML_KEY]: string    // Handlebars template
}
```

### Metadata Storage
Chat metadata stores the active schema preset:
```typescript
chat.metadata[CHAT_METADATA_SCHEMA_PRESET_KEY] = schemaPresetKey;
```

## Testing

- Use **Jest** with `@babel/preset-typescript` and `@babel/preset-react`
- Test files should mirror source structure
- Mock SillyTavern globals when needed
- Run tests with: `npm test`

## Performance Considerations

1. **Structured Clone**: Use `structuredClone()` for deep copying objects
2. **Memoization**: Leverage React's automatic memoization in React 19
3. **DOM Queries**: Cache selectors when querying repeatedly
4. **Token Limits**: Respect `maxResponseToken` to avoid excessive API costs
5. **Context Window**: Use `includeLastXMessages` to limit prompt size

## Common Issues & Solutions

### API Errors with Structured Output
**Problem**: Model doesn't support native structured output
**Solution**: Change `promptEngineeringMode` to JSON or XML in settings

### Connection Profile Issues
**Problem**: Extension not working with Text Completion profiles
**Solution**: Ensure profile contains API, preset, model, and instruct settings

### Missing Tracker Data
**Problem**: Tracker not appearing in messages
**Solution**: Check autoMode setting and verify schema is valid

## File Extensions & Build Output

- **Source**: `.ts`, `.tsx`, `.scss`
- **Templates**: `.html` (Handlebars)
- **Output**: `dist/index.js`, `dist/style.css`
- **Configuration**: `.json`, `.cjs`, `.mjs` formats

## Dependencies to Avoid

- Do not add heavy ML libraries (keep extension lightweight)
- Avoid duplicate jQuery versions (use SillyTavern's bundled version)
- Don't bundle React (it's a peer dependency from SillyTavern)

## Future Enhancement Considerations

When suggesting features:
- Maintain simplicity (core principle of this extension)
- Avoid summarization features (intentionally excluded)
- Consider connection profile compatibility
- Think about API token costs
- Preserve non-intrusive user experience

---

**Extension Key**: Use `EXTENSION_KEY` constant from config.ts for all localStorage/metadata keys
**Manifest**: Update `manifest.json` for extension metadata changes
**Build Target**: SillyTavern v1.12.0+ (supports connection profiles)
