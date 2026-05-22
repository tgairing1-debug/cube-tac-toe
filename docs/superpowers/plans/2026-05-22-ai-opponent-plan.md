# AI Opponent — Implementation Plan

**Date:** 2026-05-22  
**Spec:** `docs/superpowers/specs/2026-05-22-ai-opponent-design.md`  
**File to modify:** `index.html` (single script, grows ~430 → ~685 lines)

---

## Overview

Eight sequential phases. Each phase is self-contained and testable before moving to the next. Phases 1–5 are pure logic (no visible UI change). Phases 6–8 wire in the UI.

| Phase | What | Lines added |
|---|---|---|
| 1 | Pure rules core (LogicalState + 5 functions) | ~60 |
| 2 | `scoreState` — heuristic scoring | ~40 |
| 3 | `chooseTurn` — difficulty policies | ~55 |
| 4 | `aiTakeTurn` — async pacing sequence | ~35 |
| 5 | State machine integration | ~20 |
| 6 | Mode selector UI (HTML + CSS + JS) | ~80 |
| 7 | Reasoning hint UI | ~20 |
| 8 | First-move AI trigger + final wiring | ~10 |

---

## Phase 1 — Pure Rules Core

**Where:** New section in the `<script>` block, placed directly after the existing `/* Integer 3x3 matrix maths */` section (before the Three.js scene setup). Label it `/* ============================================================\n   Rules core — pure logical state (used by AI simulation)\n   ============================================================ */`.

### 1a. Build a `stickersInfo` lookup table (once, at startup)

After `allStickers` is fully populated (after the cube-build loop), create:

```js
// Built once; indices match allStickers[]
const stickersInfo = allStickers.map((s, i) => ({
  cubieIndex: cubies.indexOf(s.cubie),
  localNormal: s.localNormal,   // reference OK — never mutated
}));
```

This lets the pure functions work by index rather than by object reference.

### 1b. `snapshotState()`

```js
function snapshotState() {
  return {
    posArr: cubies.map(c => c.pos.slice()),
    rotArr:  cubies.map(c => c.rot.map(row => row.slice())),
    marks:   allStickers.map(s => s.mark),
  };
}
```

### 1c. `cloneState(s)`

```js
function cloneState(s) {
  return {
    posArr: s.posArr.map(p => p.slice()),
    rotArr:  s.rotArr.map(r => r.map(row => row.slice())),
    marks:   s.marks.slice(),
  };
}
```

### 1d. `applyMoveLogical(s, axis, layer, angle)`

Mirrors the integer-model update in `finishMove`. Does **not** touch meshes.

```js
function applyMoveLogical(s, axis, layer, angle) {
  const ai = AXIS_INDEX[axis];
  const M  = rotMatrix(axis, angle > 0 ? 1 : -1);
  for (let i = 0; i < cubies.length; i++) {
    if (s.posArr[i][ai] !== layer) continue;
    s.posArr[i] = matVec(M, s.posArr[i]);
    s.rotArr[i]  = matMul(M, s.rotArr[i]);
  }
}
```

### 1e. `detectWinnersLogical(s)`

Identical algorithm to `detectWinners()`, but reads `s.posArr`, `s.rotArr`, and `s.marks` instead of globals.

```js
function detectWinnersLogical(s) {
  const winners = new Set();
  for (const d of FACES) {
    const ai = d[0] !== 0 ? 0 : d[1] !== 0 ? 1 : 2;
    const [u, v] = ai === 0 ? [1,2] : ai === 1 ? [0,2] : [0,1];
    const grid = [[null,null,null],[null,null,null],[null,null,null]];
    for (let i = 0; i < stickersInfo.length; i++) {
      const ci  = stickersInfo[i].cubieIndex;
      const ln  = stickersInfo[i].localNormal;
      const wn  = matVec(s.rotArr[ci], ln);
      if (wn[0]===d[0] && wn[1]===d[1] && wn[2]===d[2]) {
        grid[s.posArr[ci][v]+1][s.posArr[ci][u]+1] = s.marks[i];
      }
    }
    for (const [a,b,c] of LINES) {
      const m = grid[a[0]][a[1]];
      if (m && grid[b[0]][b[1]]===m && grid[c[0]][c[1]]===m) winners.add(m);
    }
  }
  return winners;
}
```

### 1f. `listEmptyStickers(s)`

```js
function listEmptyStickers(s) {
  return s.marks.reduce((out, m, i) => { if (!m) out.push(i); return out; }, []);
}
```

**Verify phase 1:** After building the cube, call `snapshotState()`, mutate a clone, confirm the live state is unchanged. Run `detectWinnersLogical` on a solved snap → expect empty set. Run on a snap with three marks manually set → expect correct winner.

---

## Phase 2 — Score Function

**Where:** Immediately after the Rules Core section. Label `/* AI scoring */`.

```js
// Returns { score, reason }
// score: +10000 = AI wins, -10000 = AI loses, else heuristic (-144..+144)
function scoreState(s, aiMark, oppMark) {
  const winners = detectWinnersLogical(s);
  const aiWins  = winners.has(aiMark);
  const oppWins = winners.has(oppMark);

  if (aiWins && !oppWins) return { score: 10000, reason: 'Going for the win!' };
  if (oppWins && !aiWins) return { score: -10000, reason: '' };
  if (aiWins  &&  oppWins) return { score: 0, reason: '' };    // draw

  // Heuristic: sum over all 6 faces × 8 lines
  let score = 0;
  for (const d of FACES) {
    const ai = d[0] !== 0 ? 0 : d[1] !== 0 ? 1 : 2;
    const [u, v] = ai === 0 ? [1,2] : ai === 1 ? [0,2] : [0,1];
    const grid = [[null,null,null],[null,null,null],[null,null,null]];
    for (let i = 0; i < stickersInfo.length; i++) {
      const ci = stickersInfo[i].cubieIndex;
      const ln = stickersInfo[i].localNormal;
      const wn = matVec(s.rotArr[ci], ln);
      if (wn[0]===d[0] && wn[1]===d[1] && wn[2]===d[2])
        grid[s.posArr[ci][v]+1][s.posArr[ci][u]+1] = s.marks[i];
    }
    for (const [a,b,c] of LINES) {
      const cells = [grid[a[0]][a[1]], grid[b[0]][b[1]], grid[c[0]][c[1]]];
      const ai2  = cells.filter(x => x === aiMark).length;
      const opp2 = cells.filter(x => x === oppMark).length;
      if (ai2 > 0 && opp2 > 0) continue;   // dead line
      if (ai2 === 2 && opp2 === 0)  score += 3;
      if (ai2 === 1 && opp2 === 0)  score += 1;
      if (opp2 === 2 && ai2 === 0)  score -= 3;
      if (opp2 === 1 && ai2 === 0)  score -= 1;
    }
  }

  // Determine reason from dominant signal
  let reason = 'Building a threat';
  if (score < 0) reason = 'Blocking a line';

  return { score, reason };
}
```

**Verify phase 2:** Unit-test on a snapshot with a 2-in-a-row → score 3. With opponent 2-in-a-row → score −3. With an immediate win → 10000.

---

## Phase 3 — Difficulty Policies (`chooseTurn`)

**Where:** After `scoreState`. Label `/* AI difficulty */`.

```js
// Returns { stickerIndex, move: [axis,layer,angle], reason }
function chooseTurn(snap, aiMark, oppMark, difficulty) {
  const emptyIdx = listEmptyStickers(snap);
  const candidates = [];   // { stickerIndex, move, score, reason }

  for (const si of emptyIdx) {
    for (const [, axis, layer, dir] of MOVES) {
      const angle = dir * HALF_PI;
      const clone = cloneState(snap);
      clone.marks[si] = aiMark;
      applyMoveLogical(clone, axis, layer, angle);
      const { score, reason } = scoreState(clone, aiMark, oppMark);
      candidates.push({ stickerIndex: si, move: [axis, layer, angle], score, reason });
    }
  }

  // Separate into buckets
  const winning = candidates.filter(c => c.score ===  10000);
  const losing  = candidates.filter(c => c.score === -10000);
  const neutral = candidates.filter(c => c.score >  -10000 && c.score < 10000);
  const safe    = neutral.length ? neutral : losing;  // if all lose, pick least bad

  if (difficulty === 'easy') {
    // Avoid instant loss; take win only 33 % of the time
    const pool = neutral.length ? neutral : losing;
    if (winning.length && Math.random() < 0.33)
      return { ...winning[Math.floor(Math.random() * winning.length)], reason: 'Going for the win!' };
    const pick = pool[Math.floor(Math.random() * pool.length)];
    return { ...pick, reason: '' };
  }

  if (difficulty === 'medium') {
    if (winning.length) return { ...winning[0], reason: 'Going for the win!' };
    // Add noise then sort
    const scored = safe.map(c => ({ ...c, noisyScore: c.score + (Math.random()*10 - 5) }));
    scored.sort((a, b) => b.noisyScore - a.noisyScore);
    return scored[0];
  }

  // hard: 2-ply lookahead on top 15 neutral candidates
  if (winning.length) return { ...winning[0], reason: 'Going for the win!' };
  const top15 = [...safe].sort((a,b) => b.score - a.score).slice(0, 15);
  let bestScore = -Infinity, bestCand = top15[0];
  for (const cand of top15) {
    // Simulate opponent's best reply
    const afterAI = cloneState(snap);
    afterAI.marks[cand.stickerIndex] = aiMark;
    applyMoveLogical(afterAI, cand.move[0], cand.move[1], cand.move[2]);
    const oppEmpty = listEmptyStickers(afterAI);
    let worstOpp = Infinity;
    for (const osi of oppEmpty.slice(0, 15)) {   // sample top 15 opp stickers
      for (const [, axis, layer, dir] of MOVES) {
        const angle = dir * HALF_PI;
        const c2 = cloneState(afterAI);
        c2.marks[osi] = oppMark;
        applyMoveLogical(c2, axis, layer, angle);
        const { score } = scoreState(c2, oppMark, aiMark);   // from opp POV
        if (score < worstOpp) worstOpp = score;
      }
    }
    // We want the cand that minimises opp's best reply
    if (-worstOpp > bestScore) { bestScore = -worstOpp; bestCand = cand; }
  }
  return { ...bestCand, reason: bestCand.reason === 'Blocking a line' ? 'Blocking a line' : 'Playing it safe' };
}
```

**Verify phase 3:** Position where an immediate win exists → all levels find it (Easy 33 % pass). Position where a turn hands the opponent a win → Medium/Hard avoid it. Hard on a position where Medium would set up the opponent → Hard picks differently.

---

## Phase 4 — `aiTakeTurn()` Async Sequence

**Where:** After `chooseTurn`. Label `/* AI turn execution */`.

```js
async function aiTakeTurn() {
  const aiMark  = aiPlayer === 1 ? 'X' : 'O';
  const oppMark = aiPlayer === 1 ? 'O' : 'X';

  // 1. Compute turn (synchronous, fast)
  const snap = snapshotState();
  const { stickerIndex, move: [axis, layer, angle], reason } = chooseTurn(snap, aiMark, oppMark, aiDifficulty);

  // 2. Wait: "thinking" pause
  await new Promise(r => setTimeout(r, 700));

  // 3. Place mark directly (no state transition — state stays AI_THINKING)
  const sticker = allStickers[stickerIndex];
  sticker.mark = aiMark;
  const glyph = makeGlyph(aiMark);
  sticker.mesh.add(glyph);
  sticker.glyph = glyph;

  // 4. Reasoning hint
  showHint(reason);

  // 5. Wait: let player see the mark before the cube turns
  await new Promise(r => setTimeout(r, 500));

  // 6. Cube move — calls resolveTurn on completion
  applyMove(axis, layer, angle, resolveTurn);
}
```

**Note:** `aiTakeTurn` is `async` only so we can use `await`. The `setTimeout` delays are the only async; the actual AI computation in `chooseTurn` is synchronous (< 10 ms even for Hard).

---

## Phase 5 — State Machine Integration

**Where:** Modifications to existing functions.

### 5a. New game-state variables (add near existing `let state`, `let currentPlayer`)

```js
let gameMode    = 'human';   // 'human' | 'ai'
let aiDifficulty = 'medium'; // 'easy' | 'medium' | 'hard'
let aiPlayer    = 2;         // 1 or 2 — set at game start
```

### 5b. Modify `resolveTurn()`

After the existing `currentPlayer = currentPlayer === 1 ? 2 : 1;` swap and before `updateUI()`, add:

```js
// If next player is AI, hand off to aiTakeTurn instead of waiting for input
if (!result && gameMode === 'ai' && currentPlayer === aiPlayer) {
  state = 'AI_THINKING';
  updateUI();
  aiTakeTurn();
  return;
}
```

### 5c. Modify `updateUI()`

In the state-to-instruction mapping, add a case for `AI_THINKING`:

```js
else if (state === 'AI_THINKING')
  instrEl.textContent = 'Computer is thinking…';
```

In the turn indicator block, check whether the current player is the AI:

```js
const isAI = gameMode === 'ai' && currentPlayer === aiPlayer;
const label = isAI ? 'Computer' : `Player ${currentPlayer}`;
turnEl.innerHTML = `${label} <span class="${cls}">(${mark})</span>`;
```

In the button-disable logic, also disable during `AI_THINKING`:

```js
const enabled = (state === 'MOVE') && !(gameMode === 'ai' && currentPlayer === aiPlayer);
```

Wait — during AI's MOVE phase the state is AI_THINKING throughout, so `state === 'MOVE'` is already false. The existing check `state === 'MOVE'` is sufficient; no change needed there.

**Corrected:** Buttons disabled whenever state is not `'MOVE'`. Since AI_THINKING !== 'MOVE', buttons are already disabled during the AI turn. No additional change to the button-disable line.

---

## Phase 6 — Mode Selector UI

### 6a. HTML additions (inside `<body>`, before the existing `#banner` div)

```html
<div id="modeSelector">
  <div id="modePanel">
    <div id="modeTitle">New Game</div>
    <div id="modeButtons">
      <button id="btn2p">2 Players</button>
      <button id="btnAI">vs Computer</button>
    </div>
    <div id="diffRow" class="hidden">
      <span>Difficulty:</span>
      <button class="dbtn" data-d="easy">Easy</button>
      <button class="dbtn active" data-d="medium">Medium</button>
      <button class="dbtn" data-d="hard">Hard</button>
    </div>
    <button id="btnStart" class="hidden">Start</button>
  </div>
</div>
```

### 6b. CSS additions

```css
#modeSelector { position:fixed; inset:0; z-index:25; display:flex; align-items:center;
  justify-content:center; background:rgba(7,8,11,.82);
  -webkit-backdrop-filter:blur(4px); backdrop-filter:blur(4px); }
#modePanel { background:#1a1d24; border-radius:16px; padding:28px 32px;
  display:flex; flex-direction:column; gap:16px; align-items:center; min-width:280px; }
#modeTitle { font-size:20px; font-weight:800; }
#modeButtons { display:flex; gap:10px; }
#modeButtons button, #btnStart { padding:10px 20px; border:0; border-radius:9px;
  font-size:15px; font-weight:700; cursor:pointer; background:#3a3f4b; color:#fff; }
#modeButtons button:hover, #btnStart:hover { background:#4c93ff; }
#btnStart { background:#4c93ff; margin-top:4px; }
#diffRow { display:flex; align-items:center; gap:8px; font-size:14px; }
.dbtn { padding:6px 13px; border:0; border-radius:7px; background:#2d3038;
  color:#aaa; font-size:13px; font-weight:700; cursor:pointer; }
.dbtn.active { background:#4c93ff; color:#fff; }
```

### 6c. JavaScript for the mode selector

```js
// Show selector; onStart(mode, difficulty) called when Start is pressed
function showModeSelector() {
  document.getElementById('modeSelector').classList.remove('hidden');
}
function hideModeSelector() {
  document.getElementById('modeSelector').classList.add('hidden');
}
```

Wire up buttons (added near the existing New Game button listener):

```js
document.getElementById('btn2p').addEventListener('click', () => {
  hideModeSelector();
  startGame('human', null);
});

document.getElementById('btnAI').addEventListener('click', () => {
  document.getElementById('diffRow').classList.remove('hidden');
  document.getElementById('btnStart').classList.remove('hidden');
});

document.querySelectorAll('.dbtn').forEach(b => {
  b.addEventListener('click', () => {
    document.querySelectorAll('.dbtn').forEach(x => x.classList.remove('active'));
    b.classList.add('active');
  });
});

document.getElementById('btnStart').addEventListener('click', () => {
  const d = document.querySelector('.dbtn.active').dataset.d;
  hideModeSelector();
  startGame('ai', d);
});
```

### 6d. Refactor `newGame()` into two functions

**`startGame(mode, difficulty)`** — the actual reset (extracted from `newGame`):

```js
function startGame(mode, difficulty) {
  // Reset all sticker marks and glyphs (existing newGame logic)
  for (const s of allStickers) { … }
  for (const c of cubies) { … }
  hovered = null; animating = false; activeAnim = null;

  // Set mode variables
  gameMode     = mode || 'human';
  aiDifficulty = difficulty || 'medium';
  aiPlayer     = Math.random() < 0.5 ? 1 : 2;   // coin flip (ignored if human vs human)

  currentPlayer = 1;
  state = 'PLACE';
  bannerEl.classList.add('hidden');

  // If AI goes first (is player 1), show brief side announcement then trigger AI
  if (gameMode === 'ai') {
    const humanSide = aiPlayer === 1 ? 'O' : 'X';
    const humanFirst = aiPlayer !== 1;
    showHint(`You are ${humanSide} — you ${humanFirst ? 'move first' : 'move second'}`);
  }

  updateUI();

  // If AI is player 1, kick off its turn after a 2-second side-announcement delay
  if (gameMode === 'ai' && aiPlayer === 1) {
    state = 'AI_THINKING';
    updateUI();
    setTimeout(aiTakeTurn, 2000);
  }
}
```

**`newGame()`** (now just shows the selector):

```js
function newGame() {
  // Reset the mode panel to its default state first
  document.getElementById('diffRow').classList.add('hidden');
  document.getElementById('btnStart').classList.add('hidden');
  showModeSelector();
}
```

Update all existing references to `newGame()` in button listeners to keep pointing at `newGame()` — they already call `newGame()` so no change needed there.

**On first load:** Call `newGame()` after `updateUI()` and `tick()` so the selector appears immediately.

---

## Phase 7 — Reasoning Hint UI

### 7a. HTML (add inside `#hud`, after `#instruction`)

```html
<div id="hint"></div>
```

### 7b. CSS

```css
#hint { margin-top:4px; font-size:13px; color:#8899bb; min-height:18px;
  transition: opacity 0.25s; opacity:0; pointer-events:none; }
#hint.visible { opacity:1; }
```

### 7c. JavaScript

```js
let _hintTimer = null;
function showHint(text) {
  const el = document.getElementById('hint');
  if (!text) { el.classList.remove('visible'); return; }
  el.textContent = text;
  el.classList.add('visible');
  clearTimeout(_hintTimer);
  _hintTimer = setTimeout(() => el.classList.remove('visible'), 1800);
}
```

Call `showHint('')` in `newGame()` / `startGame()` to clear any leftover hint.

---

## Phase 8 — First Load & Final Wiring

1. **Remove** the existing `newGame` call at the bottom of the script (if any resets happen at load — currently `updateUI()` and `tick()` are the last two calls).
2. **Add** `newGame()` call after `tick()` so the mode selector appears on first load.
3. **Remove** the standalone `newGame` / `bannerNew` button listeners if they now call `newGame()` (verify they still do).
4. **Confirm** the `stickersInfo` array is built *after* the cube-build loop (after `cubies` and `allStickers` are fully populated).
5. **Smoke test** the complete flow end-to-end (see Verification below).

---

## Verification Checklist

### Mode selector
- [ ] First load: selector appears immediately.
- [ ] "2 Players" → direct start, no difficulty row shown.
- [ ] "vs Computer" → difficulty row and Start button appear.
- [ ] Difficulty buttons toggle correctly; selection persists to Start.

### Side assignment
- [ ] Over several AI games, human is assigned X roughly half the time.
- [ ] When human is O, AI starts and the cube immediately starts "thinking."
- [ ] Brief side-announcement hint appears correctly.

### Easy AI
- [ ] Occasionally misses an obvious win.
- [ ] Never consistently beats a competent player.
- [ ] Never crashes (empty-candidates edge case handled — if all turns lose, picks from losing set).

### Medium AI
- [ ] Always takes an immediate win when one exists.
- [ ] Never walks into a turn that immediately completes the opponent's line, when an alternative exists.
- [ ] Reasoning hint "Going for the win!" / "Blocking a line" / "Building a threat" appears at the right moment.

### Hard AI
- [ ] On a position where Medium's top pick would set the opponent up, Hard picks differently.
- [ ] Runs in noticeably under 1 s (check browser performance tab if in doubt).

### Turn pacing
- [ ] "Computer is thinking…" appears; all buttons + canvas input disabled.
- [ ] Mark visually appears after ~700 ms.
- [ ] Hint fades in ~1.5 s after appearing.
- [ ] Cube move animates after ~500 ms more.
- [ ] After move: if human turn → normal PLACE state; if AI's turn again (shouldn't happen mid-game unless a bug) → check.

### No regressions
- [ ] 2-player mode works exactly as before (win, draw, new game).
- [ ] Win/draw detection unchanged; tested with the same scenarios as original verification.
- [ ] "New Game" during an active AI game resets cleanly (no orphaned timeouts animating a dead game).

### Orphaned timeout guard
In `startGame()`, cancel any pending AI timeout before resetting:

```js
clearTimeout(_aiTimer);   // _aiTimer = the handle from setTimeout(aiTakeTurn, …)
```

Store the timer handle in a module-level `let _aiTimer = null` and assign it wherever `setTimeout(aiTakeTurn, …)` is called.
