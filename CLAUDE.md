# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`tennis-pong.html` is a single self-contained HTML file (markup, CSS, and JavaScript all inline) implementing a Pong-style tennis game with real tennis scoring. There is no build system, package manager, linter, or test suite — the entire project is this one file, opened directly in a browser.

## Commands

- **Run/preview:** open the file directly — `open tennis-pong.html` (macOS) or drag it into any browser. There is no dev server, bundler, or watch step.
- **No build, lint, or test commands exist.** Verify changes by opening the file in a browser and playing; for headless verification, use a headless browser (e.g. `google-chrome --headless=new --screenshot=out.png file:///path/to/tennis-pong.html`) and inspect the resulting screenshot, since there's no automated test harness.

## Architecture

Everything lives inside one `<script>` IIFE at the bottom of `tennis-pong.html`. The DOM is split into two rendering surfaces that stay in sync each frame:

- **HTML/CSS scoreboard** (`#scoreboard` and its row/column spans) — a broadcast-style score panel updated via direct DOM writes in `renderScore()`. It shows per-set game counts (`setEls`), the live point score (`pointEls`), a serve indicator dot (`serveEls`), and a pressure flag ("Break point" / "Set point" / "Match point") computed by `pressureLabel()`.
- **`<canvas id="court">`** — everything physical (court, racquets, ball, particles) is drawn here at a fixed logical resolution (`W=980, H=480`), scaled by `DPR` for crisp rendering on high-DPI screens. CSS then scales the canvas element back down to fit the layout.

Key subsystems, in the order they appear in the script:

1. **Court geometry** (`court` object) — derived once from real tennis court proportions (singles/doubles alley ratio, service-line distance) so all court lines are computed from a few constants rather than hardcoded pixels.
2. **Pre-rendered court layer** (`buildCourt()` → offscreen `courtLayer` canvas) — the static court (surface gradient, lines, net, wordmarks) is drawn once and blitted with `drawImage` each frame instead of being redrawn, since it never changes. Rebuilt once webfonts finish loading (`document.fonts.ready`) so painted text uses the right font.
3. **Tennis scoring state machine** (`match` object + `awardPoint`/`winGame`/`winSet`/`currentServer`) — implements full tennis rules: points → games → sets, deuce/advantage, alternating service each game, and a 7-point tiebreak at 6-6 games with correct serve rotation. `pointOver()` is the single entry point that feeds a point result into this state machine and triggers the corresponding UI callout.
4. **Game loop** (`update(dt, now)` / `draw()`, driven by `requestAnimationFrame`) — `dt` is normalized to 60fps units so speed is frame-rate independent. Ball movement is sub-stepped (`stepBall`) based on velocity so fast shots can't tunnel through a racquet in one frame.
5. **Racquet model** (`drawRacquet`) — the racquet head is generated procedurally as a point ring (`headPoints`) rather than a simple ellipse primitive, drawn at three insets (outer frame / mid frame / string clip) so the same point set serves multiple layers. Hit reactions (swing rotation, forward push, string vibration, glow) are spring-damped values (`springs()`) driven by `hitFx()` on every racquet contact.
6. **Effects** (`particles`, `rings`, `shake`, `edgeFlash`) — lightweight, self-expiring arrays updated in `update()` and drawn in `drawEffects()`; a new effect is added by pushing an object with a `life` field and letting the existing decay loop clean it up.

When changing gameplay constants (paddle size/speed, ball speed, AI behavior), they're grouped near the top of the relevant subsystem (e.g. `PADDLE_W/H/SPEED` near "Entities", tunable multipliers inline in `bounceOff`/`moveAI`) rather than centralized in one config block.
