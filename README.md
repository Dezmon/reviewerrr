# Sentence Triage

A single-page web app for quickly triaging a block of text one sentence at a time. Paste text, walk through each sentence with the keyboard, and copy the cleaned-up result.

## Usage

1. Paste text into the textarea.
2. Press **Start**. The textarea is replaced by a highlighted view of the parsed sentences.
3. Triage with the keyboard:

   | Key                          | Action                          |
   | ---------------------------- | ------------------------------- |
   | <kbd>→</kbd>                 | Keep current; advance           |
   | <kbd>←</kbd> / <kbd>Backspace</kbd> | Delete current; advance         |
   | <kbd>↓</kbd>                 | Undo last delete                |
   | <kbd>↑</kbd>                 | Jump to first kept sentence     |

4. Press **Copy** to put the kept text on the clipboard. Press **Reset** to edit again.

## Running it

It's one file with no dependencies.

- **Preview**: open [`index.html`](index.html) directly in a browser, or serve the directory with any static server (`python -m http.server`, `npx serve`, …).
- **Deploy**: upload [`index.html`](index.html) to any static host (S3, GitHub Pages, Netlify, …).

The Clipboard API requires HTTPS or `localhost`; on `file://` the Copy button falls back to `document.execCommand('copy')`.

## Project layout

- [`index.html`](index.html) — the entire app (HTML + CSS + JS, inline).
- [`SPECIFICATION.md`](SPECIFICATION.md) — behavioral contract. Rebuild from this, not from the code.
- [`CLAUDE.md`](CLAUDE.md) — notes for AI assistants working on the repo.

## Notes

- Sentence splitting is a simple regex on `.!?` + whitespace (or newlines). It will misclassify abbreviations and quoted speech; the workflow is rapid manual triage, not perfect parsing.
- No frameworks, no build step, no runtime network requests. Works offline.
- State is not persisted — refreshing the page clears it.
