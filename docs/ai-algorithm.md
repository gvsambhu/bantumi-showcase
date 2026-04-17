# AI Algorithm — Minimax with Alpha-Beta Pruning

## Problem Definition

<!-- Describe the game state: board of 14 boxes (6 pits + 1 store per player), seed counts, whose turn.
     Define what a "move" is, what the move space looks like, and what makes this a good fit for game-tree search. -->

---

## Why Minimax

<!-- Perfect information, two-player, zero-sum — why these properties make minimax the right choice.
     Compare briefly: why not MCTS or RL? -->

---

## Alpha-Beta Pruning

<!-- What it prunes and why. The alpha and beta values, what they track.
     How much of the tree gets pruned in practice at depth 1-6 for a Bantumi board. -->

---

## The Heuristic Evaluation Function

<!-- What the evaluation function measures:
     - Seeds in main/store box (mainWeight)
     - Total seeds in play boxes (totalWeight)
     - Weights vary by depth and turn (EngineParamConfig — 12 configurations)
     Score formula: currentPlayerScore - opponentScore
     Why a differential score matters. -->

---

## Search Depth & Difficulty

<!-- Depth 1–6, mapped to Easy/Medium/Hard/Very Hard in the UI.
     How bonus turns are handled: they don't increment depth (why).
     Time per move at each depth on a mid-range Android device. -->

---

## Evolution of the Engine

<!-- Started with three engines: heuristic-only, pure minimax, alpha-beta.
     Why alpha-beta was kept and the others retired.
     What the initial heuristic weights looked like vs. the final tuned weights. -->

---

## What a Modern Approach Would Use

<!-- MCTS: better for larger state spaces, stochastic games. Overkill here.
     Reinforcement learning: interesting, but minimax is optimal for a perfect-information zero-sum game.
     Why minimax + alpha-beta is still the right answer for Bantumi. -->
