# Chess – Architecture & Developer Reference

Single-file browser chess game (`index.html`) with a companion test suite (`test.html`).
No build step, no dependencies, no server required.

---

## Data Structures

### Board (`board` / `INIT`)

```
Array<string>  length 64  — flat, row-major
```

Index formula: `idx(row, col) = row * 8 + col`
Inverse: `rc(i) = [Math.floor(i/8), i % 8]`

Row 0 is rank 8 (Black's back rank); row 7 is rank 1 (White's back rank).

```
index   0  1  2  3  4  5  6  7     ← rank 8  (row 0)
        8  9 10 11 12 13 14 15     ← rank 7  (row 1)
        ...
       56 57 58 59 60 61 62 63     ← rank 1  (row 7)
```

Each cell is either `''` (empty) or a one-character piece code:

| Code | Piece  | Colour |
|------|--------|--------|
| `K`  | King   | White  |
| `Q`  | Queen  | White  |
| `R`  | Rook   | White  |
| `B`  | Bishop | White  |
| `N`  | Knight | White  |
| `P`  | Pawn   | White  |
| `k`  | King   | Black  |
| `q`  | Queen  | Black  |
| `r`  | Rook   | Black  |
| `b`  | Bishop | Black  |
| `n`  | Knight | Black  |
| `p`  | Pawn   | Black  |

Uppercase = White; lowercase = Black. The `PIECES` map converts codes to Unicode glyphs for rendering.

---

### Castling Rights (`castling`)

```js
{
  wK: boolean,   // White may castle kingside  (King + h1 Rook unmoved)
  wQ: boolean,   // White may castle queenside (King + a1 Rook unmoved)
  bK: boolean,   // Black may castle kingside  (king + h8 rook unmoved)
  bQ: boolean,   // Black may castle queenside (king + a8 rook unmoved)
}
```

Set to `true` at game start; cleared to `false` when the relevant king or rook moves.

---

### En Passant (`enPassant`)

```
number | null
```

The board index of the en-passant target square (the empty square a pawn may capture into). Set after any pawn double-push; cleared to `null` after every other move.

---

### History Stack (`history`)

```
Array<HistoryEntry>
```

One entry per move, pushed before each move is applied. Used by Undo.

```js
HistoryEntry = {
  board:     Array<string>,  // snapshot of board array
  turn:      'w' | 'b',
  enPassant: number | null,
  castling:  { wK, wQ, bK, bQ },
  capturedW: Array<string>,  // white pieces captured so far
  capturedB: Array<string>,  // black pieces captured so far
}
```

---

### Game State Globals

| Variable     | Type                  | Description                                      |
|--------------|-----------------------|--------------------------------------------------|
| `board`      | `Array<string>`       | Current board position                           |
| `turn`       | `'w' \| 'b' \| null` | Whose turn; `null` after checkmate/stalemate     |
| `selected`   | `number \| null`      | Index of the currently selected square           |
| `legalMoves` | `Array<number>`       | Legal destination indices for the selected piece |
| `history`    | `Array<HistoryEntry>` | Undo stack                                       |
| `castling`   | `CastlingRights`      | See above                                        |
| `enPassant`  | `number \| null`      | See above                                        |
| `capturedW`  | `Array<string>`       | White pieces captured by Black                   |
| `capturedB`  | `Array<string>`       | Black pieces captured by White                   |

---

## Core Logic

### `rawMoves(b, i, t, ep, cas) → number[]`

Returns pseudo-legal destination indices for piece at index `i` playing as colour `t`.
Does **not** filter moves that leave own king in check.

- **Pawn**: forward 1 (or 2 from start row), diagonal captures, en-passant.
- **Knight**: all 8 L-shapes that land in bounds and are not friendly.
- **Bishop**: slides along 4 diagonals via `slide()`.
- **Rook**: slides along 4 cardinal directions via `slide()`.
- **Queen**: bishop + rook combined (8 directions).
- **King**: all 8 adjacent squares, plus castling appended to the candidate list
  when rights are intact and the squares between king and rook are empty.

### `slide(b, r, c, dr, dc, t) → number[]`

Extends a ray `(dr, dc)` from `(r, c)` until it hits a boundary, friendly piece
(stop before), or enemy piece (include, then stop).

### `applyMove(b, from, to, t, ep, cas) → { nb, captured, newEp, newCas }`

Returns a **new** board array and updated metadata. Never mutates inputs.

Side-effects handled:
- En-passant: removes the captured pawn from the board.
- Castling: teleports the appropriate rook alongside the king.
- Castling rights: cleared for the moving king/rook.
- En-passant flag: set after a pawn double-push, otherwise `null`.
- Promotion: pawn reaching rank 8 (White) or rank 1 (Black) becomes a queen.
  **No promotion-choice dialog — auto-promotes to queen.**

### `legalMovesFor(b, i, t, ep, cas) → number[]`

Filters `rawMoves` to only moves that:
1. Do not leave own king in check (simulated via `applyMove` + `inCheck`).
2. For castling moves: also verify the king's origin square and the transit square
   are not attacked (king cannot castle through or out of check).

### `allLegalMoves(b, t, ep, cas) → [from, to][]`

Iterates all 64 squares, collecting legal moves for every piece of colour `t`.
Used to detect checkmate (0 moves + in check) and stalemate (0 moves, not in check).

### `inCheck(b, t, ep, cas) → boolean`

Locates `t`'s king via `findKing`, then calls `isAttacked` with the opposing colour.

### `isAttacked(b, i, byColor, ep, cas) → boolean`

Checks whether any piece of `byColor` has `i` in its `rawMoves` list.
O(64 × avg-moves) — acceptable for a two-player UI game.

---

## Game Flow

```
newGame()
  └─ initialise all globals → render()

click on square → handleSquare(i)
  ├─ no selection: select piece of current turn → render()
  ├─ legal destination clicked: makeMove(from, to)
  │     ├─ applyMove()        — produce new board
  │     ├─ push HistoryEntry  — enable undo
  │     ├─ update globals
  │     ├─ flip turn
  │     └─ checkGameState()
  │           ├─ allLegalMoves() == 0 + inCheck → Checkmate, turn = null
  │           ├─ allLegalMoves() == 0            → Stalemate,  turn = null
  │           └─ inCheck                         → "in check" status
  └─ same-colour piece clicked: re-select → render()

undoMove()
  └─ pop HistoryEntry → restore all globals → render()
```

---

## Rendering

`render()` rebuilds the board DOM from scratch on every call (no virtual DOM / diff).

Square CSS classes applied per-square:

| Class           | Condition                                        |
|-----------------|--------------------------------------------------|
| `light` / `dark`| `(row + col) % 2 === 0` → light                 |
| `selected`      | `i === selected`                                 |
| `last-move`     | `i` is in the two squares that differ between the previous board snapshot and the current board |
| `in-check`      | `i` is the king square currently in check        |
| `legal-move`    | `i` is in `legalMoves` and the square is empty  |
| `legal-capture` | `i` is in `legalMoves` and the square is occupied|

Last-move detection compares `history[last].board` against `board` and highlights
all differing indices (covers castling, which moves 2 pieces).

---

## Known Gaps / Stubbed TODOs

The following features are absent and would be natural next steps:

1. **Promotion choice** — `applyMove` auto-promotes to queen. A dialog or button
   row to choose R/B/N/Q is not implemented.

2. **Draw conditions beyond stalemate** — The following draws are not detected:
   - Threefold repetition
   - Fifty-move rule
   - Insufficient material (K vs K, K+B vs K, etc.)

3. **Computer opponent / AI** — The game is strictly two-player local. No engine,
   minimax, or random-move opponent exists.

4. **Move notation / game log** — No algebraic notation (PGN/SAN) is generated or
   displayed. `history` stores full board snapshots rather than moves.

5. **Promotion choice for Black** — `applyMove` promotes Black's pawn to `'q'`
   unconditionally (correct piece, but same no-choice issue as White).

6. **Last-move highlighting robustness** — The current diff heuristic
   (`diffs[0]` and `diffs[last]`) can highlight the wrong squares for en-passant
   (three squares change) or queenside castling (four squares change).

7. **Board orientation / flip** — No button to view the board from Black's perspective.

8. **Clocks / timers** — No time controls.

9. **Import / Export** — No FEN or PGN import/export.

10. **Accessibility** — Keyboard navigation and screen-reader labels are absent.
