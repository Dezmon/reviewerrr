# Sentence Reviewer

A single-page web app for quickly flipping through sentences in pasted text, keeping or discarding each via the keyboard, then copying the cleaned-up result.

## Architecture

One self-contained file: [index.html](index.html). All HTML, CSS, and JavaScript are inline. No build step, no dependencies, no package manager.

This is deliberate:
- Statically hostable on anything (S3, GitHub Pages, Netlify, even `file://`) by uploading one file.
- Previewable in any browser without `npm install`.
- The app is small enough (~370 lines total) that a build/bundle would add more friction than it removes.

If the project grows beyond what fits comfortably in one file, the natural next step is Vite + real TypeScript. Until then, keep everything in `index.html`.

## Code conventions

- Plain JavaScript with JSDoc type annotations (`@param`, `@type`, `@typedef`). Editors type-check from JSDoc; no `tsc` required.
- Single quotes for strings.
- No frameworks, no DOM libraries — plain `document.getElementById` + event listeners.
- No external network resources at runtime (no CDN scripts, web fonts, or analytics). The page must work offline.

## Running it

- **Preview**: open [index.html](index.html) in a browser, or serve the directory with any static server (`python -m http.server`, `npx serve`, etc.).
- **Deploy**: upload [index.html](index.html) to any static host.

The Clipboard API requires HTTPS or `localhost`. The Copy button has a `document.execCommand('copy')` fallback for `file://` and other non-secure contexts.

## Behavioral spec

See [SPECIFICATION.md](SPECIFICATION.md). That document is the contract this app implements. **If you're rewriting from scratch, work from the spec, not from the code.**

## What this project is not

- Not a precise sentence parser. Splitting is a simple regex (`.!?` + whitespace, or newlines). It will misclassify abbreviations and quoted speech. That's accepted — the workflow is rapid manual triage.
- Not a rich text editor. The textarea is plain text; formatting is not preserved.
- Not multi-user, not persistent. Refreshing the page loses state.
