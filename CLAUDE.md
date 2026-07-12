# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

Two independent, single-file HTML projects with no build system, package manager, lint, or test tooling:

- **[racing-game/index.html](racing-game/index.html)** — *Turbo Drift*, an HTML5 Canvas arcade racing game. Zero external requests: fonts, art, and audio are all generated inline (audio uses WebAudio oscillators/noise buffers, not sound files).
- **[portfolio/index.html](portfolio/index.html)** — Aman Saraogi's personal portfolio site (dark-themed, single scrolling page), plus two assets it references: `profile.jpg` (hero photo) and `resume.pdf` (wired to the Download Resume button). Also zero external requests. Intended deployment target: GitHub Pages (`AmanDebs.github.io`).

## Running either project

Open the `index.html` directly in a browser, or — if `file://` access is restricted (e.g. sandboxed/automated browser contexts) — serve locally:

```
cd racing-game && python -m http.server 8934   # game  → http://localhost:8934/
cd portfolio  && python -m http.server 8952   # site  → http://localhost:8952/
```

There are no automated tests. Verify changes by exercising the page in a browser (or via browser automation — see the gotchas at the bottom).

## Portfolio architecture

Everything (HTML, CSS, JS) is inline in `portfolio/index.html`:

- **Design tokens** live in `:root` CSS variables at the top of the `<style>` block (`--bg`, `--accent` teal, `--surface`, `--ink*` text tiers, etc.) — change colors there, not per-rule.
- **Sections** are anchored `<section id>`s (`about`, `skills`, `projects`, `experience`, `education`) plus `<footer id="contact">`; the sticky nav links to these ids, and an IntersectionObserver in the bottom `<script>` highlights the active link. Adding a section means adding both the `<section id>` and a nav `<li>`.
- **Scroll-reveal** works by putting class `reveal` on a container; the IntersectionObserver adds `.visible` once. `prefers-reduced-motion` disables all animation.
- **Responsive breakpoints**: ≤820px stacks the hero/grids, ≤760px hides the nav links — keep those two in sync if you touch nav or hero layout.
- **Content is sourced from the resume + GitHub (user `AmanDebs`)** — project cards, experience bullets, and metrics mirror `resume.pdf`. If the resume is updated, both `resume.pdf` and the corresponding HTML sections need updating; don't invent numbers.
- The hero background is a decorative inline SVG line-chart motif (`#heroLine`, draw-in animated via stroke-dashoffset in the script).

## Racing game architecture

Everything lives inside a single IIFE in the `<script>` block at the bottom of `racing-game/index.html`, organized by `// ---------- Section ----------` comment markers (grep for these to navigate):

- **Setup** (~line 281) — canvas sizing/DPR scaling, responsive `resize()`.
- **Audio** (~line 310) — WebAudio-based SFX (`beep()`, `sfxCoin()`, `sfxCrash()`, etc.), no audio files.
- **Game constants / State** (~line 348–378) — lane geometry (`LANES`, `laneX()`), and the mutable game state (`score`, `combo`, `nitro`, `obstacles[]`, `coins[]`, `pickups[]`, `particles[]`, etc.) as module-level `let` variables closed over by every function.
- **Leaderboard** (~line 380) — top-10 high score table persisted to `localStorage` (`turboDrift_leaderboard`), separate from the single best-score (`turboDrift_best`). `qualifiesForLeaderboard()` / `addLeaderboardEntry()` / `renderLeaderboard()` are the entry points.
- **Input** (~line 437) — keyboard (arrows/WASD/space/P) and touch (on-screen buttons + canvas swipe) both funnel into `moveLane()`, `nitroHeld`, `brakeHeld`. Note the guard at the top of the `keydown` listener: when `document.activeElement` is an `<input>` (i.e. the leaderboard name field), game controls are suppressed and only `Enter` is handled (submits the leaderboard entry) — don't remove this or typing a name will also steer/pause the game.
- **Entity spawning** (~line 507) — `spawnObstacleWave()` always leaves at least one lane free so every run is theoretically dodgeable; difficulty scales via `difficultyFactor()` (ramps over ~60s of `elapsed` time).
- **Update** (~line 637) — single `update(dt)` function advances all game state (delta-time based, frame-rate independent); early-returns unless `state === 'playing'`.
- **Draw** (~line 787) — single `draw()` function does all canvas rendering (road, cars, particles; HUD overlays are plain DOM, not canvas).
- **Main loop** (~line 1017) — `requestAnimationFrame` loop calling `update()` → `draw()`.
- **Buttons** (~line 1030) — DOM event wiring for all overlay buttons (start/retry/pause/leaderboard).

### State machine

`state` is one of `'start' | 'playing' | 'paused' | 'gameover'`, gating both `update()` and which HTML overlay (`#startOverlay`, `#pauseOverlay`, `#gameOverOverlay`, `#leaderboardOverlay`) is visible. Overlays are toggled via the `.hidden` CSS class, not conditional rendering.

## Testing/automation gotchas

When driving these pages through a browser-automation tool for verification:

- `document.hidden` may report `true` even when the tab appears active, which pauses `requestAnimationFrame` entirely and also blocks OS-level synthetic key events from reaching the page. Workarounds used successfully: (1) dispatch events directly in-page via `window.dispatchEvent(new KeyboardEvent('keydown', {key:'ArrowRight', bubbles:true}))`; (2) for game logic without a live render loop, temporarily add a debug hook (e.g. `window.__debugStep(steps, dtStep)` calling `update()`/`draw()` directly) — then **remove it before finishing**, since it isn't part of the shipped game.
- Navigating the in-app browser to a PDF (e.g. `resume.pdf`) triggers a save dialog that can wedge that tab's screenshot capture indefinitely; verify PDF links with `curl -I` instead, and open a fresh tab if screenshots start timing out.
