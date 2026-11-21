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
Cloning an $8 \times 8$ nested dictionary via `copy.deepcopy()` at every node of a depth-4 game tree introduces heavy garbage-collection overhead. Instead, `ChessGame` mutates `self.board` in place and returns a compact `undo` snapshot:

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

### King Safety & Pin Validation (`would_be_in_check_
