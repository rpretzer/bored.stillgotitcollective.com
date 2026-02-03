# STILL GOT IT COLLECTIVE — Infinite Platformer
## Build Plan & Resume Document

**Target:** `/home/rpretzer/bored.stillgotitcollective.com/index.html`
**Constraint:** Single file, vanilla HTML5/Canvas/JS only, under 5000 lines, zero external deps.
**Deploy:** GitHub Pages at `bored.stillgotitcollective.com`

---

## TASK STATUS (resume from here)

| Task | Phase | Status | Notes |
|------|-------|--------|-------|
| 2 | A — Skeleton | IN PROGRESS | HTML shell, constants, pixel font, canvas, bootstrap |
| 3 | B — Player | PENDING | Input system, player sprite, physics stub |
| 4 | C — World | PENDING | Draw helpers, chunk gen, parallax, collision |
| 5 | D — Enemies+Coins | PENDING | Enemy patrol, stomp, coin bob, collect |
| 6 | E — Audio | PENDING | Full Web Audio synth engine + sequencer |
| 7 | F — Juice | PENDING | Particles, shake, scoring, unlock notification |
| 8 | G — Screens | PENDING | Title, HUD, game over, touch controls, localStorage |
| 9 | H — Polish | PENDING | Final tuning, verify, line count check |

**When resuming:** Read this file first. Check which task is in_progress. The index.html file may partially exist — read it before writing to avoid clobbering. All 25 sections should appear in order inside one `<script>` block, separated by `// ─── SECTION N ───` comment headers.

---

## LINE BUDGET

| # | Section | Est. Lines |
|---|---------|-----------|
| 1 | HTML shell + meta + CSS | 60 |
| 2 | Global constants + state enum | 45 |
| 3 | Pixel font bitmap + drawPixelText() | 120 |
| 4 | Input system (keyboard + rising-edge) | 55 |
| 5 | Canvas bootstrap + resize/scale | 40 |
| 6 | Web Audio API + drum sequencer | 200 |
| 7 | Particle system | 65 |
| 8 | Screen shake manager | 20 |
| 9 | Procedural pixel-art draw helpers | 140 |
| 10 | Player sprite + 3-frame anim | 110 |
| 11 | Enemy sprite + patrol AI | 90 |
| 12 | Coin (gold chain) sprite + bob | 50 |
| 13 | World/chunk generator | 180 |
| 14 | Parallax background layers | 220 |
| 15 | Platform & collision system | 100 |
| 16 | Player physics + coyote + jump buffer | 95 |
| 17 | Scoring / Rep meter / lives / unlock | 70 |
| 18 | Main game loop (update + draw orchestration) | 80 |
| 19 | Title screen (animated logo + branding) | 130 |
| 20 | HUD overlay (meter, lives, distance) | 75 |
| 21 | Game Over screen (score, name input, share) | 110 |
| 22 | Touch controls (D-pad + jump button) | 90 |
| 23 | localStorage high-score read/write | 30 |
| 24 | State machine transitions + wiring | 50 |
| 25 | Entry point / bootstrap | 25 |
| **TOTAL CORE** | | **~2750** |
| **Contingency** | | **~500** |
| **Hard ceiling** | | **5000** |

---

## DEPENDENCY GRAPH

```
[1] HTML/CSS shell
  └→ [2] Constants + State enum
        ├→ [3] Pixel Font (no runtime deps)
        └→ [5] Canvas bootstrap
              ├→ [4] Input System ──→ [22] Touch Controls
              │     └→ [6] Audio System (gesture-gated)
              └→ [7] Particles   [8] Screen Shake
                    └→ [9] Draw Helpers
                          ├→ [10] Player Sprite
                          ├→ [11] Enemy Sprite
                          └→ [12] Coin Sprite
                    └→ [14] Parallax Backgrounds
                          └→ [13] World/Chunk Generator
                                └→ [15] Collision System
                                      └→ [16] Player Physics
                                            └→ [17] Scoring/Unlock
                                                  └→ [18] Game Loop
                                                        ├→ [19] Title Screen
                                                        ├→ [20] HUD
                                                        └→ [21] Game Over
  [23] localStorage (standalone util)
  [24] State transitions (wires 19→18→21→19)
  [25] Bootstrap (runs last)
```

---

## BUILD ORDER

```
Phase A — Skeleton:        §1 → §2 → §5 → §3 → §25
Phase B — Player:          §4 → §10 → §16 (stub)
Phase C — World:           §9 → §13 → §14 → §15
Phase D — Enemies+Coins:   §11 → §12 (+ collision in §15)
Phase E — Audio:           §6
Phase F — Juice:           §7 → §8 → §17
Phase G — Screens:         §19 → §20 → §21 → §22 → §23 → §24
Phase H — Polish:          Tune + verify
```

---

## SECTION SPECS

### §1 — HTML Shell + Meta + CSS (~60 lines)
- DOCTYPE, charset, viewport
- OG tags: `og:title="Still Got It Collective"`, `og:url="https://stillgotitcollective.com"`, `og:description`, `og:type=website`
- Twitter Card: `summary_large_image`
- Inline favicon: `<link rel="icon" href="data:image/png;base64,...">`  — 16×16 neon pink pixel icon (valid minimal PNG)
- `<title>Still Got It Collective</title>`
- CSS: `*{margin:0;padding:0;box-sizing:border-box}`, body flex-centered, `overflow:hidden`, bg `#0a0a1a`, canvas `image-rendering:pixelated`
- Single `<canvas id="gameCanvas">` in a wrapper div

### §2 — Constants + State Enum (~45 lines)
```js
const GRAVITY = 0.5;
const JUMP_VY = -12;
const DOUBLE_JUMP_VY = -8;
const COYOTE_FRAMES = 6;
const JUMP_BUFFER_FRAMES = 4;
const AUTO_SCROLL_SPEED = 3;   // px/frame
const MOVE_SPEED = 2;          // px/frame
const PLAYER_W = 16, PLAYER_H = 24;
const CHUNK_WIDTH = 400;
const COIN_VALUE = 50;
const LIVES_START = 3;
const DOUBLE_JUMP_UNLOCK_REP = 2000;
const GAME_W = 800, GAME_H = 450;
const STATE = { TITLE: 0, PLAYING: 1, GAMEOVER: 2 };
// 16-step drum pattern: kick / snare / hihat (arrays of 16 booleans)
const KICK_PAT  = [1,0,0,0, 1,0,0,0, 1,0,0,0, 1,0,0,0];
const SNARE_PAT = [0,0,0,0, 1,0,0,0, 0,0,0,0, 1,0,0,0];
const HIHAT_PAT = [1,0,1,0, 1,0,1,0, 1,0,1,0, 1,0,1,0];
```

### §3 — Pixel Font (~120 lines)
- 5×7 bitmaps for: `0-9 A-Z : ! @ . SPACE` (~40 chars)
- Each char = 7 integers, bits 4→0 = columns left→right
- `drawPixelText(ctx, text, x, y, color, scale)` — iterates chars, draws set bits as fillRect
- `measurePixelText(text, scale)` → `text.length * 6 * scale`

### §4 — Input System (~55 lines)
- `const input = {left:false, right:false, jump:false, enter:false, up:false, down:false}`
- `const prevInput = {jump:false, enter:false}`
- Rising-edge computed in game loop: `input.jumpPressed = input.jump && !prevInput.jump`
- keydown: set flags; Space → preventDefault
- keyup: clear flags
- `let audioUnlocked = false` — first keydown/touchstart resumes AudioContext

### §5 — Canvas Bootstrap (~40 lines)
- Get canvas + ctx; set `canvas.width = GAME_W; canvas.height = GAME_H`
- `resizeCanvas()`: `scale = Math.min(innerWidth/GAME_W, innerHeight/GAME_H)`, apply via CSS style
- addEventListener resize

### §6 — Web Audio + Sequencer (~200 lines)
**Gesture-gated. Create AudioContext lazily on first input.**
- Noise buffer: 1s of `Math.random()*2-1` at sampleRate, generated once lazily
- Synthesis functions (all schedule on audioCtx, fire-and-forget):
  - `playKick(t)`: sine, freq 150→30Hz ramp 0.08s, dur 0.15s
  - `playSnare(t)`: noise 0.1s + HP 200Hz + sine 200Hz body
  - `playHiHat(t)`: noise 0.05s + HP 8kHz, quiet
  - `playJumpSound()`: noise 0.08s, HP sweep 2k→800Hz
  - `playCoinSound()`: 2× sine (800 + 1200Hz), staggered 30ms, fast decay
  - `playBeatDrop()`: sine 40Hz, LP 120Hz, 0.6s slow attack
- Sequencer: `step`, `nextStepTime`, BPM (starts 120, ramps with difficulty)
- `updateSequencer(timestamp)`: lookahead loop `while(nextStepTime < audioCtx.currentTime + 0.1)` — schedule kick/snare/hat per pattern, advance step, calc next time
- Public API: `audioPlayJump()`, `audioPlayCoin()`, `audioPlayBeatDrop()`

### §7 — Particle System (~65 lines)
- `const particles = []`
- Shape: `{x, y, vx, vy, life, maxLife, color, size}`
- `emitParticles(x, y, count, {spreadX, spreadY, colors, life, size})`
- Presets: `emitLandingConfetti(x,y)` (10 particles, neon palette), `emitCoinBurst(x,y)` (6 gold)
- Update: vy += 0.1 gravity, life--, splice dead (backwards iteration)
- Draw: alpha = life/maxLife, fillRect

### §8 — Screen Shake (~20 lines)
- `let shakeAmount = 0`
- `triggerShake(mag)` — sets only if mag > current
- `getShakeOffset()` → `{x: rand*shakeAmount, y: rand*shakeAmount}`
- Decay: `shakeAmount *= 0.9` each frame in game loop

### §9 — Draw Helpers (~140 lines)
- `drawNeonRect(ctx, x, y, w, h, color, glowColor)` — shadow pass + stroke pass
- Platform drawers:
  - `drawPlatformNeonLedge(ctx, x, y, w)` — pink/cyan outlined, 4px tall, neon glow
  - `drawPlatformGraffitiWall(ctx, x, y, w, seed)` — filled + seeded random colored dots
  - `drawPlatformSubwayRoof(ctx, x, y, w)` — gray + rivets
- Props: `drawBoombox(ctx, x, y)`, `drawHydrant(ctx, x, y)`, `drawSubwaySign(ctx, x, y, ch)`
- `drawGoldChainIcon(ctx, x, y, size)` — lives HUD icon
- Seeded PRNG: `function sRand(seed) { seed=(seed*1664525+1013904223)|0; return {v:(seed>>>0)/4294967296, s:seed}; }`

### §10 — Player Sprite (~110 lines)
- 16×24 dancer. Palette: skin `#D2A679`, shirt `#FFFFFF`, chain `#FFD700`/`#B8860B`, boombox `#333`
- 4 pose functions drawing fillRects on the 16×24 grid:
  - `drawPlayerIdle(ctx, px, py)` — standing, hip pop, chain at neck, boombox on shoulder
  - `drawPlayerRunL(ctx, px, py)` — left leg forward
  - `drawPlayerRunR(ctx, px, py)` — right leg forward
  - `drawPlayerJump(ctx, px, py)` — legs tucked, arms out
- Dispatcher: `drawPlayer(ctx, player)` — picks pose by `isJumping` flag + `frame % 2`
- `player.animTick` increments each frame; frame flips every 8 ticks

### §11 — Enemy Sprite + AI (~90 lines)
- Enemy: `{x, y, w:16, h:24, platLeft, platRight, dir, frame, animTick, alive}`
- Patrol: `x += dir * 1.5`; flip dir at platLeft/platRight edges
- 2 poses: `drawEnemyA(ctx, x, y)`, `drawEnemyB(ctx, x, y)` — red jacket rival, no boombox
- Mirror helper: `drawMirrored(ctx, x, w, fn)` uses save/translate/scale(-1,1)/restore
- Hitbox: top 4px = stomp zone, rest = body kill zone
- animTick flips frame every 12 ticks

### §12 — Coin Sprite (~50 lines)
- Coin: `{x, y, baseY, bobPhase, collected}`
- Bob: `drawY = baseY + Math.sin(bobPhase) * 4`; bobPhase += 0.05/frame
- `drawCoin(ctx, x, y)` — gold chain links in 12×10 px, colors `#FFD700` + `#B8860B` + `#DAA520`
- Collection & rep logic handled in §15 collision checks

### §13 — World/Chunk Generator (~180 lines)
- Chunk: `{startX, platforms[], enemies[], coins[], props[]}`
- Platform shape: `{x, y, w, h, type, seed}`
- `generateChunk(startX, prevPlatEnd)` — returns chunk object:
  - **Continuity:** first platform starts at `prevPlatEnd.x + gap`, Y clamped within ±144px of prevPlatEnd.y (max jump height = vy²/2g = 144). Downward drop capped 150px.
  - **Heights:** layered sines: `y = 280 + sin(x*0.01)*50 + sin(x*0.03)*25`
  - **Gaps:** `40 + difficulty * 80` px (difficulty 0→1 from §17)
  - **Platform count:** 3 + sRand 0-3
  - **Types:** random from neonLedge/graffitiWall/subwayRoof
  - **Enemies:** 0–2 per chunk on platforms >80px wide. Spawn prob = `min(0.7, distance/12000)`
  - **Coins:** 1–3 per platform, 20px above surface, spaced by sRand
  - **Props:** 2–4 per chunk with parallaxLayer tag (1, 2, or 3)
- `chunks = []` array; `lastPlatEnd = {x, y}` tracks continuity state
- `ensureChunksAhead(cameraX)` — generate while last chunk right edge < cameraX + GAME_W + CHUNK_WIDTH
- `cullChunksBehind(cameraX)` — remove chunks ending < cameraX - CHUNK_WIDTH

### §14 — Parallax Backgrounds (~220 lines)
- **Sky (static):** `createLinearGradient` top `#050510` → bottom `#2a0a4a`. Fill full canvas.
- **Distant skyline (0.1×):**
  - Pre-generate 2048px wide skyline strip on first call (store in offscreen canvas or array)
  - Buildings: random widths 20-50px, heights 40-120px, fill `#0d0d1f`
  - Neon windows: 1-2px dots in `#FF2D8C` / `#00E5FF` / `#FFD700`, seeded per building
  - Tile with modulo: `drawX = (worldX - cameraX*0.1) % 2048`
- **Mid-ground (0.4×):** iterate chunk props with parallaxLayer==2. Draw boomboxes/flyers at `prop.x - cameraX*0.4`
- **Near-ground (0.8×):** chunk props with parallaxLayer==3. Subway trains + hydrants at `prop.x - cameraX*0.8`
- Off-screen culling: skip if drawX + width < 0 or drawX > GAME_W

### §15 — Collision System (~100 lines)
- `getActivePlatforms()` — flatten all chunk platforms, return as `[{x,y,w,h,type,seed}, ...]`
- **Top-only platform collision:**
  ```
  if (player.prevY + PLAYER_H <= plat.y &&
      player.y + PLAYER_H >= plat.y &&
      player.x + PLAYER_W > plat.x &&
      player.x < plat.x + plat.w)
  ```
  → snap `player.y = plat.y - PLAYER_H`, `vy = 0`, `isGrounded = true`
- **Fall death:** `player.y > GAME_H + 50` → life lost
- **Enemy collision:** for each alive enemy:
  - If AABB overlap AND player.vy > 0 AND player.y+PLAYER_H <= enemy.y+4 (stomp zone): kill enemy, player.vy = -8, +100 rep, emitCoinBurst, triggerShake(3)
  - Else if AABB overlap: player hit, life lost, triggerShake(5), brief invuln (60 frames)
- **Coin collision:** for each !collected coin: dist(player center, coin center) < 22 → collect, +50 rep, audioPlayCoin(), emitCoinBurst, triggerShake(2)

### §16 — Player Physics (~95 lines)
- Called once per frame during PLAYING state
- **Horizontal:** `player.x += AUTO_SCROLL_SPEED`. If left: `x -= MOVE_SPEED`. If right: `x += MOVE_SPEED`. Clamp: `x >= cameraX` (can't fall behind camera)
- **Gravity:** `player.vy += GRAVITY`
- **Coyote:** when `wasGrounded && !isGrounded` → `coyoteTimer = COYOTE_FRAMES`. Decrement each frame.
- **Jump buffer:** on `jumpPressed && !isGrounded` → `jumpBufferTimer = JUMP_BUFFER_FRAMES`. Decrement each frame.
- **Jump:** fires if `jumpPressed || jumpBufferTimer > 0` AND (`isGrounded || coyoteTimer > 0`):
  - First jump: `vy = JUMP_VY`, audioPlayJump(), emitLandingConfetti, reset coyote+buffer
  - Double jump (if unlocked && !hasDoubleJumped && airborne && !first-jump-eligible): `vy = DOUBLE_JUMP_VY`, same effects + extra particles
- **Landing:** when collision sets isGrounded → reset hasDoubleJumped, emit landing confetti
- Store `player.prevY = player.y` BEFORE applying vy (needed by §15)
- **Camera:** `cameraX = player.x - 150` (player rides ~150px from left edge)

### §17 — Scoring / Rep / Unlock (~70 lines)
- `game.rep += AUTO_SCROLL_SPEED / 10` each frame (distance-based)
- `game.distance += AUTO_SCROLL_SPEED` each frame
- Coin collect adds COIN_VALUE (50). Enemy stomp adds 100. (Called from §15)
- `METER_THRESHOLD = 500 + Math.floor(game.distance / 2000) * 200` (grows over time)
- `game.meterFill = game.rep % METER_THRESHOLD` (for HUD bar)
- Double-jump unlock: `if (game.rep >= DOUBLE_JUMP_UNLOCK_REP && !game.doubleJumpUnlocked)` → set flag, `game.unlockFlashTimer = 120`
- `getDifficulty()` → `Math.min(1, game.distance / 15000)` (0→1 over 15k px)
- Beat drop: `if (game.distance % 5000 < AUTO_SCROLL_SPEED)` → audioPlayBeatDrop(), triggerShake(4), bump BPM +5

### §18 — Main Game Loop (~80 lines)
- `let lastTime = 0; function gameLoop(ts) { ... requestAnimationFrame(gameLoop); }`
- Top of loop: compute rising edges (`jumpPressed`, `enterPressed`, `upPressed`, `downPressed`)
- State dispatch:
  - TITLE: `updateTitle(); drawTitle();`
  - PLAYING: full update chain then full draw chain (see below)
  - GAMEOVER: `updateGameOver(); drawGameOver();`
- **PLAYING update chain:**
  1. updateSequencer(ts)
  2. updatePlayerPhysics(player, input)
  3. ensureChunksAhead(cameraX) / cullChunksBehind(cameraX)
  4. updateEnemies (all alive enemies)
  5. updateCoinBobs (bobPhase += 0.05)
  6. checkPlatformCollision
  7. checkEnemyCollision
  8. checkCoinCollision
  9. updateParticles()
  10. updateScoring()
  11. shakeAmount *= 0.9
  12. death/respawn/gameover check
- **PLAYING draw chain:**
  1. clearRect(0,0,GAME_W,GAME_H)
  2. ctx.save(); ctx.translate(shakeOffX, shakeOffY)
  3. drawSky()
  4. drawDistantSkyline(cameraX)
  5. drawMidground(cameraX, chunks)
  6. drawNearground(cameraX, chunks)
  7. drawPlatforms(cameraX, chunks)  ← §9 platform drawers
  8. drawEnemies(cameraX, chunks)
  9. drawCoins(cameraX, chunks)
  10. drawPlayer(ctx, player)
  11. drawParticles(ctx)
  12. ctx.restore()  ← SHAKE ENDS
  13. drawHUD()      ← outside shake
  14. drawTouchControls() if touch device

### §19 — Title Screen (~130 lines)
- `let titleTime = 0` — increments each frame
- `updateTitle()`: if jumpPressed or enterPressed → transitionToPlaying()
- `drawTitle()`:
  - Sky gradient background (same as gameplay)
  - Animated neon logo: draw "STILL GOT IT" twice — first with ctx.shadowBlur pulsing (8 + sin(titleTime*0.05)*6), color `#FF2D8C`, then again clean. Scale 5.
  - "COLLECTIVE" below, scale 3, `#B388FF`, steady glow (shadowBlur 6)
  - Crew roster scroll: array of ~10 fake names. Each at `y = 340 - (titleTime % 200)` offset, spaced 22px. Modulo-wrapped. Scale 2, dim white.
  - Blink "PRESS SPACE TO START": visible when `Math.floor(titleTime/20) % 3 !== 2`. Scale 2, `#00E5FF`, centered bottom.
  - Decorative boombox left side, dancer silhouette right side (using §9 helpers)
  - Scanlines: loop `y += 4`, fillRect with rgba(0,0,0,0.15 + sin(titleTime)*0.05)

### §20 — HUD Overlay (~75 lines)
Drawn AFTER ctx.restore — UI does not shake.
- **Rep meter (top-left, y=10):**
  - "STILL GOT IT" label, scale 2, `#FF2D8C`
  - Bar BG: fillRect 200×14, `#1a1a2e`, stroke 1px `#FF2D8C`
  - Bar fill: fillRect width = `(meterFill/METER_THRESHOLD)*200`, color `#00E5FF`. If `game.meterFlash > 0`: color = white, decrement flash timer.
  - Rep number to right: `drawPixelText(ctx, String(game.rep), ...)` scale 2, white
- **Lives (top-right):** draw `game.lives` gold chain icons via `drawGoldChainIcon`. If `game.lifeLostAnim > 0`: rightmost icon fades (alpha = lifeLostAnim/30), decrement.
- **Distance (below lives):** "DIST:" + distance number, scale 2, `#aaa`
- **Unlock flash:** if `game.unlockFlashTimer > 0`: dark overlay (rgba 0,0,0,0.5), "DOUBLE JUMP UNLOCKED!" scale 3, `#FFD700` pulsing. Decrement timer.
- **Mobile hint:** if touch device and `game.hintTimer < 300`: show "TAP LEFT/RIGHT | JUMP" scale 1, fading after frame 240

### §21 — Game Over Screen (~110 lines)
- State: `let goPhase = 'display'`, `goTimer = 0`, `goName = [0,0,0]` (indices into A-Z), `goCursor = 0`
- `updateGameOver()`:
  - `display`: goTimer++. At 90 → if `game.rep > highScore.score` → goPhase='nameInput' else goPhase='done'
  - `nameInput`: Up → cycle char up. Down → cycle char down. Left/Right → move cursor. Enter → save `saveHighScore(game.rep, name)`, goPhase='done'
  - `done`: Enter/Space → transitionToTitle()
- `drawGameOver()`:
  - Dark bg gradient
  - "GAME OVER" scale 5, `#FF2D8C`, centered, shadowBlur 15
  - "YOUR REP: XXXXX" scale 3, white
  - "BEST: XXXXX (NAME)" scale 2, `#FFD700`
  - If nameInput phase: 3 char boxes (60×60px each), selected one has neon border. Letter drawn scale 4 inside each box. Instructions below in scale 1.
  - If `goPhase=='done'`: "PRESS SPACE TO PLAY AGAIN" blinking
  - Share text: "I got XXXXX rep! stillgotitcollective.com" scale 1, dim
  - Footer: "stillgotitcollective.com – Dance like you did. Sleep like you should. Collective energy." scale 1, `#666`, bottom center
- **Key routing:** during nameInput, Up/Down/Left/Right are consumed here, NOT by player physics. Check `gameState === STATE.GAMEOVER` in the input routing.

### §22 — Touch Controls (~90 lines)
- `const isTouchDevice = navigator.maxTouchPoints > 0`
- Button defs (screen-space, not world-space):
  - Left: `{x:20, y:GAME_H-90, w:60, h:60}`
  - Right: `{x:90, y:GAME_H-90, w:60, h:60}`
  - Jump: `{x:GAME_W-100, y:GAME_H-90, w:70, h:70}`
- Pressed state: `touchLeft`, `touchRight`, `touchJump` booleans
- `initTouchControls()`: add touchstart/touchmove/touchend on canvas with `{passive:false}`. Each handler:
  - preventDefault()
  - Map each touch in `event.touches` to canvas coords: `cx = (t.clientX - rect.left) / (rect.width / GAME_W)`
  - Point-in-rect test against each button def → set corresponding input flag
  - touchend: clear all touch input flags
- `drawTouchControls(ctx)`: draw 3 rounded-rect buttons. Color: border `#00E5FF`/`#FF2D8C`, fill `rgba(0,40,60,0.3)`. Brighten fill if pressed.

### §23 — localStorage (~30 lines)
- `const LS_KEY = 'sgic_highscore'`
- `function loadHighScore()`: try `JSON.parse(localStorage.getItem(LS_KEY))` → return obj or `{score:0,name:'AAA'}` on fail
- `function saveHighScore(score, name)`: `localStorage.setItem(LS_KEY, JSON.stringify({score, name}))`
- `let highScore = {score:0, name:'AAA'}` — loaded at bootstrap

### §24 — State Transitions (~50 lines)
- `transitionToTitle()`: `gameState = STATE.TITLE; titleTime = 0; sequencerRunning = false;`
- `transitionToPlaying()`:
  - `gameState = STATE.PLAYING`
  - Init game: `{lives: LIVES_START, rep: 0, distance: 0, doubleJumpUnlocked: false, unlockFlashTimer: 0, meterFlash: 0, lifeLostAnim: 0, hintTimer: 0}`
  - Init player: `{x: 150, y: 200, vy: 0, prevY: 200, isGrounded: false, coyoteTimer: 0, jumpBufferTimer: 0, hasDoubleJumped: false, frame: 0, animTick: 0, invulnTimer: 0}`
  - Clear chunks array, reset lastPlatEnd to `{x: 150, y: 300}`
  - Generate 3 initial chunks
  - `cameraX = 0; sequencerRunning = true;`
- `transitionToGameOver()`: `gameState = STATE.GAMEOVER; goPhase = 'display'; goTimer = 0; sequencerRunning = false;`

### §25 — Bootstrap (~25 lines)
- `highScore = loadHighScore()`
- `resizeCanvas()`
- `gameState = STATE.TITLE`
- `if (isTouchDevice) initTouchControls()`
- `requestAnimationFrame(gameLoop)`

---

## 10 CRITICAL GOTCHAS

1. **AudioContext policy:** Lazy-create on first user gesture only. Guard all audio calls.
2. **Lookahead scheduling:** `while(nextStepTime < audioCtx.currentTime + 0.1)` — never call synths per-frame directly.
3. **Top-only collision needs prevY:** Compare previous frame Y to current. Without it, jumping into undersides = false landing.
4. **Coyote + jump buffer are independent:** Two timers, two reset paths. Neither allows infinite jump.
5. **Touch listeners passive:false:** Otherwise browser scrolls the page on tap and preventDefault is ignored.
6. **Multi-touch:** Iterate `event.touches` array for simultaneous Left + Jump.
7. **Seeded PRNG call order:** Fixed sequence per chunk. Any reorder = different chunk = visual flicker.
8. **HUD drawn after ctx.restore:** UI must NOT shake. restore() before any HUD draw calls.
9. **Key remapping on game over:** nameInput phase consumes arrow keys — do not route to player.
10. **Favicon must be valid PNG:** Bad base64 silently fails. Use a known-good minimal PNG template.

---

## PALETTE REFERENCE

| Name | Hex | Usage |
|------|-----|-------|
| Neon Pink | `#FF2D8C` | Logo, platform ledges, key UI |
| Neon Cyan | `#00E5FF` | Meter fill, buttons, accents |
| Gold | `#FFD700` | Chains, coins, high score |
| Shadow Gold | `#B8860B` | Chain shadows |
| Dim Gold | `#DAA520` | Coin highlights |
| Purple | `#B388FF` | "COLLECTIVE" text |
| Sky Top | `#050510` | Gradient top |
| Sky Bottom | `#2a0a4a` | Gradient horizon |
| Building Sil | `#0d0d1f` | Distant skyline fill |
| BG Dark | `#0a0a1a` | Page/canvas background |
| Skin | `#D2A679` | Player/enemy skin tone |
| Player Shirt | `#FFFFFF` | Player tank top |
| Enemy Jacket | `#E63946` | Enemy red jacket |

---

## VERIFICATION CHECKLIST

- [ ] Opens in Chrome/Firefox/Safari with zero console errors
- [ ] Title: logo pulses, crew names scroll, PRESS SPACE blinks
- [ ] Gameplay: player auto-scrolls, jumps land on platforms
- [ ] Coyote time: walk off edge, jump still works for ~6 frames
- [ ] Jump buffer: press jump 1-2 frames before landing, jump fires on land
- [ ] Coins: collected with cha-ching sound, +50 rep, particles
- [ ] Enemies: patrol on platforms, stomp kills them (+100 rep, bounce), side-hit loses life
- [ ] Audio: drum loop plays, jump scratch fires, coin cha-ching fires, beat drop on difficulty ramp
- [ ] At 2000 rep: flash notification, double jump works (second jump in air)
- [ ] 3 deaths: game over screen appears
- [ ] New high score: name input (3 chars, arrow nav, enter confirm)
- [ ] Reload: high score persists from localStorage
- [ ] Window resize: canvas rescales, no distortion, game continues
- [ ] Touch device (or DevTools mobile sim): D-pad renders, buttons work, no page scroll
- [ ] Line count: `wc -l index.html` under 5000
- [ ] OG tags present in page source
