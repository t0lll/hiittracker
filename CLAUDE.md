# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A zero-dependency, single-file HIIT (High-Intensity Interval Training) timer web app. The entire application logic lives in `hiit timer.html` (note the space in the filename). `index.html` is just a redirect to it. The site is wrapped in Jekyll for GitHub Pages deployment.

## Running Locally

Open `hiit timer.html` directly in a browser — no build step required. Alternatively, serve it with Jekyll:

```bash
bundle exec jekyll serve
```

There are no npm packages, no build tooling, no linter, and no automated tests. Changes are verified by opening the file in a browser.

## Architecture

The app is a single-page application with three virtual "pages" toggled via CSS `.active` class:

1. **Setup page** — build a workout by adding blocks
2. **Review page** — preview the full workout plan
3. **Workout page** — execute the timer

### Workout State

`let workout = []` is the only persistent runtime state — an array of block objects. There is no localStorage or server. Workouts are shared/restored via URL query param `?w=` containing Base64-encoded JSON.

Encoding functions: `encodeWorkout(blocks)` → Base64 string, `decodeWorkout(str)` → block array. The encoded format uses compact arrays keyed by type prefix (`"t"`, `"r"`, `"a"`, `"i"`).

### Block Types

| Type | Prefix | Description |
|---|---|---|
| Timed | `"t"` | Single interval with a name, duration, and phase (`"work"`/`"rest"`) |
| Rep | `"r"` | Rep-based exercise with optional time limit |
| AMRAP | `"a"` | Timed circuit of exercises done for max rounds |
| Interval | `"i"` | Named set of repeating rounds, each with multiple timed steps |

### Execution Model

Before a workout starts, `buildSteps()` flattens all blocks into a linear `steps[]` array. The timer then walks this array sequentially. `loadStep(idx)` renders the current step; `tick()` runs every 1000 ms; `advance()` moves to the next step.

### Audio

Web Audio API only — no audio files. `beep(freq, dur)` creates oscillator tones. Beeps fire at countdown (3 s remaining), step transitions, and workout completion.

## Key Functions

- `init()` — bootstrap; decodes URL param if present
- `renderSetup()` — re-renders the block list on the setup page
- `openModal(type)` / `closeModal(id)` — modal management for adding blocks
- `addTimedBlock()`, `addRepBlock()`, `addAmrapBlock()`, `addIntervalBlock()` — read form inputs and push to `workout[]`
- `renderReview()` — builds the review page from `workout[]`
- `goToWorkout()` — calls `buildSteps()`, then starts the timer
- `updateShareUrl()` — encodes current workout into the URL bar

## Styling Conventions

- Dark theme: background `#0a0a0a`, accent yellow `#e8ff00`
- CSS custom properties defined at `:root` for all theme colors (`--accent`, `--work`, `--rest`, `--bg`, etc.)
- Fonts: "Bebas Neue" (display/headings), "DM Mono" (body/timers) via Google Fonts
- Phase colors: work = yellow (`--work`), rest = blue (`--rest`), rep = green (`--rep`), amrap = purple (`--amrap`)
