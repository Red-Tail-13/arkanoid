# Arkanoid — Project Documentation

## Overview

A fully self-contained browser Arkanoid clone built as a single HTML file (`arkanoid.html`). No dependencies, no build step — open in any browser and play. The game features 10 levels, 8 visual themes, a main menu, pause menu, save/load slots, and themed animated logos.

---

## File Structure

```
arkanoid.html          — entire game: HTML + CSS + JS in one file (~940 lines)
CLAUDE.md              — this document
```

Everything lives in one file intentionally. CSS is in `<style>`, JS in `<script>`, HTML in `<body>`. No modules, no bundler.

---

## Screens

The game uses a screen overlay system. All screens are `position:fixed` divs. Only the one with `.active` is shown (`display:flex`). The game canvas sits behind them in `#game-view`.

| Screen ID          | Purpose                                      |
|--------------------|----------------------------------------------|
| `screen-menu`      | Main menu (New Game, Load, Skins, Quit)      |
| `screen-pause`     | Pause menu (Resume, Save, Load, Skin, Menu)  |
| `screen-slots`     | Save/load slot picker (3 slots)              |
| `screen-skins`     | Theme selector (8 color themes)              |
| `screen-level`     | Level clear transition with score bonus      |
| `screen-end`       | Game over / all levels cleared               |

`showScreen(id)` activates a screen and hides `#game-view`. Passing `null` hides all screens and shows the game.

---

## Game Architecture

### Constants (`arkanoid.html` line ~756)
```js
W=480, H=520         // canvas dimensions
PW=90, PH=12, PY=484 // paddle width, height, Y position
BR=7                 // ball radius
COLS=10              // brick columns
BW=44, BH=16, BP=4   // brick width, height, padding
BOX=8, BOY=48        // brick grid offset X, Y
```

### Game State Variables
```js
paddle        // { x, y, w, h }
ball          // { x, y, dx, dy, speed, attached }
bricks        // array of brick objects
score         // current score (persists across levels)
lives         // 3 lives, persists across levels
currentLevel  // 1–10
gameRunning   // boolean, controls the game loop
animId        // requestAnimationFrame handle
particles     // explosion particle array
flashAlpha    // red flash intensity on life lost
```

### Game Loop (`loop()`)
Runs via `requestAnimationFrame`. Each frame:
1. `drawBg()` — clear + draw grid
2. `updateParticles()` + `drawParticles()`
3. `update()` — physics, collisions, win/lose checks
4. `drawBricks()`, `drawPaddle()`, `drawBall()`
5. `drawFlash()` — red overlay on life lost

### Physics (`update()`)
- **Paddle**: keyboard (`ArrowLeft`/`ArrowRight`) moves at 6px/frame; mouse sets position directly
- **Ball**: simple Euler integration (`ball.x += ball.dx`)
- **Wall bounces**: reflect `dx` or `dy` at canvas edges
- **Paddle hit**: angle calculated from hit position on paddle — center = straight up, edges = steep angle
- **Brick collision**: AABB overlap test, resolves bounce axis by smallest overlap
- **Ball lost**: decrements lives, flashes screen, resets ball to paddle

### Speed Scaling
```js
function levelSpeed(lvl){ return 4.5 + lvl * 0.35; }
// Level 1 → 4.85 px/frame, Level 10 → 8.0 px/frame
```

---

## Level System

### Layouts (`LEVELS` array, line ~733)
10 hand-crafted 6×10 grid layouts. Each cell value:
- `0` — empty (no brick)
- `1–6` — brick with color index `value - 1` mapped to `theme.brickRows[0–5]`

| Level | Layout Name     |
|-------|-----------------|
| 1     | Classic full    |
| 2     | Checkerboard    |
| 3     | Diamond         |
| 4     | Cross / Plus    |
| 5     | V-shape         |
| 6     | Vertical stripes|
| 7     | Pyramid         |
| 8     | Hourglass       |
| 9     | Maze            |
| 10    | Final chaos     |

### Adding a New Level
Add a new 6×10 array to the `LEVELS` array. Values 0–6. That's all — the system picks it up automatically. Max row count is flexible (the grid renders any number of rows).

### Level Progression
- Clearing all bricks → `endGame(true)` is called
- If `currentLevel < 10` → shows **Level Clear** screen with `level × 100` score bonus
- Player clicks **NEXT LEVEL** → `advanceLevel()` rebuilds bricks, resets ball/paddle, increments `currentLevel`
- After level 10 → shows **YOU WIN** end screen

---

## Theme System

### Theme Object Shape
```js
{
  id,           // string key, stored in localStorage
  label,        // display name
  accent,       // primary UI color (HEX)
  dim,          // accent at ~10% opacity for backgrounds
  bg,           // canvas background color
  grid,         // canvas grid line color
  pageBg,       // page/overlay background color
  p1, p2,       // paddle gradient top/bottom
  b1, b2, b3,   // ball radial gradient stops
  brickRows,    // array of 6 [color, glowColor] pairs
  logoClass,    // CSS class for the animated main menu logo
  logoText,     // text content ('ARKANOID')
}
```

### Applying a Theme
`applyTheme()` sets 5 CSS custom properties on `:root`:
```
--accent, --accent-dim, --page-bg, --screen-bg, --pause-bg
```
Then calls `recolorBricks()` to update live brick colors, `renderMainLogo()`, and `updateGameLogo()`.

### Changing Brick Colors Mid-Game
`recolorBricks()` iterates all bricks and updates `color`/`glow` using each brick's stored `colorIdx`. This is called whenever the theme changes, including from the pause menu skin picker.

### Adding a New Theme
Add an object to the `THEMES` array following the shape above. Add a corresponding CSS class for the logo animation (see logo classes below).

---

## Themed Logos

Each theme has a unique animated logo on the main menu:

| Theme  | Logo Class      | Effect                                          |
|--------|-----------------|-------------------------------------------------|
| Neon   | `logo-neon`     | Cyan glow pulse animation                       |
| Retro  | `logo-retro`    | Pixel font (Press Start 2P) + animated grain    |
| Synth  | `logo-synth`    | Pink-purple gradient with pulsing drop shadow   |
| Matrix | Special (JS)    | Live katakana rain canvas + green glow flicker  |
| Gold   | `logo-gold`     | Metallic bevel gradient + shimmer               |
| Ice    | `logo-ice`      | Frosted white with glimmer flash                |
| Lava   | `logo-lava`     | Fire gradient + random flicker keyframes        |
| Void   | `logo-void`     | Shifting deep-purple gradient                   |

The Matrix logo is special — `renderMainLogo()` injects a canvas element and calls `startMatrixRain()` which runs its own `requestAnimationFrame` loop (`matrixRainAnim`).

---

## Save System

Uses `localStorage`. 3 save slots keyed `ark_slot_1`, `ark_slot_2`, `ark_slot_3`.

### Save Data Shape
```js
{
  score,        // current score
  lives,        // remaining lives
  level,        // current level number
  bricks,       // full brick array snapshot
  bricksLeft,   // count of alive bricks (for slot display)
  date,         // Date.now() timestamp
  themeId,      // active theme id — restored on load
}
```

### Save/Load Flow
- **Save** (`showSlots('save', from)`) — opens slot picker, `saveSlot(i)` writes to localStorage, returns to `from` screen after 900ms
- **Load** (`loadSlot(i)`) — reads slot, restores theme, calls `restoreState(s)`, starts game loop
- Theme preference saved separately as `ark_theme` and `ark_shape` (shape key is legacy, shapes were removed)

---

## Controls

| Input               | Action                          |
|---------------------|---------------------------------|
| Mouse move          | Move paddle                     |
| `←` / `→` arrows   | Move paddle (keyboard)          |
| `Space` or click    | Launch ball                     |
| `ESC`               | Open / close pause menu         |
| `↑` / `↓` arrows   | Navigate menus                  |
| `Enter`             | Confirm menu selection          |

---

## Particles

`spawnParticles(x, y, color)` emits 14 particles on brick destruction. Each particle has velocity, gravity (`dy += 0.08`), radius, and a `life` value that decays each frame. Drawn with `globalAlpha = life` for a natural fade.

---

## Known Limitations / Future Ideas

- **No sound** — Web Audio API could add synthesis-based SFX per theme
- **Power-ups** — could drop from bricks (multi-ball, wide paddle, slow ball)
- **Mobile/touch** — touch events not wired up yet
- **High score board** — `localStorage` already available, just needs a leaderboard screen
- **Level editor** — the `LEVELS` array format is simple enough for a visual editor
- **Animated bricks** — some levels could use moving or regenerating bricks
