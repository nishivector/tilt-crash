# Tilt Crash — Design Document
Round 16

---

## The One Thing

**Momentum is your weapon and your judge.** The same force that annihilates an enemy line will slide your ball into the void three seconds later. Every tilt decision is a negotiation with physics you can't fully control — and you learn to love that feeling.

---

## Identity

**Game Name:** Tilt Crash
**Tagline:** *Iron rolls. Everything breaks.*

**What is the player:** You are the board — the arena itself. You don't control the ball; you shape the world around it.

**World Feel:** A brutalist foundry floor suspended in amber void, lit from below by molten-orange furnace light. The iron ball is ancient, impossibly heavy, indifferent to your intentions — it remembers every speed you've given it.

**Emotional Experience:** Controlled destruction that keeps threatening to become uncontrolled destruction. The high is a long, slow crush across a packed enemy cluster; the terror is watching your momentum carry you two inches past where you wanted to stop.

**Reference DNA:**
- **Labyrinth (classic marble maze):** Pure tilting input. But where Labyrinth rewards caution, Tilt Crash punishes it — slow is dangerous because enemies keep spawning.
- **Thumper:** Precision under pressure; a rhythm undercurrent you feel before you hear. The 118 BPM pulse should feel *physical*, not decorative.
- **Katamari Damacy:** The joy of mass. As the ball gains speed it feels heavier, realer. Weight as game feel.
- **SUPERHOT:** Time-under-control illusion. Players will slow their inputs instinctively thinking the ball will respond. It doesn't.
- **R-Type:** Enemy placement is compositional — patterns exist to be read and exploited, not just dodged. Levels are puzzles.

---

## Visual Spec

**Background:** `#C07838` — amber, warm, furnace-hot. Flat fill, no gradient, no texture. This is the void the arena floats in.

**Primary (Ball & Board):** `#1A1A1A` — near-black iron. The board surface and the ball. Matte, not shiny. Weight over chrome.

**Secondary (Enemies):** `#E8D5A3` — pale brass. Enemy cubes are lighter than the ball, emphasizing the iron ball's dominance. They shatter into this color.

**Accent (UI, Score, Danger):** `#FF4B1F` — ember red-orange. Score counters, danger indicators, edge-glow when ball approaches board rim. Also used for kill-streak flash.

**Board Edge Color:** `#8B5A1A` — dark burnt sienna. The board rim, 4px border. Visible but not distracting.

**Particle/Debris Color:** `#FFB347` — golden amber. Enemy death fragments catch the amber light.

**Bloom:** Strength `0.6`, Threshold `0.72`. Applied only to the ball and enemy death bursts. The board does not bloom. This keeps weight in the ball — it *glows* slightly with its own kinetic heat at high speed (bloom increases linearly from 0.3 at rest to 0.6 at max velocity).

**Camera:** Fixed orthographic, top-down, 15° tilt toward the player (slight 3/4 perspective, not full isometric). The board fills 72% of viewport width. Camera does NOT follow anything — it is locked. The arena is always fully visible. Board shadow falls directly downward, offset `4px 6px`, color `#7A3D10` at 60% opacity.

**Player Silhouette in 5 words:** Heavy sphere, iron, slightly oiled.

---

## Sound Spec

**Music: 118 BPM**

Instrumentation: Kick drum on every beat, sidechained to a low analog synth bass that pumps with each hit. Snare on beats 2 and 4 with a sharp metallic transient (like hitting iron pipe). A high-passed rhythm guitar loop (palm-muted, industrial tone) runs the 8th-note groove. Over this: a single synth lead — sawtooth, slight chorus, playing a pentatonic motif that repeats every 4 bars. No melody variation until music state changes. The track should feel like factory machinery that has learned to swing. No vocals. No softness.

**Music State Changes:**

| Trigger | Response |
|---|---|
| Game start | Music fades in over 1.2 seconds from silence. Full arrangement immediately. |
| Kill streak ×3 | Add a distorted second synth layer (octave above lead, +8% drive). Lasts 4 bars. |
| Kill streak ×7 | Snare doubles to 16th notes for 2 bars. Creates urgency without chaos. |
| Ball near edge (within 12% of board boundary) | Low-pass filter drops from 8kHz to 2.4kHz over 0.4 seconds. Everything goes muffled, claustrophobic. Releases when ball moves to safety. |
| Level complete | Music cuts to single kick hit + bass note. Then 1.5s silence. Then next level music fades in at slightly higher tempo feel (same BPM but tighter quantization). |
| Game over | Music cuts instantly. Single low resonant tone (120Hz sine, 2.2s decay). |

**6 SFX with Tone.js hints:**

1. **Ball Roll (continuous):** Low rumble that scales with velocity. `Tone.Noise` type `'brown'`, gain mapped from `0.0` to `0.35` based on ball speed. Filter: `Tone.Filter` lowpass at 280Hz. Starts at ball speed > 0.5 units/s, cuts at rest.

2. **Enemy Crush:** Sharp metallic impact. `Tone.MembraneSynth` with `pitchDecay: 0.02`, `octaves: 4`, `frequency: 180`. Plus `Tone.MetalSynth` burst at 400Hz, `envelope.decay: 0.08`. Layered — the thud and the clank.

3. **Board Tilt (input):** Subtle creak. `Tone.Synth` with `oscillator.type: 'sawtooth'`, frequency sweep from 90Hz to 140Hz over 0.15s on pointerdown. Reversed (140→90) on pointerup. Very quiet — `gain: 0.08`. Present but subliminal.

4. **Ball Near Edge Warning:** A faint, fast heartbeat pulse. `Tone.Synth`, `oscillator.type: 'sine'`, frequency `55Hz`, `envelope.attack: 0.001`, `envelope.decay: 0.08`, triggered every 0.3s when ball is within 15% of edge. Gain increases as ball gets closer: `0.0` at 15%, `0.25` at edge.

5. **Level Complete:** A single deep metal gong. `Tone.MetalSynth`, `frequency: 60`, `harmonicity: 5.1`, `modulationIndex: 32`, `envelope.decay: 1.8`, `envelope.sustain: 0`. Resonant and conclusive.

6. **Ball Fall (game over):** Doppler-style pitch drop. `Tone.Synth`, `oscillator.type: 'triangle'`, frequency glides from `220Hz` to `40Hz` over `0.9s` using `frequency.rampTo`. Gain: `0.4` falling to `0.0`. Accompanied by a very short `Tone.Noise` burst (white, `decay: 0.12`) for the impact suggestion.

---

## Mechanic Spec

**Core Loop:** Tip the board to build momentum, aim the ball into enemy clusters, then recover control before your own mass kills you.

**Primary Input:**

- `pointerdown`: Begins tilt. Records initial pointer position as anchor.
- `pointermove`: The board rotates toward the drag direction. Tilt angle = pointer delta × sensitivity (0.045 radians per pixel). Tilt is applied to both axes (X and Y). The board is a 3D object rotating in 2D screen space — the tilt creates a physical slope the ball rolls down.
- `pointerup`: Board begins returning to flat. Spring return: not instant. Uses damped spring — `stiffness: 140`, `damping: 18`. The board oscillates slightly before settling. This means releasing too fast can send the ball in an unintended direction.
- **Touch support:** Same as pointer events. Single-touch only. Multitouch ignored.
- **No cursor/crosshair shown.** The board IS the input device.

**Key Physics Values:**

| Parameter | Value |
|---|---|
| Board size | 400×400 units (square) |
| Ball radius | 18 units |
| Ball mass | 14.0 (arbitrary but consistent units) |
| Max tilt angle | ±28° (both axes) |
| Tilt input sensitivity | 0.045 rad/px |
| Spring stiffness (return) | 140 |
| Spring damping (return) | 18 |
| Gravity constant | 9.8 units/s² |
| Rolling friction | 0.012 (very low — iron on iron) |
| Max ball velocity | 22 units/s |
| Ball velocity at which bloom activates | 8 units/s |
| Edge "danger zone" width | 12% of board half-width (24 units) |
| Ball falls off when | Center exits board boundary |
| Enemy cube size | 20×20 units |
| Enemy mass (for collision) | 1.0 (light — they scatter) |
| Collision impulse transfer | Ball loses 8% velocity per enemy hit; enemy flies at 4× ball's speed component |

**Win/Lose Conditions:**

- **Win a level:** Destroy all enemies on the board within the time limit.
- **Lose:** Ball exits the board boundary (falls off edge). Instant game over — no lives, no retry within round. Start from Level 1.
- **No health, no lives.** One mistake ends everything. The ball is yours to protect.

**Score System:**

- Base kill: **100 points** per enemy destroyed.
- **Combo multiplier:** Each enemy killed within 1.5 seconds of the previous kill increases multiplier by 1× (cap: 8×). Multiplier resets after 1.5s gap.
- **Speed bonus:** If ball speed > 16 units/s at moment of kill, +50 bonus per kill.
- **Level clear bonus:** `(time_remaining_seconds × 20) + (combo_peak × 150)`.
- **No penalty for slow play** except the time limit — the game doesn't shame you, it just closes the window.
- Score displayed as a 6-digit counter top-center, accent color `#FF4B1F`, monospace font, no decimals.

**Difficulty Curve (5 Levels):**

| Level | Enemies | Time Limit | Enemy Behavior | Special |
|---|---|---|---|---|
| 1 | 8 | 45s | Static | None |
| 2 | 12 | 50s | Static + 2 drifting slowly (0.5 units/s) | None |
| 3 | 16 | 55s | 4 drifting (1.2 units/s), rest static | Board return spring damping reduced to 14 (more oscillation) |
| 4 | 20 | 55s | 8 drifting (1.8 units/s), 2 bouncing off walls | Max tilt angle reduced to ±22°; board input sensitivity 0.038 |
| 5 | 24 | 60s | 12 drifting (2.4 units/s), 4 bouncing, 2 chasing ball slowly (chase speed 1.0 units/s) | Board return spring damping 10 (very oscillatory); rolling friction reduced to 0.008 |

---

## Level Design

**Level 1 — "First Iron"**
- What's new: Everything. The player learns that dragging tips the board.
- 8 static enemies arranged in a loose 2×4 grid, centered on the board. Generous spacing between them — each is an individual target.
- Parameters: Standard all. Time: 45s. No drifting enemies.
- Design intent: The player will accidentally roll the ball near the edge at least once. That near-miss teaches everything. The board's spring return should feel like a living thing — that's the tutorial.

**Level 2 — "Drift"**
- What's new: Two enemies slowly drifting left-right. Player must time their approach.
- 10 static enemies in a scattered pattern, 2 drifters moving on a horizontal axis across the center of the board.
- Parameters: Standard. Time: 50s.
- The drifters move at 0.5 units/s. They reverse direction when they reach 80% of board edge. Their movement is a clock — if you don't build momentum fast enough, the window to hit them while they cluster with statics closes.

**Level 3 — "Momentum Debt"**
- What's new: Reduced spring damping makes the board feel drunk after releasing input.
- 12 static enemies in tight clusters of 3 near each corner, 4 drifters moving diagonally.
- Parameters: `damping: 14`. Time: 55s. Drifters: 1.2 units/s.
- The corners are the score opportunities — cluster kills. But the drifters threaten to push statics off the board? No — enemies do not fall off. They bounce at board edge. But they can scatter your planned path.
- Design intent: The player discovers that releasing the board has *consequences*. That wobble is no longer charming. It is the enemy.

**Level 4 — "Tight Angles"**
- What's new: Tilt range capped at ±22°. Input sensitivity reduced. Two enemies bounce wall-to-wall.
- 14 static enemies, 4 drifters, 2 bouncers (bouncing between opposite walls, high speed: 2.8 units/s).
- Parameters: `maxTilt: 22°`, `sensitivity: 0.038`, `damping: 14`. Time: 55s.
- The reduced tilt range is a revelation of what the player has been relying on. Subtle inputs suddenly feel necessary for the first time. Bouncers cross the board in under 3 seconds — they punish slow ball speeds.
- Design intent: Precision under constraint. The player's mental model must upgrade.

**Level 5 — "Gravity Engine"**
- What's new: Nearly frictionless surface, two enemies slowly chase the ball, very low spring damping.
- 14 static enemies scattered, 8 drifters (fast), 4 bouncers, 2 chasers.
- Parameters: `friction: 0.008`, `damping: 10`, `maxTilt: 22°`, `sensitivity: 0.038`. Time: 60s.
- Chasers move at 1.0 units/s toward ball position. They're not fast — but they are patient. A player who stalls too long will be surrounded.
- The near-frictionless surface means the ball stores everything. A tilt that would have been safe on Level 1 will send the ball to the edge in 0.4 seconds on Level 5.
- Design intent: The player must plan three moves ahead. Every tilt has a recovery plan. This is when the game becomes chess played at 118 BPM.

---

## The Moment

**Level 3, mid-run, combo at 5×.** The player has a tight cluster of three enemies in the upper-left corner. They drag hard right to build a cross-board run. The ball accelerates, hits all three enemies in a single pass — bloom pulses, the crush SFX triple-fires — the combo multiplier flashes to 8×. The score jumps by 2,400 points in one second. But the ball is now traveling at 19 units/s toward the left wall, two drifters are converging, and the spring damping is already fighting the board. The player has to immediately drag left to counter — and for a moment, it looks like they're going to eat everything, enemies and edge both. They thread it. The board settles. The ball rolls to a stop near center. No enemy near it. The player breathes.

That moment — the three-kill pass that almost kills you — is what this game is.

---

## Emotional Arc

**First 30 seconds:** Curiosity and discovery. The tilt feels surprisingly physical. The ball's weight is immediately convincing. The first enemy hit produces a satisfying crunch. The player tries to be precise and discovers precision is impossible at speed — then laughs when the ball does something unexpected. They die (almost certainly) to the edge. They are not frustrated. They are hooked.

**After 2 minutes:** Investment and flow. The 118 BPM pulse has synced with the player's decision tempo without them noticing. They are making micro-corrections instinctively. The board's spring feels like a tool rather than a liability. They are surviving longer. They start to *plan* — "if I tip right to reach that cluster, I need to already be preparing the counter-tip." The game is no longer reactive. It is a conversation.

**Near win (final 10 enemies on Level 5):** Everything is wrong and everything is working. The friction is nearly gone. The spring barely returns. Two enemies are chasing. The remaining statics are scattered — hard pattern to solve. The player is deep in the zone, making corrections so fast their hands are ahead of their thoughts. The music's doubled snare feels like a pulse monitor. Every kill now feels enormous. When the last enemy shatters, the silence of the level-complete cut is the most satisfying sound in the game.

---

## This game's identity in one line

**"This is the game where you build up speed to win, then spend the rest of your life paying for it."**

---

## Start Screen

**Layout:** Game title centered, board visible in background at 60% opacity, ball in motion.

**Idle Animation (game-world specific):**
The arena runs on autopilot. A ghost ball — semi-transparent, `#1A1A1A` at 40% opacity — rolls a slow figure-8 path across the board surface. The board tilts gently to guide it, performing small, meditative oscillations. Three enemy cubes sit at the figure-8 crossing points. Every 6 seconds, the ghost ball rolls through a crossing and one enemy shatters in slow motion (0.4× speed) — debris fans outward, amber particles arc and fade. The enemy respawns 2 seconds later. The board never fully stops moving. This is not decorative: it is a live demonstration of the core mechanic, playing itself forever.

**SVG Overlay:**

**Option A — Title Glow (required):**
Title text "TILT CRASH" in a heavy, condensed sans-serif (weight 800). SVG `<text>` element with:
- Fill: `#1A1A1A`
- Outer glow via `<feGaussianBlur>` stdDeviation `8`, composited with `feComposite operator="over"`, glow color `#FF4B1F` at 70% opacity
- A second glow pass: stdDeviation `24`, color `#C07838` at 40% — the amber halo that makes the title feel warm-hot
- Letter-spacing: `0.18em`
- The glow pulses with a slow sinusoidal animation: glow intensity cycles from 60% to 100% opacity over 2.4 seconds (not synced to BPM — deliberately lazy, pre-game feel)

**Option B — Board Ring (fitting — adds spatial grounding):**
An SVG circle, `r="220"`, centered behind the title. Stroke only, no fill. Stroke: `#FF4B1F`, `stroke-width: 1.5`, `stroke-dasharray: "12 6"`. Rotates at 0.6 RPM clockwise, continuously. This suggests the arena, the ball's orbit, the dangerous edge. Opacity: 0.35 — present but not competing with the title.

**Option C — Amber Drip Lines (fitting — reinforces material weight):**
3 thin vertical SVG lines descending from the top of the viewport to the board area. Each `stroke: #FF4B1F`, `stroke-width: 1`, `opacity: 0.2`. They don't move — they are suggestions of liquid amber, static drips. Spaced at 25%, 50%, 75% of viewport width. Extremely subtle. Communicates: something is always about to fall.

---

## Implementation Notes for Programmer

- Physics should run at a fixed timestep of `1/60s`, independent of render framerate. Ball velocity and tilt angle are the only state that needs to persist between frames.
- The board is a 2D rectangle rendered with a slight perspective transform (CSS `perspective: 800px`, `rotateX(8deg)` in base state) to suggest 3D. Actual tilt is additional CSS `rotateX` and `rotateY` applied dynamically.
- Enemy collision: treat enemies as solid squares. Ball collision is circle-square. On collision, apply impulse to enemy, subtract 8% from ball velocity magnitude (direction unchanged). Enemy flies off in ball's direction component.
- Enemies do not fall off the board. They bounce at board edges with `restitution: 0.7`.
- The score combo timer should be visualized: a thin `#FF4B1F` line under the score counter depletes over 1.5 seconds. When an enemy dies, the line resets to full. This is the player's combo clock.
- All animations (bloom, glow, particle arcs) should be achievable with CSS + Canvas 2D. No WebGL required unless the programmer prefers it.
- The board's spring return must be *felt*. If it feels instant, increase `stiffness`. If it feels like jelly, increase `damping`. Tune on mobile first — the input is finger drag, and the game must feel heavy on a touchscreen.
- Start screen idle animation should loop seamlessly. Ghost ball path is a Lissajous curve (a=1, b=2, δ=π/2) scaled to board dimensions.
