# System Architecture & Technical Walkthrough

This document explains the internal design, search algorithms, mathematical evaluation heuristics, and concurrency model powering the **Minimax Chess Engine & Kivy GUI** (`main.py`).

---

## 1. High-Level Architecture

The codebase is decoupled into two distinct layers inside `main.py`:

1. **Core Engine (`ChessGame`):** Pure Python game state, FIDE ruleset validator, move generator, static board evaluator, and Alpha-Beta search tree. It has zero dependency on Kivy graphics primitives and communicates UI updates strictly through optional callbacks.
2. **Presentation & Event Loop (`HomeScreen`, `GameScreen`, `ChessApp`):** Built on Kivy's `ScreenManager`, handling user input, Fischer increment clocks, scrollable move history rendering, CSV match logging, and background worker thread orchestration.

```text
┌─────────────────────────────────────────────────────────────────┐
│                        Kivy Main UI Thread                      │
│  ┌────────────┐      ┌───────────────────────────────────────┐  │
│  │ HomeScreen │─────▶│              GameScreen               │  │
│  └────────────┘      │  • 8x8 Board Buttons & Piece Sprites  │  │
│                      │  • 1Hz Fischer Clock & Move History   │  │
│                      └──────────┬──────────────────▲─────────┘  │
└─────────────────────────────────┼──────────────────┼────────────┘
                                  │ spawns           │ Clock.schedule_once
                                  ▼                  │ (thread-safe UI sync)
┌─────────────────────────────────┴──────────────────┴────────────┐
│                   Background Search Worker Thread               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                        ChessGame                          │  │
│  │  • Ray/Offset Move Generator  • MVV-LVA Move Ordering     │  │
│  │  • In-Place Make/Unmake State • Depth-4 Alpha-Beta Search │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Board Representation & In-Place State Mutation

### Data Structures
* **Board Grid (`self.board`):** An $8 \times 8$ list of lists where row `0` corresponds to Rank 8 (Black's back rank) and row `7` corresponds to Rank 1 (White's back rank). Empty squares hold `None`; occupied squares hold lightweight dictionaries:
  ```python
  {'piece': 'knight', 'colour': 'white'}
  ```
* **Auxiliary State:** Castling availability (`self.castling_rights`) and en-passant vulnerability (`self.en_passant_target`) are tracked explicitly alongside `self.move_history`.

### Zero-Allocation Search (`apply_move` / `undo_move`)
Cloning an $8 \times 8$ nested dictionary via `copy.deepcopy()` at every node of a depth-5 game tree introduces heavy garbage-collection overhead. Instead, `ChessGame` mutates `self.board` in place and returns a compact `undo` snapshot:

```python
undo = {
    'move': (from_row, from_col, to_row, to_col),
    'piece': piece,
    'captured': self.board[to_row][to_col],
    'rights': {c: dict(r) for c, r in self.castling_rights.items()},
    'ep': self.en_passant_target,
    'rook_move': None,
    'ep_capture': None,
}
```

When `undo_move(undo)` is called during tree backtracking, it reverses normal captures, restores castled rooks (`rook_move`), puts back pawns removed via en-passant (`ep_capture`), and reverts castling rights and en-passant targets in $O(1)$ time.

---

## 3. Rules Enforcement & Move Generation

### Ray & Offset Candidate Generation (`get_all_legal_moves`)
Rather than scanning all 64 destination squares for every piece on the board, `get_all_legal_moves()` generates only geometrically reachable candidate squares per piece type:
* **Pawns:** Single push, double push from starting rank, and two diagonal capture targets.
* **Knights & Kings:** Precomputed coordinate offset tuples (`KNIGHT_OFFSETS`, `KING_OFFSETS`, including castling `±2` file steps).
* **Sliding Pieces (Rooks, Bishops, Queens):** Cast directional rays (`ROOK_DIRS`, `BISHOP_DIRS`), terminating immediately when encountering a friendly piece or capturing the first enemy piece along a ray.

### King Safety & Pin Validation (`would_be_in_check_after_move`)
Every candidate move is passed through `is_valid_move()`, which temporarily executes the move on `self.board`, locates the moving side's king via `find_king()`, and tests `is_square_attacked()` against all opposing pieces before reverting the board. Special edge cases handled include:
* **Castling:** Verifies the king is not currently in check, neither the king nor the target rook has moved (`CORNER_RIGHTS`), intervening squares are empty, and the king does not pass through or land on an attacked square.
* **En-Passant:** Validates diagonal capture onto `self.en_passant_target` on the immediately following half-move and tests discovered-check exposure when removing the adjacent pawn.
* **Pawn Promotion:** Prompts an `auto_dismiss=False` modal popup during human turns (`show_promotion_popup`) and automatically promotes to a queen during Minimax lookahead (`make_temporary_move`).

---

## 4. Search Algorithm (`minimax` & `score_move`)

The AI plays Black using a depth-5 Minimax search with **Alpha-Beta pruning** and **MVV-LVA move ordering**.

### MVV-LVA Move Ordering (`score_move`)
Alpha-Beta pruning achieves maximum branch cutoffs when strong moves are evaluated first. Before iterating through `legal_moves`, the engine sorts candidates descending by heuristic priority:
1. **Captures (MVV-LVA):** Scored as $10 \times V_{\text{victim}} - V_{\text{attacker}}$ (e.g., a pawn capturing a queen scores $10(9) - 1 = 89$, searched before a queen capturing a pawn at $10(1) - 9 = 1$).
2. **En-Passant Captures:** Scored at `+9`.
3. **Pawn Promotions:** Scored at `+80`.
4. **Central Control:** Moves landing inside `CENTRE_WEIGHTS` receive a `+1` tie-breaker.

### Depth-Aged Checkmate Scoring
When a node has zero legal moves:
* If the active king is in check, the engine returns $\pm(\text{MATE\_SCORE} + \text{depth})$. Adding the remaining search `depth` ensures the engine always prefers a mate-in-1 over a mate-in-3, and delays forced checkmates as long as possible when defending.
* If the active king is not in check, it returns `0` (stalemate draw).

---

## 5. Mathematical Positional Evaluation (`evaluate_board`)

Leaf nodes at `depth == 0` are scored from White's perspective (positive favours White; negative favours Black) across four components:

1. **Material Balance:** Standard piece weights (`pawn=1`, `knight=3`, `bishop=3`, `rook=5`, `queen=9`, `king=1000`).
2. **Minor-Piece Development:** `+0.2` (or `-0.2` for Black) for knights and bishops developed off their starting back rank.
3. **Pawn Advancement:** `+0.1` per rank advanced toward promotion.
4. **Exponentially Decaying Centre-Pawn Bonus:**
   Early central control ($d4, e4, d5, e5$ weighted at `1.0`, surrounding ring $c3\text{--}f6$ weighted at `0.4`) is modelled as a first-order differential decay process with respect to full moves played $n$:
   $$\frac{dB}{dn} = -kB, \quad B(0) = B_0 \implies B(n) = B_0 e^{-kn}$$
   With $B_0 = 0.6$ and $k = 0.25$ (a half-life of $\ln(2)/0.25 \approx 2.77$ full moves), the engine fights aggressively for central space in the opening before smoothly decaying that weight toward zero in the middlegame and endgame. Leaf positions are aged accurately using `full_moves = (len(self.move_history) + plies_ahead) / 2`.

---

## 6. Concurrency & Kivy UI Lifecycle

* **Non-Blocking AI Execution (`ai_turn`):** Running a depth-4 tree search directly on Kivy's main loop would block frame rendering and trigger OS unresponsive-window warnings. When Black's turn begins, `GameScreen` locks board input (`self.ai_thinking = True`) and spawns a daemon `threading.Thread` to execute `self.game.ai_move(depth=5)`. Once complete, `Clock.schedule_once` schedules `_finish_ai_turn()` back on the main thread to safely mutate Kivy UI textures and timers.
* **Texture-Safe Move History (`update_move_history`):** Rather than concatenating an entire game into a single giant `Label` (which can exceed OpenGL `GL_MAX_TEXTURE_SIZE` limits on long matches), each full move pair (`1. e4   e5`) is rendered as a fixed-height row inside a width-bound `GridLayout` within a `ScrollView`.
* **Telemetry Logging (`save_game_record`):** Upon checkmate, stalemate, timeout, or resignation, match metadata—including ISO-8601 UTC timestamp, player names, time control, Fischer increment, remaining clock times, and pipe-delimited algebraic moves—is appended to `chessgame.csv`.
