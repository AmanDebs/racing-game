# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This repo contains **Turbo Drift**, a single-file HTML5 Canvas racing game at [racing-game/index.html](racing-game/index.html). There is no build system, package manager, or dependency tree — the entire game (HTML, CSS, JS) lives in one `.html` file with zero external requests (fonts, audio, and art are all generated inline; audio uses WebAudio oscillators/noise buffers, not sound files).

## Running the game

There is no build/lint/test tooling. To run it:

- Simplest: open [racing-game/index.html](racing-game/index.html) directly in a browser.
- If `file://` access is restricted (e.g. some sandboxed/automated browser contexts), serve it locally instead:
  ```
  cd racing-game && python -m http.server 8934
  ```
  then open `http://localhost:8934/`.

There are no automated tests. To verify a change works, run the game in a browser and exercise it manually (or via browser automation — see the note below).

## Architecture

Everything lives inside a single IIFE in the `<script>` block at the bottom of `index.html`, organized by `// ---------- Section ----------` comment markers (grep for these to navigate):

- **Setup** (~line 281) — canvas sizing/DPR scaling, responsive `resize()`.
- **Audio** (~line 310) — WebAudio-based SFX (`beep()`, `sfxCoin()`, `sfxCrash()`, etc.), no audio files.
- **Game constants / State** (~line 348–378) — lane geometry (`LANES`, `laneX()`), and the mutable game state (`score`, `combo`, `nitro`, `obstacles[]`, `coins[]`, `pickups[]`, `particles[]`, etc.) as module-level `let` variables closed over by every function.
- **Leaderboard** (~line 380) — top-10 high score table persisted to `localStorage` (`turboDrift_leaderboard`), separate from the single best-score (`turboDrift_best`). `qualifiesForLeaderboard()` / `addLeaderboardEntry()` / `renderLeaderboard()` are the entry points.
- **Input** (~line 437) — keyboard (arrows/WASD/space/P) and touch (on-screen buttons + canvas swipe) both funnel into `moveLane()`, `nitroHeld`, `brakeHeld`. Note the guard at the top of the `keydown` listener: when `document.activeElement` is an `<input>` (i.e. the leaderboard name field), game controls are suppressed and only `Enter` is handled (submits the leaderboard entry) — don't remove this or typing a name will also steer/pause the game.
- **Entity spawning** (~line 507) — `spawnObstacleWave()` always leaves at least one lane free so every run is theoretically dodgeable; difficulty scales via `difficultyFactor()` (ramps over ~60s of `elapsed` time).
- **Update** (~line 637) — single `update(dt)` function advances all game state (delta-time based, frame-rate independent); early-returns unless `state === 'playing'`.
- **Draw** (~line 787) — single `draw()` function does all canvas rendering (road, cars, particles, HUD overlays are plain DOM, not canvas).
- **Main loop** (~line 1017) — `requestAnimationFrame` loop calling `update()` → `draw()`.
- **Buttons** (~line 1030) — DOM event wiring for all overlay buttons (start/retry/pause/leaderboard).

### State machine

`state` is one of `'start' | 'playing' | 'paused' | 'gameover'`, gating both `update()` and which HTML overlay (`#startOverlay`, `#pauseOverlay`, `#gameOverOverlay`, `#leaderboardOverlay`) is visible. Overlays are toggled via the `.hidden` CSS class, not conditional rendering.

### Testing/automation gotcha

When driving this game through a browser-automation tool for verification, `document.hidden` may report `true` even when the tab appears active, which pauses `requestAnimationFrame` entirely and also blocks OS-level synthetic key events from reaching the page. Two workarounds used successfully during development:
1. Dispatch events directly in-page via `window.dispatchEvent(new KeyboardEvent('keydown', {key:'ArrowRight', bubbles:true}))` instead of relying on OS-level key injection.
2. For exercising game logic without a live render loop, temporarily add a debug hook (e.g. `window.__debugStep(steps, dtStep)` calling `update()`/`draw()` directly in a loop) — then **remove it before finishing**, since it isn't part of the shipped game.
