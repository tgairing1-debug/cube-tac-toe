# AI Opponent — Design Spec

**Date:** 2026-05-22  
**Project:** Cube Tac Toe (`index.html`)  
**Scope:** Add a computer opponent with three difficulty levels to the existing 2-player hot-seat game.

---

## Context

Cube Tac Toe is a single self-contained `index.html` (Three.js, no build step). Players alternate placing X/O marks on a 3×3 Rubik's Cube, then must make one standard cube turn; a player wins by getting three marks in a row on any single face after the turn completes. The game is currently hot-seat only.

This spec adds a computer opponent so the game is playable solo, while keeping the existing 2-player mode. The AI must pick *both* a sticker (where to place) and a cube move — the move scrambles the board, so the AI's approach is heuristic evaluation rather than deep search.

---

## Decisions Made During Brainstorming

| Topic | Decision |
|---|---|
| Mode selection | Mode selector panel on New Game; both modes kept |
| Difficulty | Three levels: Easy / Medium / Hard |
| Player side | Randomly assigned each AI game (coin flip) |
| AI turn presentation | Paced with a reasoning hint |
| AI algorithm | Heuristic turn-scoring (1-ply); Hard adds 2-ply lookahead on top candidates |

---

## 1. Architecture — Logical State & Pure Functions

### The Problem

The live game stores logical data (`cubies[].pos/rot`, `allStickers[].mark`) alongside Three.js mesh references. The AI must simulate hundreds of candidate turns on *copies* without animating anything. Mixing that with mesh objects would be messy and slow.

### Solution: Lightweight Snapshot

A `LogicalState` is a plain data object — no mesh references:

```js
{
  posArr: [[x,y,z], …],      // 26 cubie positions (integers)
  rotArr: [[[…]], …],        // 26 cubie 3×3 rotation matrices
  marks:  [null|'X'|'O', …]  // 54 entries, same order as allStickers[]
}
```

Five pure functions operate on it:

| Function | Signature | What it does |
|---|---|---|
| `snapshotState()` | `() → LogicalState` | Reads live `cubies` / `allStickers` → fresh snapshot |
| `cloneState(s)` | `LogicalState → LogicalState` | Deep-copies for simulation |
| `applyMoveLogical(s, axis, layer, angle)` | mutates `s` | Same integer rotation math as `finishMove`; updates posArr/rotArr |
| `detectWinnersLogical(s)` | `LogicalState → Set<'X'|'O'>` | Same algorithm as `detectWinners()`; reads from state instead of globals |
| `listEmptyStickers(s)` | `LogicalState → number[]` | Indices where `marks[i] === null` |

**Existing code is unchanged.** The live game's `detectWinners`, `finishMove`, and `resolveTurn` stay as-is. The AI uses the parallel pure functions exclusively for search, then `aiTakeTurn` executes the chosen turn by calling raw operations (mark + glyph placement, then `applyMove`) rather than the state-transitioning wrappers — see §2.3 for the pacing sequence.

---

## 2. AI Engine

### Turn Enumeration

Each candidate turn is a `(sticker_index, move)` pair. Early in the game this is up to ~650 candidates (≤ 54 empty stickers × 12 moves). For each candidate the AI:

1. Clones the snapshot.
2. Sets `marks[i] = aiMark`.
3. Calls `applyMoveLogical(clone, axis, layer, angle)`.
4. Scores the result with `scoreState()`.

### Scoring (`scoreState`)

Priority order:

1. **Immediate win** `(+10 000)` — `detectWinnersLogical` returns an AI line but no opponent line.
2. **Immediate loss** `(−10 000)` — result has an opponent line (the AI's own move completed it).
3. **Draw** `(0)` — result has both lines simultaneously.
4. **Heuristic threat count** — for each of the 6 faces' 8 lines, score the cells in the result:

   | Pattern | Score |
   |---|---|
   | 2 AI marks + 1 empty | +3 (live threat) |
   | 1 AI mark + 2 empty | +1 (seed) |
   | 2 opponent marks + 1 empty | −3 (dangerous) |
   | 1 opponent mark + 2 empty | −1 (minor) |
   | Mixed or all empty | 0 |

   Sum over all 48 lines = heuristic score.

The scoring function also returns a `reason` string used by the reasoning hint (see §2.3).

### Difficulty Policies

**Easy**
- Pick uniformly at random among non-losing turns (score > −10 000).
- Takes an obvious immediate win only 33% of the time (otherwise picks randomly, ignoring it).
- Reasoning hint: usually blank; occasionally *"Playing randomly."*

**Medium**
- Always takes an immediate win.
- Never picks an immediately losing turn when any alternative exists.
- Among remaining turns, picks the highest heuristic score with uniform random noise in the range [−5, +5] added per candidate (so it doesn't feel robotic on tied positions; the noise is small relative to the threat-score range of roughly ±144).
- Reasoning hint: reflects the actual reason for the choice.

**Hard**
- Same win/block logic as Medium.
- For the top 15 non-winning candidates by heuristic score, simulates the opponent's best heuristic reply (2-ply look-ahead: 15 × 15 = ~225 leaf evaluations — well under 10 ms).
- Picks the turn that minimises the opponent's best counter-score.
- Reasoning hint: *"Playing it safe"* when the look-ahead determined the choice; otherwise same as Medium.

### Pacing & Reasoning Hint

When `resolveTurn` determines the next player is the AI, a dedicated async function `aiTakeTurn()` manages the sequence. Crucially, state stays at `AI_THINKING` throughout — `aiTakeTurn` does **not** call `placeMark` or `doMove` (which gate on `state === 'PLACE'` / `state === 'MOVE'` and would wrongly transition state and enable buttons). Instead it calls the underlying operations directly:

1. **State → `AI_THINKING`.** HUD shows *"Computer is thinking…"* with an animated `…`. All input locked. (`~700 ms` timeout scheduled.)
2. **Mark placement.** `aiTakeTurn` directly sets `sticker.mark = aiMark` and creates + attaches the glyph mesh — the same visual steps as `placeMark`, without the state transition. The glyph appears on the cube.
3. **Reasoning hint fades in** below the turn indicator for ~1.5 s:
   - *"Going for the win!"*
   - *"Blocking a line"*
   - *"Building a threat"*
   - *"Playing it safe"* (Hard look-ahead)
   - *(blank on Easy most of the time)*
4. **Pause ~500 ms**, then cube move animates via `applyMove(axis, layer, angle, resolveTurn)` called directly (not through `doMove`; state remains `AI_THINKING`).
5. **Animation completes** → `resolveTurn()` runs as normal — the same function as for human turns.

---

## 3. Mode Selector & UI

### New Game Flow

Clicking "New Game" (or on first load) shows a modal panel instead of resetting immediately:

```
┌─────────────────────────────────┐
│  New Game                       │
│                                 │
│  [ 2 Players ]  [ vs Computer ] │
│                                 │
│  ── shown after vs Computer ──  │
│  Difficulty:  Easy  Med  Hard   │
│                                 │
│            [ Start ]            │
└─────────────────────────────────┘
```

On **Start (vs Computer)**:
- Coin flip assigns sides. Brief overlay: *"You are X — you move first"* or *"You are O — computer moves first"* shown for ~2 s, then game starts.
- If the computer is X, it goes first immediately (AI_THINKING).

On **Start (2 Players)**: existing reset behaviour, no change.

### HUD Additions (purely additive)

| Element | Behaviour |
|---|---|
| Turn indicator | Shows "Computer (X)" or "Computer (O)" on AI turns; existing logic unchanged otherwise |
| `#hint` line | Below the instruction text. Hidden by default; fades in/out for the reasoning hint. Muted colour, same font. |
| `AI_THINKING` instruction | *"Computer is thinking…"* with CSS ellipsis animation |

### State Machine

New state: **`AI_THINKING`** — inserted between the previous player's resolve and the AI's mark placement. All user input is locked; move buttons disabled.

New variables:
- `gameMode`: `'human' | 'ai'`
- `aiDifficulty`: `'easy' | 'medium' | 'hard'`
- `aiPlayer`: `1 | 2` (set at game start by coin flip)

### Existing Code Changes (minimal)

| Location | Change |
|---|---|
| `resolveTurn()` | +4 lines: if next player is AI → set `AI_THINKING`, schedule `aiTakeTurn()` |
| `newGame()` | +6 lines: show mode-selector panel instead of resetting directly |
| `updateUI()` | +10 lines: handle `AI_THINKING` state; label AI player correctly |

---

## 4. Code Size Estimate

| Category | Lines |
|---|---|
| LogicalState + pure functions | ~60 |
| `scoreState()` | ~40 |
| `aiTakeTurn()` + difficulty policies | ~80 |
| Mode-selector HTML + CSS | ~40 |
| `#hint` element + fade logic | ~15 |
| Changes to existing functions | ~20 |
| **Total added / changed** | **~255** |

The file grows from ~430 lines to ~685 lines — still a single readable script.

---

## 5. Out of Scope

- Online / networked multiplayer.
- AI vs AI demo mode.
- Undo / move history.
- Sound or haptic feedback.
- Persisted scores across sessions.

---

## 6. Verification

1. **Mode selector** — New Game shows the panel; both modes start correctly; difficulty buttons select properly.
2. **Side assignment** — over several games, human is assigned X roughly half the time.
3. **Easy AI** — occasionally makes obviously suboptimal moves; sometimes misses a win; never consistently beats a competent player.
4. **Medium AI** — always takes an immediate win; always blocks an obvious opponent win; builds recognisable threats.
5. **Hard AI** — on a position where Medium would blunder into setting up the opponent, Hard avoids it.
6. **Reasoning hint** — correct label appears at the right moment and fades away cleanly.
7. **No regressions** — 2-player mode works exactly as before; win/draw detection unchanged; New Game in 2-player mode skips difficulty and side steps.
