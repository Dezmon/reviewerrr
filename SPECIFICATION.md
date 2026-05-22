# Sentence Triage — Specification

This document describes *what* the app does. Use it to rebuild the app from scratch in any language or framework and reach behavioral parity with the current implementation. Visual details are illustrative; behavior is the contract.

## Purpose

A keyboard-driven tool for triaging text one sentence at a time. The user pastes a block of text, then walks through it sentence-by-sentence using arrow keys to keep or discard each sentence. The remaining "kept" text can be copied to the clipboard.

## User interface

Three regions, top to bottom:

1. **Text area** — Plain `<textarea>` where the user pastes input. Spans the full page width minus body padding. Tall enough to show roughly 10 lines.
2. **Controls row** — `Start` button, four triage buttons (`Delete`, `Keep`, `Undo`, `First`), `Copy` button, and a status indicator (e.g. `3 / 7 kept`). The triage buttons mirror the keyboard arrow actions one-for-one, for touch users. They are disabled in the idle state and enabled in the active state.
3. **View** — A rendered panel showing the parsed sentences with the current one highlighted. Empty (with placeholder hint) until `Start` is pressed.

Above the text area: a title and a one-line hint listing the arrow-key bindings.

## States

The app has two states.

### Idle (initial state)
- Text area is editable.
- View region shows placeholder text ("Press Start to begin triaging sentences.").
- Status indicator is empty.
- Arrow keys have no effect.
- `Start` button is labeled "Start" with primary styling.

### Active
- Text area is hidden. Its value is still maintained internally (read-only) and reflects the current "kept" text, so Copy continues to work, but it is not visible to the user — the view region is the only display of the text being edited.
- View region shows each sentence as an inline span. The current sentence has a highlight (e.g. yellow background with a ring). Deleted sentences are hidden from view but retained in state so they can be undone.
- Status indicator shows `<kept count> / <total count> kept`.
- Arrow keys are bound (see Keyboard).
- `Start` button is relabeled "Reset" without primary styling.

## State model

```
state = {
  active: boolean
  sentences: Array<{ text: string, trailing: string, kept: boolean }>
  current: number   // index into sentences; -1 if no sentence highlighted
  history: number[] // stack of indices marked as deleted (most recent on top)
}
```

`trailing` is the whitespace/punctuation that followed the sentence in the original input — preserved so the rejoined output keeps the original formatting between kept sentences.

## Transitions

- **Start (idle → active)**: parse the textarea content into sentences. If zero non-empty sentences result, flash "No text to triage" in the status and stay idle. Otherwise: set `active = true`, `current = index of first kept sentence`, clear `history`, make textarea read-only and hide it, blur whatever element had focus.
- **Reset (active → idle)**: clear `sentences`, `history`, set `current = -1`, make textarea editable and visible again, clear the view and status.
- **Copy (any state)**: copy the textarea's current value to the clipboard. Flash "Copied" in the status. If the textarea is empty, flash "Nothing to copy" instead.

## Keyboard (active state only)

| Key                       | Action                                                                                                                              |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `→` (ArrowRight)          | Keep current sentence; advance highlight to the next kept sentence. If already at the last kept sentence, stay.                     |
| `←` (ArrowLeft)           | Mark current sentence as deleted; push its index onto the undo stack; advance to the next kept sentence. If no next, fall back to the previous kept sentence. If none remain, set `current = -1`. |
| `Backspace`               | Same as ArrowLeft.                                                                                                                  |
| `↓` (ArrowDown)           | Pop the most recent index from the undo stack; mark that sentence as kept again; move the highlight to it. No-op if the stack is empty. |
| `↑` (ArrowUp)             | Move the highlight to the first kept sentence. Does not change kept/deleted state or the undo stack. No-op if no sentences remain kept. |

Arrow keys are suppressed (no effect, no `preventDefault`) when focus is in a writable `<input>`, writable `<textarea>`, or `contenteditable` element. The active-mode textarea is read-only, so it does **not** swallow keys.

The triage buttons (`Delete`, `Keep`, `Undo`, `First`) invoke the same handlers as the corresponding arrow keys.

When a handled key is pressed, the handler should call `preventDefault()` so the browser doesn't also scroll or move the textarea caret.

## Sentence splitting

Input is split into sentences using **both** of these boundary types:

1. A sentence-ending punctuation mark (`.`, `!`, `?`) followed by one or more whitespace characters.
2. A run of one or more newlines (`\n+`).

Reference regex (used with `String.prototype.split` + capturing group so separators can be recovered):

```
/((?<=[.!?])\s+|\n+)/
```

For each emitted sentence, store the trailing separator. Discard any "sentence" whose text is empty or whitespace-only.

This heuristic imperfectly handles abbreviations (`Dr.`, `Mr.`, `e.g.`) and quoted speech. That is acceptable — the workflow is rapid manual triage, not perfect parsing.

## Textarea content during active state

The textarea always reflects the current kept text, computed as:

```
sentences.filter(s => s.kept).map(s => s.text + s.trailing).join('')
```

This means **Copy** always copies the current cleaned text, regardless of where the highlight is. Re-render the textarea after every state change (advance, delete, undo).

## View rendering

Render each sentence as a `<span class="sentence">`. Apply:
- `.current` to the span at index `state.current`.
- `.deleted` (hides via `display: none`) to spans with `kept === false`.

After each sentence span, if the sentence is kept and has a non-empty trailing, append a text node containing the trailing. Do NOT append trailing for deleted sentences (so the output reads cleanly without double-spaces).

After each render, scroll the current sentence into view if needed.

## Edge cases

| Situation                                | Behavior                                                                                       |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Empty/whitespace-only input on Start     | Flash "No text to triage", stay idle.                                                          |
| Empty textarea on Copy                   | Flash "Nothing to copy".                                                                       |
| All sentences deleted                    | `current = -1`, status shows `0 / N kept`. Undo still works.                                   |
| Undo with empty history                  | No-op.                                                                                          |
| Right at last kept sentence              | Highlight stays put. (Do not wrap.)                                                            |
| Clipboard API unavailable (`file://`, etc.) | Fall back to `document.execCommand('copy')`: temporarily clear `readonly`, `select()` the textarea, copy, restore `readonly`, clear selection. |
| Restart after Reset                      | Behaves identically to first Start — re-parse from current textarea content.                   |

## Status messages

Flash messages (~1.2s) override the kept/total counter:

- `"No text to triage"` — Start with empty input.
- `"Nothing to copy"` — Copy with empty textarea.
- `"Copied"` — successful Copy.

After the flash duration, the indicator returns to the active-state counter, or to empty if idle.

## Non-functional requirements

- **Static hosting**: the deployed artifact must be servable as static files with no server-side runtime.
- **No build step required for development**: source files should be directly editable and previewable in a browser.
- **No runtime dependencies**: no CDN scripts, no remotely-loaded fonts, no analytics. The page must work offline once loaded.
- **Light + dark theme**: respect `prefers-color-scheme`. No manual theme toggle.
- **Responsive**: works at narrow (phone) and wide (desktop) viewports. Text area and view scale with the page width.
- **Accessible**: status uses `aria-live="polite"`; keyboard is the primary interaction; controls are real `<button>` elements with visible focus.

## Visual style (illustrative, not contractual)

- System sans-serif font stack.
- Light background ~`#f7f7f8`; dark background ~`#1c1c1e`.
- Primary blue ~`#0a84ff` for the Start button in idle state.
- Current-sentence highlight: yellow background (~`#ffe066` light, `#ffd43b` dark) with a subtle outline ring.
- 8px border radii, 1px borders, subtle hover/active button states.

A rewrite is free to restyle as long as the current-sentence highlight is clearly visible and the controls are obvious.
