# Architecture

## Component Overview

<!-- High-level diagram: Players → Game Engine → Move Executor → Board State → View
     Mermaid or ASCII — your choice. -->

---

## Package Structure

<!-- Two top-level packages in the Studio version:
     - platform/  — pure game logic, zero Android dependencies
     - android/   — views, activities, Android lifecycle wrappers
     Why this split matters for testability and portability. -->

---

## Key Abstractions

### Player Hierarchy
<!-- Player interface → AbstractPlayer → HumanPlayer / MachinePlayer / SimulationPlayer
     Why SimulationPlayer exists — lookahead without polluting game state. -->

### Game State
<!-- GameState / GamePlayState / SimulationState
     What each owns and why they're separate. -->

### Move Executor
<!-- AbstractMoveExecutor — single implementation of Mancala rules
     GameMoveExecutor (display) vs MachineMoveExecutor (simulation)
     Why this duplication-avoidance matters. -->

---

## Threading Model

<!-- Game loop runs on a dedicated thread.
     Human input: wait on touch events.
     AI: blocks the game thread while searching (with animated "thinking" indicator).
     UI updates: listener callbacks → invalidate() on the View thread. -->

---

## Persistence

<!-- SharedPreferences: three stores — BANTUMI_SETTINGS, BANTUMI_STATE, BANTUMI_STATS
     What each stores and when it's written/read.
     Game state saved on onPause(), restored on resume. -->

---

## Rendering

<!-- Custom View subclass (GameBoardView) — direct Canvas drawing, no XML layouts for the board.
     Why: responsive scaling across device densities, animation control.
     MachineThinkerAnimator: spinning arc on a separate thread during AI search. -->
