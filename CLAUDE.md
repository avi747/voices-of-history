# CLAUDE.md — Voices of History

## Project Overview

**Voices of History** is a single-page web application that lets users hold conversational text (and optional audio) chats with AI-simulated historical and Israeli figures. The entire frontend is a single `index.html` file (~56 KB). All AI logic and TTS are handled by a Google Apps Script (GAS) backend that calls external LLM/ElevenLabs APIs.

---

## Repository Structure

```
voices-of-history/
├── index.html          ← Entire application (HTML + CSS + JS in one file)
└── .git/
```

There are **no build tools, no package manager, no test runner, and no CI/CD configuration**. The project is deployed by serving `index.html` as a static file.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Vanilla JavaScript (ES5-style, no modules) |
| Styling | CSS3 (embedded `<style>` in `index.html`) |
| Markup | HTML5 |
| Web APIs | Fetch API, Web Audio API (`AudioContext`, `AnalyserNode`) |
| Web Fonts | Google Fonts: Cormorant Garamond, IM Fell English, Crimson Pro |
| Backend | Google Apps Script (GAS) — external, not in this repo |
| TTS | ElevenLabs (called server-side from GAS) |
| Build | None — no bundler, minifier, or transpiler |

---

## index.html Internal Structure

The file has three logical sections:

| Lines (approx.) | Content |
|---|---|
| 1–7 | HTML head, meta tags, Google Fonts links |
| 8–1119 | Embedded CSS (`<style>`) |
| 1120–1209 | HTML body — two screen containers (`#screen-select`, `#screen-chat`) |
| 1210–1910 | Embedded JavaScript (`<script>`) |

### Screen Layout

- **#screen-select** — Grid of persona cards. Each card shows portrait, name, era tag, and a short description.
- **#screen-chat** — Chat interface with: portrait sidebar (animated), chat header with language + audio toggles, message list, suggested-questions row, and message input.

---

## Key JavaScript Architecture

### Global State Variables

```javascript
var GAS_URL = "https://script.google.com/...";  // Backend endpoint (hardcoded)
var PERSONAS = [];               // Fetched from GAS on load
var currentPersona = null;       // Active persona object
var conversationHistory = [];    // In-memory; cleared on navigation back
var currentLanguage = "en";      // "en" | "he" | "fr" | "ru" | "es"
var audioEnabled = true;         // User toggle
var currentAudio = null;         // Active HTMLAudioElement
var blinkTimer = null;           // setTimeout handle for portrait blinking
var portraitFrames = [];         // Array of portrait image URLs
```

### Module-style groupings (no actual modules — all global scope)

| Function group | Purpose |
|---|---|
| `initPortrait`, `setFrameByAmplitude`, `resetToFirstFrame`, `setPortraitSpeaking`, `startBlinking`, `stopBlinking` | Portrait frame animation tied to audio amplitude |
| `fetchIntroAudio`, `toggleAudio`, `playAudioB64`, `wireAnalyser`, `runAmpLoop`, `pauseAmpLoop`, `stopAmplitudeAnimation`, `togglePlayPause`, `stopAudio` | Web Audio API + ElevenLabs audio playback and visualization |
| `setLanguage` | Switch UI language, update RTL for Hebrew, refresh suggested questions |
| `mapPersona` | Normalise raw GAS persona data to local shape |
| `loadPersonas`, `renderPersonaGrid`, `getInitials`, `makeAvatar` | Fetch and render persona selection screen |
| `openChat`, `goBack`, `appendMessage`, `showTyping`, `removeTyping`, `sendMessage`, `sendSuggested` | Chat screen logic |

---

## Backend API (GAS Endpoint)

Single URL; action dispatched via query parameter.

### `?action=personas`
Returns the list of all historical figures.
```json
{ "personas": [ <PersonaData>, ... ] }
```

### `?action=chat`
Parameters: `personaId`, `language`, `audio` (true/false), `messages` (JSON-encoded array).  
Returns:
```json
{ "reply": "string", "audio": "<base64 mp3 or null>" }
```

### `?action=tts`
Parameters: `personaId`, `text`.  
Returns:
```json
{ "audio": "<base64 mp3>" }
```

---

## Persona Data Shape

After `mapPersona()` normalises raw GAS data:

```javascript
{
  id: string,
  name: string,
  title: string,          // Role/occupation
  era: string,            // Historical period key
  eraLabel: string,       // Human-readable era string
  tag: string,            // Category label displayed on card
  tagClass: string,       // CSS class for tag colour
  desc: string,           // Short biography shown on card
  intro: string,          // Opening message sent as first assistant turn
  portrait: string,       // URL to default portrait image
  frames: string[],       // Up to 7 portrait image URLs for mouth animation
  system: string,         // System prompt for LLM (set on GAS side)
  voiceId: string,        // ElevenLabs voice ID
  suggestions: string[],  // Default English suggested questions
  suggestionsAll: {       // Per-language suggested questions
    en: string[], he: string[], fr: string[], ru: string[], es: string[]
  }
}
```

---

## Coding Conventions

### JavaScript

- **No ES6 modules** — everything is in global scope. Do not introduce `import`/`export`.
- **Function declarations** over arrow functions at the top level.
- **Fetch chains** use `.then()/.catch()` — do not convert to `async/await` without a clear reason (no Babel).
- **DOM access** via `document.getElementById()` and direct `innerHTML`/`textContent`.
- **Inline event handlers** (`onclick="fn()"`) are used in HTML alongside `addEventListener`. Prefer `addEventListener` for new code.
- **No comments** unless the reason is non-obvious. Do not add block-comment docstrings.

### CSS

- **Embedded** in `<style>` — do not create external `.css` files.
- **CSS Custom Properties** for all colours and theme values (defined in `:root`).
- Key design tokens:
  ```
  --parchment: #f4ead5   --ink: #1a1208     --sepia: #6b4f2a
  --gold: #c9973a        --red-wax: #8b2020 --blue-deep: #1a2e4a
  ```
- **Responsive** breakpoints use `max-width: 640px`.
- Use `clamp()` for responsive typography.
- Sections are separated with comment banners (`/* ===== SECTION ===== */`).

### HTML

- All app state lives in JavaScript — avoid encoding state in HTML attributes beyond what's already established.
- RTL support for Hebrew is applied dynamically by `setLanguage()` via `document.body.dir`.

---

## Language Support

Five languages are supported: `en`, `he`, `fr`, `ru`, `es`.

- `LANG_LABELS` maps label keys → per-language strings (declared near the top of the `<script>` block).
- `setLanguage(lang, btn, context)` updates all visible UI labels, switches RTL direction for Hebrew, and refreshes the suggested questions row.
- When adding a new UI string, add it to **all five** language keys in `LANG_LABELS`.

---

## Audio & Portrait Animation

Portrait animation is driven by real-time audio amplitude analysis:

1. `playAudioB64(b64)` decodes a base64 MP3 and creates an `<audio>` element.
2. `wireAnalyser(audioEl)` attaches a Web Audio `AnalyserNode`.
3. `runAmpLoop()` reads frequency data each animation frame and calls `setFrameByAmplitude(amp)`.
4. `setFrameByAmplitude(amp)` selects one of the `portraitFrames[]` images based on amplitude threshold (7 frames at most).
5. On audio end, `resetToFirstFrame()` restores the neutral portrait.

The `doBlink()` function exists but is currently disabled (timer is set to null).

---

## Development Workflow

### Running locally

No build step required. Open `index.html` directly in a browser:

```bash
# Option 1 — file:// protocol (may block CORS for GAS requests)
open index.html

# Option 2 — local server (recommended)
npx serve .          # or: python3 -m http.server 8080
# then open http://localhost:8080
```

> **Note**: The app makes `fetch()` calls to `script.google.com`. These will work from any origin because GAS sets permissive CORS headers on deployed web apps.

### Making changes

All edits are made inside `index.html`. The three logical sections (CSS, HTML, JS) are clearly separated in the file.

- CSS changes: edit the `<style>` block.
- HTML structure changes: edit the body section between the style and script tags.
- Logic changes: edit the `<script>` block.

### Testing

There is no automated test suite. Manually test:

1. Persona grid loads and displays cards.
2. Clicking a persona opens the chat screen with the intro message.
3. Sending a message returns a reply.
4. Language toggle updates all labels, including RTL for Hebrew.
5. Audio toggle enables/disables TTS playback.
6. Back button returns to persona grid and resets state.

### Git workflow

- Development branch: `claude/claude-md-docs-JHpkI`
- Commits go to `main` after review.
- Commit messages follow the pattern: `Update index.html` (existing convention) or a short imperative description for significant changes.
- Push: `git push -u origin <branch>`

---

## Constraints & Known Limitations

- **Single-file architecture**: The entire app must remain in `index.html`. Do not split into separate files unless explicitly asked.
- **No build tooling**: Do not introduce webpack, Vite, Rollup, or any bundler without an explicit decision to restructure the project.
- **No framework**: Do not introduce React, Vue, Svelte, etc. without explicit direction.
- **GAS URL is hardcoded** — this is intentional for a public static SPA. Do not move it to a `.env` file without a corresponding server infrastructure change.
- **Conversation history is ephemeral** — it lives only in `conversationHistory[]` and is cleared when the user navigates back. This is by design.
- **No authentication** — the app is fully public; do not add auth flows without explicit direction.
- **ES5-compatible JS** — avoid ES6+ features that aren't already used (arrow functions in callbacks are fine; ES6 modules, classes, optional chaining are not in use).

---

## Security Notes

- Text inserted into the DOM uses `.textContent` (safe from XSS) for message content.
- The GAS URL is public and intentional for a client-side-only app.
- Do not introduce `eval()`, `innerHTML` for user content, or dynamic `<script>` injection.

---

## File Reference

| Symbol | Location (approx.) | Purpose |
|---|---|---|
| `GAS_URL` | JS line ~1211 | Backend endpoint |
| `PERSONAS` | JS line ~1212 | Loaded persona list |
| `LANG_LABELS` | JS line ~1218 | All UI strings, all languages |
| `loadPersonas()` | JS line ~1623 | Initial data fetch |
| `mapPersona()` | JS line ~1575 | Raw → local persona shape |
| `renderPersonaGrid()` | JS line ~1650 | Render selection screen |
| `openChat()` | JS line ~1714 | Initialise chat screen |
| `sendMessage()` | JS line ~1830 | Send user message to GAS |
| `playAudioB64()` | JS line ~1380 | Decode + play base64 audio |
| `wireAnalyser()` | JS line ~1430 | Hook Web Audio to audio element |
| `runAmpLoop()` | JS line ~1460 | Animation frame loop |
| `setLanguage()` | JS line ~1336 | Language switcher |
