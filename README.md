# Minimax Chess Engine & Kivy GUI

A desktop chess application and AI engine built from scratch in Python using Kivy. It enforces full FIDE rules, supports configurable Fischer time controls, and features a depth-4 Alpha-Beta search engine running asynchronously so the GUI remains responsive during search.

## Technical Highlights

* **Alpha-Beta Search & Move Ordering:** Implements Minimax (`depth=4`) with Alpha-Beta pruning, Most Valuable Victim – Least Valuable Attacker (MVV-LVA) capture sorting, promotion prioritisation, and depth-aged checkmate scoring (`MATE_SCORE + depth`) to favour faster mates.
* **Zero-Allocation Tree Traversal:** Mutates board state in place (`apply_move` / `undo_move`) using reversible state records—restoring castling rights, rook coordinates, and en-passant targets without `deepcopy` overhead—paired with piece-specific ray and offset move generation.
* **Differential-Equation Positional Evaluation:** Evaluates board states using material balance, minor-piece development bonuses, pawn advancement, and a central-control term ($d4, e4, d5, e5$ and the surrounding ring) modelled via exponential decay:
  $$B(n) = B_0 e^{-kn}$$
  where $n$ is the number of full moves played ($B_0 = 0.6$, $k = 0.25$), smoothly shifting engine priorities from opening centre control to endgame pawn promotion.
* **Non-Blocking Kivy Interface:** Executes engine search on a background worker thread (`threading`) and marshals board updates back onto Kivy's main UI thread via `Clock.schedule_once`, alongside live Fischer increment clocks, scrollable algebraic move history, and automated UTC match logging to CSV.

## Project Structure

```text
├── main.py              # Chess rules engine, Alpha-Beta AI, and Kivy GUI
├── images/              # Optional PNG piece sprites (e.g., white-king.png)
├── requirements.txt     # Python dependencies
├── chessgame.csv        # CSV file, saves game information
├── LICENSE              # MIT License
└── .gitignore           # Excludes virtual environments, caches, and local CSV logs
