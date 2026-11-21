# Project Evolution & Engineering Improvements (`IMPROVEMENTS.md`)

## Background: From A-Level Computer Science NEA to Production Engine

This project originated as my **A-Level Computer Science Non-Exam Assessment (NEA)**—a standalone desktop chess application built in Python using Kivy. While the NEA version successfully satisfied coursework requirements for full FIDE rule enforcement, user input validation, Fischer clocks, and CSV match logging, post-NEA engineering focused on transforming the codebase from an academic prototype into an efficient, portable, and responsive chess engine.

Key areas of refinement include:
* Eliminating hardcoded local file paths and adding graceful UI fallbacks for cross-platform portability.
* Replacing brute-force search and naive board evaluation with **Alpha-Beta pruning**, **MVV-LVA move ordering**, and a **differential-equation positional heuristic**.
* Decoupling the AI search onto a background thread (`threading` + `Clock.schedule_once`) so the Kivy event loop never freezes during deep lookahead.
* Fixing subtle edge-case rule bugs (pawn double-step jumping, castling out of check, modal popup dismissal, and OpenGL texture limits on long move histories).

---

## Iteration Roadmap & Version Breakdown

### Phase 1: UI Skeleton & Input Validation (`chess1.py`)
* **What was built:** Established the Kivy `ScreenManager` architecture (`HomeScreen` and `GameScreen`), an $8 \times 8$ `GridLayout` board of buttons, player setup forms, regex username validation, and placeholder timer/control layouts.
* **Limitations:** No underlying game state or rules engine existed yet (`initialise_game` only updated player labels), and board squares were not bound to click handlers.

### Phase 2: Core Rules Engine & Basic Game Loop (`chess3.py`)
* **What was built:** Introduced the `ChessGame` model class with an $8 \times 8$ board representation, piece-specific movement rules (pawns, rooks, knights, bishops, queens, kings), sliding-piece path-clearance checks, king-safety verification (`would_be_in_check_after_move`), checkmate/stalemate/insufficient-material detection, algebraic move notation, and a 1-second Kivy `Clock` countdown timer.
* **Limitations:**
  * Lacked castling, en-passant, and interactive pawn promotion (pawns auto-promoted to queens).
  * Hardcoded an absolute macOS path (`/Users/david/vscode/157069077/school_chess/images/...`) with no fallback text if images were missing.

### Phase 3: Special FIDE Rules & Interactive Promotion (`chess4.py` & `chessgame.py`)
* **What was built:**
  * Added **kingside and queenside castling** with `castling_rights` tracking for kings and corner rooks.
  * Added **en-passant** target tracking (`self.en_passant_target`) and diagonal capture removal.
  * Replaced automatic queen promotion with an interactive Kivy `Popup` (`show_promotion_popup`) allowing players to choose between Queen, Rook, Bishop, and Knight.
* **Limitations:** `GameScreen` did not pass a self-reference into `ChessGame()`, causing `self.game_screen` callbacks during promotion to fail unless wired externally.

### Phase 4: Match Telemetry & Code Documentation (`chess5.py` & `chess5commented.py`)
* **What was built:**
  * Wired bidirectional communication (`ChessGame(game_screen=self)`) so promotion popups properly refreshed the board and checked for checkmate immediately upon piece selection.
  * Implemented persistent match logging (`save_game_record`) to `chessgame.csv`, recording UTC ISO-8601 timestamps, player names, time controls, increments, match outcomes, remaining clock times, and pipe-delimited move histories, guarded by `self.saved_game_record` to prevent duplicate writes.
  * Produced comprehensive docstring and inline documentation (`chess5commented.py`).
* **Limitations:** The **"Play vs AI"** button on `HomeScreen` was functional in the UI, but no AI opponent existed yet—playing against AI still required manual moves for Black.

### Phase 5: First Minimax AI & Alpha-Beta Search (`chess6.py`)
* **What was built:**
  * Implemented the first autonomous Black AI opponent using **Minimax with Alpha-Beta pruning** (`depth=3`), `get_all_legal_moves()`, `make_temporary_move()`, and `undo_temporary_move()`.
  * Built an initial static evaluation function (`evaluate_board`) combining material weights, knight/bishop development bonuses, pawn advancement, and early-game rook discipline.
  * Added defensive bounds checks (`rook is not None`) to prevent `TypeError` crashes during castling validation when corner rooks had been captured.
* **Limitations:**
  * Calling `self.game.ai_move()` synchronously inside `on_square_press` froze the Kivy UI thread until the search completed.
  * Unordered move generation limited Alpha-Beta cutoff efficiency.
  * `make_temporary_move` did not handle castling rooks, en-passant captures, or promotions inside the search tree.

### Phase 6: Search Optimisation, Differential Evaluation & UI Hardening (`chess7_2.py`)
* **What was built:**
  * **Full-Fidelity In-Place Search (`apply_move` / `undo_move`):** Replaced partial temporary moves with a complete reversible state mutator that accurately simulates castling, en-passant, promotions, and castling-right revocations throughout the search tree, enabling deeper `depth=5` search.
  * **MVV-LVA Move Ordering (`score_move`):** Sorted candidate moves by Most Valuable Victim – Least Valuable Attacker, pawn promotions (`+80`), and central-square control before recursion, dramatically increasing Alpha-Beta pruning cutoffs.
  * **Continuous Exponential-Decay Positional Heuristic:** Modelled central-pawn control ($d4, e4, d5, e5$ and the $c3\text{--}f6$ ring) via the differential equation $\frac{dB}{dn} = -kB \implies B(n) = B_0 e^{-kn}$ ($B_0 = 0.6, k = 0.25$), smoothly shifting AI priorities from opening centre control to endgame pawn promotion, paired with depth-aged checkmate scoring (`MATE_SCORE + depth`).
  * **Rule Bug Fixes:** Fixed pawn double-step jumping over occupied squares, prevented castling out of check, and added discovered-check validation for en-passant captures.
  * **UI & Portability Upgrades:**
    * Replaced hardcoded user paths with relative `BASE_DIR` resolution and added algebraic fallback letters (`P, N, B, R, Q, K`) when sprite PNGs are absent.
    * Replaced the monolithic `move_history` `Label` with a `ScrollView` + per-move `GridLayout` to avoid exceeding OpenGL `GL_MAX_TEXTURE_SIZE` limits on long games.
    * Locked `auto_dismiss=False` on promotion popups and added `GameScreen.show_error()`.

### Phase 7: Multi-Threaded Concurrency & Ray-Based Move Generation (`main.py`)
* **What was built:**
  * **True Background Threading:** Moved AI search execution off Kivy's main UI loop onto a background worker thread (`threading.Thread`), marshalling board updates back to the main thread via `Clock.schedule_once` so timers and window events stay 100% responsive during search.
  * **Targeted Ray & Offset Move Generation:** Replaced $64 \times 64$ brute-force board scanning in `get_all_legal_moves()` with piece-specific directional rays (`ROOK_DIRS`, `BISHOP_DIRS`) and precomputed jump offsets (`KNIGHT_OFFSETS`, `KING_OFFSETS`), cutting candidate move generation overhead significantly.
