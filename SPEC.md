# Bubble Poke: Game Specification

A webcam game for children. The camera picture fills the screen as a mirror and bubbles float across it. Waving a hand at a bubble pops it for points. The game gets faster and harder over a fixed length of time (65 seconds by default). Extras: a phoenix to avoid, a disco-ball super power set off by clapping, one- or two-player split screen, two visual themes, English and Swedish, and a local leaderboard.

This document describes the game as it currently plays in `index.html`. It is meant to be enough to rebuild the game on another platform without reading the source. Numbers are exact unless marked approximate. Where the current build made a deliberate trade-off, it is noted as **Design note**.

---

## 1. Platform requirements

| Need | Reference implementation | Notes for a port |
|---|---|---|
| Camera | `getUserMedia({video: {width: 960, height: 720, facingMode: 'user'}})` | Front camera, any resolution ≥ 320×240. Only needed at low resolution (see §7). |
| Microphone | `getUserMedia({audio: {echoCancellation: false, noiseSuppression: false, autoGainControl: false}})` | Optional. Requested **separately** from the camera and only when the player turns clapping on, so refusing one never blocks the other. Processing must be **off**, because echo cancellation and noise suppression remove claps. |
| Rendering | Canvas 2D, scaled by devicePixelRatio (capped at 2) | Any immediate-mode 2D renderer. About 60 fps. |
| Audio | WebAudio oscillators and noise buffers only | No audio files. Every sound is synthesised (§13). |
| Storage | `localStorage` | Any small key–value store (§15). |
| Network | **None** | The game must work offline and send nothing anywhere. |

**Privacy promise (shown on the start screen; must remain true):** "Your camera feed never leaves this computer. Nothing is recorded, uploaded, or saved." Camera frames are processed in memory and discarded. Nothing is written to disk except the settings and the leaderboard.

---

## 2. Screens and flow

```
Start screen ──"Turn on camera"──▶ (camera permission) ──▶ Game
     │        ──"No camera — play with the mouse"──────────▶ Game (pointer only)
     │        ──"Settings"──▶ Settings ──"Done"/Esc──▶ back to Start
     │
Game ──clock runs out / every player out of hearts──▶ Game Over
     ──⚙ Settings──▶ Settings (game paused) ──Done──▶ Game resumes
     ──↻ Restart──▶ new Game immediately

Game Over ──"Play again"──▶ new Game
          ──"Settings"──▶ Settings ──Done──▶ back to Game Over
```

### 2.1 Start screen (top to bottom)
1. **Language picker**: two pill buttons, 🇬🇧 English / 🇸🇪 Svenska. The choice applies immediately to all text and is remembered.
2. **Title** "Bubble Poke".
3. **Tagline** (two lines): "Your camera becomes the game board. / Wave a hand at a bubble to pop it."
4. **Player-count picker**: 👤 1 player / 👥 2 players. Remembered.
5. **"How to play ▾"**: a disclosure, **collapsed by default**. When opened it shows the seven rules (§2.2).
6. **Primary button**: "Turn on camera".
7. Text links: "No camera — play with the mouse", "Settings".
8. Error box (hidden until a camera error occurs; messages in §16).
9. Privacy note (small print).

### 2.2 The seven rules (inside "How to play")
Each rule has an icon, a main sentence, and a dimmer follow-up. `{powerEvery}` and `{length}` are live values from Settings.

| Icon | Rule | Follow-up |
|---|---|---|
| 🫧 | Pop bubbles for points. | Small ones are worth more than big ones. |
| ⭐ | Golden star bubbles are worth 50. | They fly faster, too. |
| 🔥 | From level 2 a phoenix swoops across. | Don't touch it — it burns 75 points. |
| 🪩 | Every {powerEvery} points you earn a disco ball. | Clap once, loudly, and lasers pop bubbles for you. |
| ♥ | Let five bubbles escape and the game ends early. | Each new level gives one heart back. |
| ⏱ | {length}, getting faster and faster. | A harder level every 13 seconds. Change the length in Settings. |
| 🏆 | Add your name to the leaderboard when the game ends. | — |

`{length}` is written as prose, not a clock: under 120 s → "65 seconds"; a whole number of minutes → "3 minutes"; otherwise "2 min 30 sec".

### 2.3 Game Over screen
- **Title**: one player: "New best!" if the score is above 0 and at least the stored best, otherwise "Game Over". Two players: "Player N wins!" or "It's a draw!" if the scores are equal.
- **Line**: one player: "Time! You reached level N." or "Five bubbles got away on level N.", followed by "You scored S." Two players: "Player 1: S1 · Player 2: S2".
- **Name form**: one text field (two in two-player mode), max 24 characters, plus "Save score". The form is hidden if the best score in the game is 0. Player 1's field is pre-filled with the last name used, and **all of it is selected on focus**, so typing replaces it rather than appending. Enter submits. An empty field saves as "Player N". After saving, the form disappears (one entry per player per game).
- **Leaderboard table** (§14). The rows just saved are highlighted.
- "Play again" button; "Settings" link; hint: "Settings has the language, the game length and the disco ball."

### 2.4 In-game HUD
- **Top left: player 1's stack**: "P1" tag (two-player only), score with a "SCORE" caption, hearts (5 ♥; lost hearts dimmed), power pill (§9), combo text (§6.3).
- **Top centre/right: clock and level**: timer `M:SS` (pulses red at ≤ 10 s); "LEVEL N" pill with a progress bar for the current level.
- **Top right (two-player only): player 2's stack**, a mirror image of player 1's and right-aligned.
- **Bottom left**: "⚙ Settings" and "↻ Restart" buttons.
- **Centre banner** (shown for 1.8 s): level announcements and "Disco ball!" (§11).
- Losing a heart shakes that player's hearts pill (0.32 s, ±7 px).
- An eliminated player's stack fades to 42 % opacity.

---

## 3. Coordinate system and mirroring

- The camera image is drawn **mirrored horizontally**, so it behaves like a mirror: moving your right hand moves the image on the right.
- It is scaled to **cover** the screen (like CSS `object-fit: cover`): `s = max(W/vw, H/vh)`, centred, overflow cropped. Motion detection must use the **same** mapping so that motion lines up with what is drawn (§7.3).
- Modern theme: the video is darkened with `rgba(8,12,30,0.30)`. Arcade theme: darkened with `rgba(0,8,6,0.55)`, then overlaid with `rgba(0,255,170,0.22)` (overlay blend) for a green phosphor tint.
- With no camera, the background is a vertical gradient `#22306b → #0b1020` (modern), or black `#05060a` with a 44 px cyan grid at 10 % alpha (arcade).

---

## 4. Game clock, levels and tempo

### 4.1 Constants
| Name | Value | Meaning |
|---|---|---|
| `GAME_SECONDS` default | **65** | Total game length; adjustable 20–300 s in steps of 5 |
| `LEVEL_SECONDS` | **13** | A new level begins every 13 s of play |
| `TEMPO_PER_SEC` | 0.013 | Continuous speed-up per second |
| `TEMPO_MAX` | 2.1 | Tempo ceiling |
| `HARDEST` | 8 | Difficulty stops increasing at level index 8 (level 9) |
| `MIN_CROSS_SEC` | **1.15** | No bubble crosses its playfield faster than this |
| `MAX_LIVES` | 5 | Hearts per player |

A 65-second game therefore has 5 levels (0–12 s, 13–25 s, … 52–64 s).

### 4.2 Level index
`level = floor(elapsed_ms / 13000)`, where index 0 is shown as "Level 1". When the level increases:
- every player still in the game gets **+1 heart** (up to 5),
- a fanfare plays,
- the banner shows "Level N" with a subtitle (§4.4).

The level bar shows `(elapsed mod 13000) / 13000`.

### 4.3 Level configuration (`k = min(level, 8)`)
| Field | Formula | L1 (k=0) | L2 | L3 | L4 | L5 | L9+ |
|---|---|---|---|---|---|---|---|
| `speed` multiplier | `1 + 0.55k` | 1.00 | 1.55 | 2.10 | 2.65 | 3.20 | 5.40 |
| `size` multiplier | `max(0.58, 1 − 0.10k)` | 1.00 | 0.90 | 0.80 | 0.70 | 0.60 | 0.58 |
| `gap` ms between spawns | `max(260, 880 − 150k)` | 880 | 730 | 580 | 430 | 280 | 260 |
| `max` bubbles per playfield | `min(20, 12 + 2k)` | 12 | 14 | 16 | 18 | 20 | 20 |
| `golden` chance | `min(0.16, 0.08 + 0.015k)` | 8 % | 9.5 % | 11 % | 12.5 % | 14 % | 16 % |
| `dirs` (entry directions) | see right | up | up, left, right | all four | all four | all four | all four |
| `phoenixGap` ms | `max(3800, 10000 − 900k)` | (none) | 9100 | 8200 | 7300 | 6400 | 3800 |

"up" means the bubble rises from below the screen; "down" means it falls from the top; "left" enters from the left edge moving right; "right" enters from the right edge moving left.

### 4.4 Level subtitles
| Level shown | Subtitle |
|---|---|
| 1 | Bubbles float up from below |
| 2 | Sides open up — and the phoenix wakes |
| 3 | From every direction now |
| 4 and later | Faster. Smaller. Good luck. |

### 4.5 Tempo
On top of the level steps, everything speeds up continuously:
`tempo = min(2.1, 1 + elapsed_seconds × 0.013)`; it is 1.0 when no game is running.
Tempo multiplies bubble speed at spawn and divides the spawn gap.

### 4.6 Ending
- Clock reaches 0 → game over, reason "time".
- Every player has 0 hearts → game over, reason "lives".
- During the last 5 seconds a short 880 Hz square-wave tick plays once per second.

---

## 5. Bubbles

### 5.1 Spawning
Every frame, for the set of **live** (not eliminated) players:
- If `now − lastSpawn > gap / tempo / liveCount`, pick a live player at random. If that player's playfield holds fewer than `max` bubbles, spawn one there and reset `lastSpawn`.
- Dividing by the number of live players keeps each half as busy as a one-player screen.

### 5.2 Properties at spawn
- **Golden?** With probability `golden`.
- **Radius** in px: golden `(34 + rand·10) × size`; normal `(28 + rand·min(52, 0.075·W)) × size`.
- **Direction**: uniformly random from `dirs`.
- **Speed** in px/s:
  `raw = (52 + rand·44) × (golden ? 1.7 : 1) × (40/r + 0.55) × speed × tempo`
  Smaller bubbles move faster.
  `span` = playfield height (for up/down) or playfield width (for left/right).
  `speed = min(raw, span / 1.15)`.
  **Design note:** without this cap, late levels reached about 2000 px/s, which crosses a screen in under 0.4 s: that is unplayable rather than hard. Lower `MIN_CROSS_SEC` toward 0.9 for a more frantic game, or raise it toward 1.4 for younger players.
- **Start position**: just outside the playfield edge it enters from (one radius off-screen), at a random point along that edge, kept one radius in from the corners.
- **Wobble**: each bubble sways *across* its line of travel, never along it. Phase is random; `wobSpeed = 0.7 + rand·0.9` rad/s; `wobAmt = 12 + rand·22` px. The drawn position is `pos + perpendicular × sin(wob) × wobAmt`. Drawing, pointer hits and motion hits all use this **drawn** position.
- **Hue**: golden = 45. Normal modern bubbles use one of `[190, 208, 262, 288, 320, 342, 158, 128]`; normal arcade bubbles use one of `[180, 300, 60, 120, 200, 330]` (neon only).

### 5.3 Escaping
A bubble that moves more than `2r` beyond its playfield's edge (left, right, top or bottom) is removed and counts as a **miss** for its owner (§8).

### 5.4 Popping
- **Grace period**: a bubble cannot be popped by motion during its first **350 ms** (it would otherwise pop the instant it appeared on a moving hand).
- When popped, a bubble grows to 1.9× while fading out over about 0.28 s (`pop += dt × 3.6` until 1), then is removed. While popping it cannot be hit and cannot escape.
- Popping spawns 14 particles (26 for golden), a floating "+N" text, and a pop sound.

---

## 6. Scoring

### 6.1 Points per bubble
`base = golden ? 50 : round(30 − 0.22·r)`, floored at 5.
`points = max(5, base) × multiplier`.

### 6.2 Combo multiplier
- A pop within **1400 ms** of the same player's previous pop extends the combo; otherwise the combo resets to 1.
- `multiplier = min(5, 1 + floor((combo − 1) / 3))`. So pops 1–3 score ×1, pops 4–6 ×2, 7–9 ×3, 10–12 ×4, and 13 or more ×5.
- From combo 2 upward the HUD shows "{combo} in a row! ×{multiplier}".
- Missing a bubble or being burned resets the combo to 0.
- The pop sound's pitch rises with the combo: `min(2.2, 1 + 0.07·(combo−1)) × (golden ? 1.35 : 1) × 44/r`.

### 6.3 Score rules
- A score can **never go below 0** (this is a children's game).
- The best score ever is stored and updated live; in two-player mode, either player can set it.

---

## 7. Motion detection (the "poke")

This is motion detection, **not** hand tracking. Anything that moves can pop a bubble. That is deliberate: it needs no model and works for small hands, at any distance, in poor light.

### 7.1 Per frame
1. Draw the current video frame, **mirrored**, into a **192 × 144** buffer.
2. Convert to luma: `Y = (77R + 150G + 29B) >> 8`.
3. Compare with the previous frame's luma. A pixel counts as "moving" if `|ΔY| > threshold`, where `threshold = 74 − sensitivity` (the sensitivity slider runs 4–60, default 30, so threshold 70 → 14, default 44).
4. `globalMotion` = the fraction of moving pixels.

### 7.2 Sanity gate
If `globalMotion ≥ 0.45`, ignore the frame: a lighting change or a bumped camera moves everything at once.

### 7.3 Hit test
For a circle at screen position (x, y) with radius r:
- Map it into the 192×144 mask using the same cover-fit transform as the drawn video.
- Use an effective radius of `min(r, 50)` px. Large bubbles only watch their middle, so they are not harder to hit than small ones; a hand is hand-sized.
- `coverage` = moving pixels ÷ total pixels inside that ellipse. If fewer than 6 pixels are inside, coverage is 0.
- It is a hit if `coverage > need`, where `need = 0.42 − sensitivity/100 × 0.5` (range 0.40 → 0.12; default 0.27).

### 7.4 Motion trail
The centroid of all moving pixels (when there are at least 25) is added to a trail of up to 24 rainbow dots that fade at 2.4/s. This is visual feedback only.

### 7.5 Motion view (debug)
When on, a 168×126 panel in the bottom right shows the motion mask in pink on dark blue.

### 7.6 Pointer fallback
A mouse click or touch also works, in camera mode and in no-camera mode. A press first checks phoenixes (within `0.8r` of one burns you; the phoenix takes priority), then pops the topmost bubble under the pointer.

---

## 8. Lives and elimination

- Each player has 5 hearts. Each escaped bubble costs one heart, resets the combo, shakes the hearts pill and plays a miss sound.
- At 0 hearts the player is **out**: their stack dims, their disco ball (if running) stops, their remaining bubbles are cleared, and their phoenixes leave. Nothing more spawns on their side, and further escapes are ignored (a player's hearts never go below 0).
- One player: being out ends the game. Two players: the other player keeps playing; the game ends when both are out or the clock runs out.
- Level-up mercy (+1 heart) only applies to players still in the game.

**Implementation note:** bubbles are escaped from inside the bubble update loop. Do **not** remove the eliminated player's bubbles from inside the miss handler, because that shortens the list being iterated and crashes. Sweep them in a separate pass after the loop.

---

## 9. Disco ball super power

### 9.1 Earning charges
- `powerEvery` (default **500**, adjustable 100–2000 in steps of 50) points **earned** gives one charge.
- Progress counts points *gained* only, so a phoenix burn lowers your score but **never** sets back your progress toward the next disco ball.
- A player can hold up to **3 charges**. Points earned while full still move the next threshold forward (charges are not banked beyond 3).
- On earning a charge: an ascending arpeggio plays and the banner shows "Disco ball!" with "Clap once, loudly" (if the microphone is on) or "Press the space bar".

### 9.2 Power pill
- No charges: `⚡ {progress}/{powerEvery}`.
- Charged: `⚡ CLAP!` or `⚡ SPACE!` (depending on whether the microphone is on), prefixed `2× ` or `3× ` when holding several. The pill glows and pulses.

### 9.3 Firing
- Triggered by a **clap** (§10) or the **space bar**.
- **Two players:** one microphone cannot tell who clapped, so a trigger fires the ball for **every** player who holds a charge, is not out, and does not already have a ball running. Each spends one charge. **Design note:** this is a known limitation.
- The ball lasts **1500 ms**. Every **105 ms** it fires a laser at a **random** unpopped bubble in its owner's playfield (so "some of the bubbles", not a board wipe), popping it with normal scoring (combo included) and a zap sound. Each beam stays visible for 260 ms.

### 9.4 Look
- Position: horizontally centred in the owner's playfield, at 38 % of screen height. Radius: `0.085 × min(playfield width, H)`. Pops in over the first sixth of its life and shrinks away over the last 15 %.
- A sphere of **9 latitude × 20 longitude facets**, rotating (`spin = now/520` rad). Back-facing facets are skipped. Shading: dot product with light direction (−0.42, 0.5, 0.76). Some facets twinkle to full brightness (`sin(j·12.9898 + i·78.233 + 2·spin) > 0.86`), a set that shifts as the ball turns.
  - Modern: each facet is coloured `hsl(190 + (17j + 9i + 30·spin) mod 90, 55 %, 18 + 74·brightness %)`.
  - Arcade: four flat bands `#8ffff0 / #21f5d3 / #128f7f / #0a4a44` (white when twinkling), with dark wire edges.
- 8 rotating light shafts reaching to the screen edge (additive blend), a soft halo, and laser lines with a glowing tip from the ball to each bubble it hit.

---

## 10. Clap detection

Off by default. Turned on in Settings, which asks for microphone permission at that moment. If permission is refused, clapping stays off and the space bar still works.

### Algorithm (every frame while playing)
1. `peak` = the largest absolute sample in the latest audio frame (1024-sample window), 0–1.
2. `threshold = clapSlider / 100` (slider 4–60, default **18**, so 0.18).
3. `spike = peak > threshold AND peak > 4 × noiseFloor`.
4. Update the room's noise floor (starting value 0.01):
   - during a spike: `noiseFloor = 0.99·noiseFloor + 0.01·peak` (one clap barely moves it),
   - otherwise: `noiseFloor = 0.96·noiseFloor + 0.04·peak`.
   A room that stays loud raises the floor until the ratio test stops firing.
5. A clap is registered on a spike at least **500 ms** after the last one.

The requirement that a clap be both loud *and* four times the noise floor stops a noisy room from firing constantly.

### Calibration UI (Settings)
A live level meter (full scale = 0.6) with a marker at the current threshold. The bar flashes white for 220 ms when the meter registers a clap. Instructions: "Clap now — the bar shows what the microphone hears. Put the marker just under your clap." The meter runs its own simplified check (peak above threshold, 400 ms apart) because the game loop is paused while Settings is open.

---

## 11. Phoenix

- Appears from **level 2** onward, at most one new phoenix every `phoenixGap` ms (§4.3), assigned to a random live player, with at most 2 at a time per playfield.
- Crosses its owner's playfield horizontally, from a random side, at a random height between 18 % and 78 % of the screen (never at the very top or bottom). Speed `240 + 45·level + rand·60` px/s. Radius 46–60 px. It bobs vertically: `y + sin(phase + now/620) × (26–56 px)`.
- Plays a three-note square-wave cry when it appears.
- **Touching it** (motion coverage above `need` within `0.8r` of its centre, or a pointer press there): −**75** points (score floor 0), combo reset, a falling sawtooth screech, a 380 ms screen shake (up to ±9 px), a 90 ms orange edge flash, a floating "−75" and an ember burst.
- Each phoenix burns **once**. Afterwards it is drawn at 40–45 % opacity and is harmless.
- It leaves a trail of rising embers. It is removed once more than `3r` past its playfield's edge, or when its owner is out.
- **Look**: modern is a glowing bird with gradient wings, three flapping tail streamers, beak and eye. Arcade is a blocky sprite on a grid of `max(3, round(r/9))` px "pixels", with wings that snap between three poses, flat colours and an orange glow. Both face their direction of travel.

---

## 12. Two-player mode

- The screen is split down the middle by a **dashed vertical line** (12 on / 10 off, 2 px; white 30 % in modern, cyan 55 % in arcade).
- Each half is a **self-contained playfield**: player 1 has the left half and player 2 the right (as seen in the mirrored image, so each player stands on their own side). Bubbles spawn inside their owner's half, escape past that half's edges (including the centre line), and can only cost their owner hearts.
- Each player has their own score, hearts, combo, disco charges, disco ball and phoenixes.
- **Shared**: the clock, the level, tempo, and the clap/space trigger (§9.3).
- **Known cosmetic quirk**: a bubble near the centre line can be drawn overlapping the other half, and motion from either player can pop it. It still belongs to its owner, and points go to the owner.
- The game over screen names the winner (or a draw) and offers two name fields; each player gets their own leaderboard entry.

---

## 13. Sound

All sound is synthesised. Global on/off toggle. In the modern theme the default waveform is triangle; in arcade it is square. Envelope for each `tone(freq, delay, duration, volume, wave)`: exponential ramp from 0.0001 to the volume in 15 ms, then exponential decay to 0.0001 at the duration.

| Event | Recipe |
|---|---|
| Pop | Oscillator (sine in modern, square in arcade) sweeping `660·p → 150·p` Hz over 0.11 s, gain 0.28 → 0 over 0.13 s; plus 60 ms of decaying white noise through a band-pass filter at `1800·p` Hz (Q 1.1), gain 0.16. `p` = pitch factor from §6.2. |
| Miss | Sawtooth 300 Hz (0.16 s), then 190 Hz at +0.09 s (0.22 s). |
| Level up | Fanfare 523, 659, 784, 1047 Hz, 90 ms apart, 0.3 s each. |
| Game over | Descending 392, 349, 311, 262 Hz, sawtooth, 160 ms apart, 0.45 s each. |
| Charge earned | 523, 784, 1047, 1319 Hz, 70 ms apart. |
| Disco ball fired | Sine 261, 329, 392, 523, 659 Hz, 50 ms apart (0.5 s each), plus a 60 Hz thump (0.7 s). |
| Laser zap | Square wave from 1800–2700 Hz down to 420 Hz over 0.09 s, gain 0.11. |
| Phoenix appears | Square 880, 1180, 990 Hz, 70 ms apart. |
| Phoenix burn | Sawtooth sweeping 1500 → 320 Hz over 0.32 s. |
| Final 5 s tick | Square 880 Hz, 0.08 s. |

Audio must be unlocked by a user gesture (the start button or the clapping toggle).

---

## 14. Leaderboard

- **Top 10**, sorted by score, highest first.
- Entry: `{ name: string (≤ 24 chars), score: integer ≥ 0, level: integer ≥ 1 (the level shown at game end), id: string }`.
- Rows: rank (🥇 🥈 🥉, then "4." onward), name, "lvl N", score. Rows just saved are highlighted in yellow.
- Names are always rendered as **plain text**, never as markup.
- Everything that reads stored data cleans it: drop entries with no string name or a non-numeric score, cut names to 24 characters, round and clamp numbers.
- **Settings → Export** downloads `bubble-poke-leaderboard.json`:
  ```json
  { "game": "bubble-poke", "saved": "<ISO timestamp>", "scores": [ …entries… ] }
  ```
- **Import** accepts that format or a plain array of entries, merges by `id` (skipping duplicates), and keeps the top 10. It reports "Imported N scores.", "Imported N scores — none were new.", "No scores found in that file." or "That file could not be read as a leaderboard."
- **Clear leaderboard** (in Settings) deletes all entries **and** the best score. Cannot be undone.

---

## 15. Settings and persistence

### 15.1 Settings screen
Opening Settings during a game **pauses** it; closing resumes it with no time lost. Esc closes Settings. Each change shows a short confirmation line.

| Section | Control | Range / options | Default | Effect |
|---|---|---|---|---|
| Language | Instructions | English / Svenska | English | Immediate |
| Look | Style | Modern / Arcade | Modern | Immediate |
| Game | Game length | 20–300 s, step 5 | **65 s** | Next game. Shows "{length} — that's {n} levels." (`n = ceil(length/13)`) |
| Game | Disco ball every | 100–2000 pts, step 50 | 500 | Next game |
| Game | Poke sensitivity | 4–60 | 30 | Immediate. "Left if bubbles pop by themselves, right if you have to flail." |
| Game | Sound | On / Off | On | Immediate |
| Game | Motion view | On / Off | Off | Immediate |
| Clapping | Clap to set off the disco ball | On / Off | Off | Turning it on asks for the microphone |
| Clapping | How loud is a clap | live meter | — | — |
| Clapping | Clap threshold | 4–60 | 18 | Immediate |
| Leaderboard | Saved in this browser | Export / Import | — | Shows the number of saved scores |
| Leaderboard | Clear leaderboard | button | — | Immediate |

The player count is set on the start screen, not in Settings.

### 15.2 Storage keys (`bubblepoke.*`)
| Key | Value |
|---|---|
| `best` | Best score (number) |
| `board` | JSON array of leaderboard entries |
| `name` | Last name entered by player 1 |
| `len` | Game length in seconds |
| `v` | Settings schema version (currently `"2"`) |
| `sens` | Poke sensitivity |
| `sound` | `"1"` / `"0"` |
| `debug` | Motion view, `"1"` / `"0"` |
| `power` | Disco-ball points |
| `clap` | Clapping on, `"1"` / `"0"` |
| `clapsens` | Clap threshold |
| `theme` | `"modern"` / `"arcade"` |
| `lang` | `"en"` / `"sv"` |
| `players` | `1` / `2` |

**Settings migration:** settings are remembered, so a returning player would never see a new default. When a default must actually take effect, bump the version: if the stored `v` differs from the current version, overwrite only the affected key(s) and store the new version. Version 2 reset `len` to 65.

All storage access must tolerate storage being unavailable (private mode): reads fall back to defaults and writes fail silently.

---

## 16. Pausing and time accounting

All timers use wall-clock time. Anything that takes time away from play must give it back by shifting every stored timestamp forward by the time lost: game start, game end, last spawn, last phoenix, last clap, banner expiry, shake expiry, each player's disco expiry, last beam and combo window, each beam's birth, and each bubble's birth (so the 350 ms grace period is not lost).

This happens on:
- **Settings opened during a game** (paused until closed),
- **app/tab sent to the background** (visibility hidden → visible).

The frame time step is capped at 50 ms, so a stall never teleports objects.

### Camera errors (start screen)
| Cause | Message |
|---|---|
| No camera API | This browser will not share a camera with the page. Chrome, Safari or Firefox over http://localhost all work. |
| Permission denied | Camera permission was denied. Allow it in the address bar, then press the button again. |
| No device | No camera found on this computer. |
| Device busy | The camera is busy in another app (Zoom, Photo Booth…). Close it and try again. |
| Anything else | Could not open the camera: {error} |

---

## 17. Themes

### 17.1 Modern (default)
- Font: rounded sans ("Baloo 2", Trebuchet MS, Avenir Next, system-ui).
- Background `#0b1020`; start screen uses a radial gradient `#21306a → #0b1020`.
- Glassy translucent pills: `rgba(11,16,32,0.48)` with backdrop blur and a white 22 % border, fully rounded.
- Accent pink `#ff4fa3`; gradient `#7ee8ff → #ff4fa3 → #ffe27a` for the title, banner and primary button.
- Bubbles: soft radial gradient (88 % → 66 % → 52 % lightness, decreasingly opaque), a light rim, a white highlight ellipse at the top left, and a small secondary glint. Golden bubbles have a thicker rim and a ★.
- Particles: round dots.

### 17.2 Arcade (1980s)
- Font: monospace, UPPERCASE, wide letter spacing.
- Black `#05060a`; screens have a faint 44 px cyan grid.
- Square-cornered panels with a 3 px cyan `#21f5d3` border and outer glow.
- Neon palette: yellow `#ffe64d` (score, primary button), cyan `#21f5d3` (timer, borders), magenta `#ff2ea6` (selected toggles, combo), red `#ff3b6b` (hearts, warnings).
- Title and banner: yellow with magenta/cyan chromatic offset shadows (±3–4 px).
- **CRT overlay** over everything: scanlines (1 px dark every 4 px) plus a vignette, with a slow 7 s opacity drift. **It must never strobe.**
- Bubbles: hard-edged neon rings with a dark fill, a glow, an inner ring, and a **square** glint. Particles are square pixels. The phoenix is a sprite (§11). The disco ball is banded (§9.4).
- Sound uses square waves (§13).

---

## 18. Localisation

Two languages, English (`en`) and Swedish (`sv`). Every string a player can see has both versions. Strings use `{0}`, `{1}` placeholders. A missing Swedish key falls back to English.

Swedish notes: "every 13 seconds" is written with the ordinal: "var 13:e sekund". The game is called "Bubble Poke" in both languages.

### 18.1 String table

| Key | English | Svenska |
|---|---|---|
| tag | Your camera becomes the game board.⏎Wave a hand at a bubble to pop it. | Din kamera blir spelplanen.⏎Vifta med handen mot en bubbla så spricker den. |
| r1 / r1e | Pop bubbles for points. / Small ones are worth more than big ones. | Spräck bubblor för poäng. / Små är värda mer än stora. |
| r2 / r2e | Golden star bubbles are worth 50. / They fly faster, too. | Gyllene stjärnbubblor är värda 50. / De flyger snabbare också. |
| r3 / r3e | From level 2 a phoenix swoops across. / Don't touch it — it burns 75 points. | Från nivå 2 sveper en fenix förbi. / Rör den inte — den bränner 75 poäng. |
| r4 / r4e | Every {0} points you earn a disco ball. / Clap once, loudly, and lasers pop bubbles for you. | Var {0} poäng får du en discokula. / Klappa en gång, högt, så spräcker lasrar bubblor åt dig. |
| r5 / r5e | Let five bubbles escape and the game ends early. / Each new level gives one heart back. | Om fem bubblor kommer undan tar spelet slut. / Varje ny nivå ger tillbaka ett hjärta. |
| r6 / r6e | {0}, getting faster and faster. / A harder level every {1} seconds. Change the length in Settings. | {0}, och det går fortare och fortare. / En svårare nivå var {1}:e sekund. Ändra längden i Inställningar. |
| r7 | Add your name to the leaderboard when the game ends. | Skriv in ditt namn på topplistan när spelet är slut. |
| howto | How to play | Så spelar du |
| p1 / p2 | 1 player / 2 players | 1 spelare / 2 spelare |
| start | Turn on camera | Slå på kameran |
| nocam | No camera — play with the mouse | Ingen kamera — spela med musen |
| settings / restart / done | Settings / Restart / Done | Inställningar / Börja om / Klar |
| privacy | Your camera feed never leaves this computer.⏎Nothing is recorded, uploaded, or saved. | Din kamerabild lämnar aldrig den här datorn.⏎Ingenting spelas in, laddas upp eller sparas. |
| score / levelLbl / level | SCORE / LEVEL / Level {0} | POÄNG / NIVÅ / Nivå {0} |
| lvlSub0 | Bubbles float up from below | Bubblor stiger underifrån |
| lvlSub1 | Sides open up — and the phoenix wakes | Sidorna öppnas — och fenixen vaknar |
| lvlSub2 | From every direction now | Från alla håll nu |
| lvlSub3 | Faster. Smaller. Good luck. | Snabbare. Mindre. Lycka till. |
| combo | {0} in a row! ×{1} | {0} i rad! ×{1} |
| discoBall | Disco ball! | Discokula! |
| clapNow / pressSpace | Clap once, loudly / Press the space bar | Klappa en gång, högt / Tryck på mellanslag |
| clapPill / spacePill | ⚡ CLAP! / ⚡ SPACE! | ⚡ KLAPPA! / ⚡ MELLANSLAG! |
| over / newBest | Game Over / New best! | Spelet slut / Nytt rekord! |
| outOfTime | Time! You reached level {0}. | Tiden är ute! Du nådde nivå {0}. |
| outOfLives | Five bubbles got away on level {0}. | Fem bubblor kom undan på nivå {0}. |
| scored | You scored {0}. | Du fick {0} poäng. |
| yourName / saveScore / again | Your name / Save score / Play again | Ditt namn / Spara poäng / Spela igen |
| lvlShort | lvl {0} | nivå {0} |
| playerWins / draw | Player {0} wins! / It's a draw! | Spelare {0} vinner! / Oavgjort! |
| finalScores | Player 1: {0} · Player 2: {1} | Spelare 1: {0} · Spelare 2: {1} |
| nameP | Player {0} | Spelare {0} |
| overHint | Settings has the language, the game length and the disco ball. | I Inställningar finns språket, speltiden och discokulan. |
| hLang / hLook / hGame / hClap / hBoard | LANGUAGE / LOOK / GAME / CLAPPING / LEADERBOARD | SPRÅK / UTSEENDE / SPEL / KLAPPA / TOPPLISTA |
| sLang / sLangSub | Instructions / Language for the instructions and buttons. | Instruktioner / Språk för instruktioner och knappar. |
| sStyle | Style | Stil |
| sStyleModern | Modern — soft colours and glassy panels. | Modern — mjuka färger och glasiga rutor. |
| sStyleArcade | Arcade — neon phosphors, scanlines and bleepy sound. | Arkad — neon, rutlinjer och pipiga ljud. |
| sLen / sLenSub | Game length / A harder level every {0} seconds, and it speeds up all the way through. | Speltid / En svårare nivå var {0}:e sekund, och det går fortare hela tiden. |
| sPower / sPowerSub | Disco ball every / Points you must earn to charge the super power. | Discokula var / Poäng du måste samla för att ladda superkraften. |
| sPoke / sPokeSub | Poke sensitivity / Left if bubbles pop by themselves, right if you have to flail. | Känslighet / Åt vänster om bubblor spricker av sig själva, åt höger om du måste vifta vilt. |
| sSound / sSoundSub | Sound / Pops, fanfares and the phoenix. | Ljud / Smällar, fanfarer och fenixen. |
| sMotion / sMotionSub | Motion view / Shows what the camera thinks is moving. | Rörelsevy / Visar vad kameran tycker rör sig. |
| sClapOn | Clap to set off the disco ball | Klappa för att starta discokulan |
| sClapLoud / sClapLoudSub | How loud is a clap / Clap now — the bar shows what the microphone hears. Put the marker just under your clap. | Hur högt är en klapp / Klappa nu — stapeln visar vad mikrofonen hör. Sätt markören strax under din klapp. |
| sClapThr / sClapThrSub | Clap threshold / Lower if claps are missed, higher if the game fires by itself. | Klappgräns / Lägre om klappar missas, högre om spelet startar av sig självt. |
| sSaved | Saved in this browser | Sparad i den här webbläsaren |
| sClear / sClearSub | Clear leaderboard / Also resets the best score. This cannot be undone. | Rensa topplistan / Nollställer även bästa poängen. Går inte att ångra. |
| on / off | On / Off | På / Av |
| export / import / clear | Export / Import / Clear | Exportera / Importera / Rensa |
| boardEmpty | No scores yet. They save in this browser as you play. | Inga poäng än. De sparas i den här webbläsaren när du spelar. |
| boardCount | {0} scores saved in this browser. Export makes a file you can keep. | {0} poäng sparade i den här webbläsaren. Exportera för att få en fil du kan behålla. |
| boardCount1 | 1 score saved in this browser. Export makes a file you can keep. | 1 poäng sparad i den här webbläsaren. Exportera för att få en fil du kan behålla. |
| paused | Game paused. | Spelet är pausat. |
| micOff | Off — the space bar sets off the disco ball instead. | Av — mellanslag startar discokulan i stället. |
| micOn | Microphone on. Clap once to set off the disco ball. | Mikrofonen är på. Klappa en gång för att starta discokulan. |
| micWait | Turning the microphone on… | Slår på mikrofonen… |
| micDenied | Microphone permission denied. Allow it in the address bar and try again. | Mikrofonen nekades. Tillåt den i adressfältet och försök igen. |
| micBusy | Could not open the microphone: {0} | Kunde inte öppna mikrofonen: {0} |
| micNone | This browser will not share a microphone with the page. | Den här webbläsaren delar inte mikrofonen med sidan. |
| clapStaysOff | Clapping stays off — the space bar still works. | Klappa förblir av — mellanslag fungerar ändå. |
| camNone | This browser will not share a camera with the page. Chrome, Safari or Firefox over http://localhost all work. | Den här webbläsaren delar inte kameran med sidan. Chrome, Safari eller Firefox via http://localhost fungerar. |
| camDenied | Camera permission was denied. Allow it in the address bar, then press the button again. | Kameran nekades. Tillåt den i adressfältet och tryck på knappen igen. |
| camMissing | No camera found on this computer. | Ingen kamera hittades på den här datorn. |
| camBusy | The camera is busy in another app (Zoom, Photo Booth…). Close it and try again. | Kameran används av ett annat program (Zoom, Photo Booth…). Stäng det och försök igen. |
| camFail | Could not open the camera: {0} | Kunde inte öppna kameran: {0} |
| seconds / minutes / minsec | {0} seconds / {0} minutes / {0} min {1} sec | {0} sekunder / {0} minuter / {0} min {1} sek |
| lenSaid | {0} — that's {1} levels. | {0} — det blir {1} nivåer. |
| lenSaidLater | {0} — takes effect next game ({1} levels). | {0} — gäller nästa spel ({1} nivåer). |
| powerSaid | A disco ball every {0} points. | En discokula var {0} poäng. |
| powerSaidLater | A disco ball every {0} points — takes effect next game. | En discokula var {0} poäng — gäller nästa spel. |
| cleared | Leaderboard cleared. | Topplistan rensad. |
| exported | Saved as bubble-poke-leaderboard.json. | Sparad som bubble-poke-leaderboard.json. |
| nothingExport | Nothing to export yet. | Inget att exportera än. |
| imported / importedNone | Imported {0} scores. / Imported {0} scores — none were new. | Importerade {0} poäng. / Importerade {0} poäng — inga var nya. |
| importEmpty / importBad | No scores found in that file. / That file could not be read as a leaderboard. | Inga poäng hittades i filen. / Filen gick inte att läsa som en topplista. |

⏎ marks a line break.

---

## 19. Keyboard

| Key | Action |
|---|---|
| Space | Fire the disco ball (any player with a charge) |
| F | Full screen |
| Esc | Close Settings |
| Enter (in a name field) | Save score |

Keyboard shortcuts are **ignored while typing in a text field**, so a name containing "f" does not toggle full screen.

---

## 20. Acceptance checklist

A port should pass these. The camera and microphone items need real hardware.

**Flow**
- [ ] A fresh install shows a 1:05 timer, English, 1 player, "How to play" collapsed, and an empty leaderboard.
- [ ] "No camera" mode starts a game that can be played with mouse or touch.
- [ ] A 65 s game passes through levels 1–5, with a banner and fanfare at 13, 26, 39 and 52 s.
- [ ] Bubbles only rise in level 1, come from three sides in level 2, and from all four from level 3.
- [ ] No bubble ever crosses its playfield in under 1.15 s, at any level or tempo.
- [ ] Opening Settings mid-game pauses it; the clock shows the same time on return. Same when the app is backgrounded.

**Scoring and lives**
- [ ] Score never goes below 0, including after phoenix burns at a low score.
- [ ] The combo multiplier reaches ×2 on the 4th consecutive pop within 1.4 s.
- [ ] Five escapes end a one-player game early; a level-up restores one heart (to a maximum of 5).
- [ ] A phoenix burns once only; a spent phoenix is translucent and harmless.

**Disco ball**
- [ ] A charge arrives at exactly `powerEvery` points earned; a burn does not reduce progress toward it.
- [ ] Charges cap at 3.
- [ ] The space bar fires it; it lasts 1.5 s and pops some, but not necessarily all, bubbles.
- [ ] With the microphone on and calibrated, one loud clap fires it; steady room noise does not.
- [ ] Changing "Disco ball every" in Settings updates the pill and the rules text without errors.

**Two players**
- [ ] No bubble or phoenix is ever assigned to or escapes through the wrong half.
- [ ] Both halves stay roughly equally busy.
- [ ] When one player is eliminated, their half empties and stays empty, their hearts stay at 0 (never negative), and the other player continues.
- [ ] The winner (or draw) is announced and both names are saved.

**Persistence**
- [ ] The leaderboard and settings survive a reload.
- [ ] Export, then Clear, then Import restores the same leaderboard; importing the same file twice reports "none were new".
- [ ] A name like `<b>x</b>` is shown literally, not as bold.
- [ ] A returning player whose stored length was the old default of 90 s gets 65 s once (migration).

**Themes and language**
- [ ] Switching language updates every visible string, including the rules, the settings descriptions and the HUD.
- [ ] Arcade mode changes fonts, colours, bubbles, phoenix, disco ball, particles and sounds, and adds the CRT overlay, which does not flicker.

---

## 21. Known limitations and open tuning questions

- **Motion, not hands.** A swinging ponytail, a sibling walking past or a moving shadow can pop bubbles. The sensitivity slider and the 45 % global-motion gate reduce this but do not eliminate it.
- **One microphone, two players.** A clap cannot be attributed to a player (§9.3).
- **The centre line is not a wall.** A bubble that overlaps the divider can be popped by either player (§12).
- **Tuning that has not been playtested with the target player:** `MIN_CROSS_SEC` (1.15), `TEMPO_PER_SEC` (0.013), the level speed step (`+0.55` per level), and the default clap threshold (18). The clap path has not yet been confirmed on real hardware.
- **Arcade font.** It uses the system monospace font. A bundled pixel font (e.g. Press Start 2P, shipped locally to stay offline) would look more authentic.
