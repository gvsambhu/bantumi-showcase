# AI Algorithm — Minimax with Alpha-Beta Pruning

## The Problem

On its turn, the bot has to answer one question: *which of my six pits should I empty right now?*

The board is a row of fourteen positions — six pits per player plus one store each. Every state is fully visible, no hidden cards, no dice. The game is **deterministic, perfect-information, two-player, zero-sum** — every seed I capture is a seed the opponent doesn't. That shape is the textbook fit for classical game-tree search, and that's what the bot runs.

---

## Why Minimax

Minimax assumes the opponent plays optimally against you. You pick the move that maximises your worst-case outcome — the move that gives the best result even when the opponent replies with *their* best move. It's the right default for this kind of game: both sides see the whole board, one side's gain is the other's loss, and there's no randomness to model.

No coin flips, no hidden information — so no need for MCTS-style sampling. No reward signal across episodes — so no training loop, no neural net. For a game with a modest state space and full observability, minimax does more with less code than any of the alternatives.

---

## Alpha-Beta Pruning

Plain minimax explores every node in the game tree up to the chosen depth. Most of those nodes are wasted work. If I'm about to pick between move A and move B, and my analysis of A shows it's going to give me at least X, then as soon as I start analysing B and see a line where the opponent can force me below X, I stop. I already know I'll pick A. The rest of the B subtree doesn't matter.

That's alpha-beta in one paragraph. Two running bounds — α (the best I've found so far) and β (the best the opponent has found so far) — get passed down the recursion and prune away any branch that can't improve the current decision.

For Bantumi, the branching factor is low (at most six pit choices per ply, often fewer when some pits are empty), which means alpha-beta prunes aggressively. A depth-5 search that would've been prohibitive without pruning runs fast enough that the UI doesn't feel frozen — I didn't formally measure how much of the tree got pruned in practice, but the felt difference was large enough that adding pruning was the point at which the bot moved from "too slow to ship" to "shippable."

---

## The Heuristic Evaluation Function

When the search hits its depth limit without the game ending, it needs to score the current position. Bantumi uses a straightforward evaluation:

```
score = mainWeight × (seedsInMyStore − seedsInOpponentStore)
      + totalWeight × (seedsOnMySide − seedsOnOpponentSide)
```

Two terms, both differentials:

- **`mainWeight`** — weight on the seeds that have already landed in the store. These are permanent. They count at the end of the game.
- **`totalWeight`** — weight on the seeds still in play on my side versus the opponent's. These are potential: they're not locked in, but having more seeds on my side means more move options and more chances to chain bonus turns.

A positive score means I'm winning the position, negative means the opponent is. Differentials matter more than absolute counts — what you want to know isn't "how much do I have" but "how much more than the opponent."

**What I didn't explore.** Mobility (counting available legal moves), capture-potential (positions one move away from grabbing an opponent pit), endgame-specific terms — none of these are in the current evaluator. When I built this, these two terms covered enough of the positional intuition to produce a challenging bot. I'm open to revisiting.

---

## Search Depth and Difficulty

The five difficulty levels map directly to search depth:

| Level | Search depth |
|---|---|
| Beginner | 1 |
| Easy | 2 |
| Medium | 3 |
| Hard | 4 |
| Very Hard | 5 |

Depth-1 looks at only the immediate next move. Depth-5 looks five plies ahead. The search time roughly doubles with each depth increment, which is why 5 is the ceiling.

### Bonus turns don't consume depth

A key detail: when a move ends in your own store, you earn another turn. The AI explores that bonus turn **at the same depth**, not as a separate ply.

This matters because chain opportunities are the heart of competitive Mancala. A well-timed sequence of moves can land three or four bonus turns in a row. If the AI treated each bonus turn as a depth decrement, it would run out of lookahead budget inside the chain and miss the payoff sitting at the end. By not consuming depth on bonus turns, the AI evaluates a chain as a single extended move, which is how a human plays it.

The cost: a depth-5 search inside a chain-heavy position can unroll to eight or ten actual sowing events. That's why Very Hard takes up to ten seconds on older hardware and under a second on modern phones — the work scales with how much chaining the position allows, not just with nominal depth.

### Keeping the ceiling at depth 5

I could have added a Depth-6 or Depth-7 "expert" level. I didn't, for a deliberate reason: the game needs to feel responsive on the full range of phones that install it, not just on flagships. Depth-5 is the highest I could push while keeping the worst-case think-time on low-end hardware tolerable. Flagships breeze through it in a second. That's a happy accident, not a design target — the target was the slow phone.

---

## Evolution of the Engine

I didn't start with alpha-beta. I got here through iteration.

**v1 — plain heuristic.** The earliest engine just scored every legal move by applying the evaluation function one ply deep and picked the best. No lookahead, no opponent modelling. It played, but it played badly.

**v2 — minimax without pruning.** Added recursion: score my move assuming the opponent replies with their best response, assuming I reply with mine, and so on. Significantly stronger. Significantly slower.

**v3 — alpha-beta.** Same algorithm as minimax, with the pruning logic layered in. Same move choices, a fraction of the nodes visited.

Once alpha-beta was working, the first two engines were dead code. Plain minimax produces identical results to alpha-beta — alpha-beta is strictly an optimisation, not a different algorithm. The pure-heuristic engine was a stepping stone whose only purpose had been to bootstrap the evaluation function.

Consolidating to a single engine was the right call. Three implementations to maintain, test, and tune would've been three times the surface area for one working bot.

---

## Tuning the Weights

Picking numbers for `mainWeight` and `totalWeight` by intuition was a dead end. Too low on `mainWeight` and the bot hoarded seeds on the board instead of securing them. Too high and it rushed to the store and missed capture opportunities. The right answer had to come from measurement.

The platform/android split (see [architecture](architecture.md#package-structure)) existed so I could run the engine headlessly — thousands of bot-vs-bot games per weight configuration, no UI, no emulator, just a plain Java main method running tournaments.

### The configuration grid

The weights aren't universal — they're tuned per **(difficulty level, turn)** combination:

- **Difficulty level.** Beginner through Very Hard. Each level maps to its own search depth (1 through 5), and at each depth the evaluation function carries a different share of the decision — shallow depths lean on the weights, deeper depths let the search smooth over weight imperfections. So optimal weights shift with level.
- **Turn — who opens the game.** Whether the bot is the first or second player changes the positional calculus. The opener has a tempo advantage; the responder reacts. The best weights for one aren't necessarily the best for the other.

That's a grid of (level × turn) cells. Each cell got its own tuning pass. The answer the experiment produced isn't a single `(mainWeight, totalWeight)` pair — it's a table indexed by *what difficulty am I* and *am I opening or responding*.

A note on the turn dimension: the app currently doesn't let the user pick who opens — Player 1 always starts, and that's the bot in Computer Mode. Supporting a "you open" option is a planned upgrade. The tuning already accounts for both cases, so the data is ready when the UI catches up.

### The experiment

For each cell in the grid, candidate weight pairs were played against each other bot-vs-bot, 100 to 1000 games per pairing, alternating sides to neutralise any opening advantage. The winning configurations at each cell were kept.

### What I found

Not a single clear winner at any cell — a **plateau**. Within each cell, a band of weight combinations performed comparably well, and differences between them sat below the noise floor of a 1000-game sample. Which is its own kind of answer: the evaluator is robust to small perturbations. Pick any point inside the plateau and the bot plays competently.

The weights shipped in the app sit inside that plateau for each (level, turn) cell. The difficulty the user picks selects the level; the match-state selects the turn; the table lookup picks the tuned weights.

---

## What a Modern Approach Might Add

Classical minimax-with-alpha-beta is a well-fitted tool for Bantumi. But a decade of AI progress has happened since this engine was written, and a few directions are worth keeping open:

- **MCTS (Monte Carlo Tree Search).** Stronger than minimax when the evaluation function is hard to hand-craft. For Mancala the heuristic is simple enough that minimax holds its own, but MCTS would likely produce slightly subtler positional play and scale better at higher depths.
- **Richer heuristic terms.** Mobility, capture-threat, endgame pattern recognition, seed-parity on each side. Any of these could be added to the existing evaluator without replacing the engine. This is the lowest-cost upgrade path and the one most likely to happen first.
- **Self-play reinforcement learning.** Train an evaluator via a neural network playing itself, AlphaZero-style. Overkill for a free Android board game, but theoretically interesting — Mancala variants are small enough that a modest network could learn near-optimal play in hours, not weeks.

The game as it ships today is complete in itself. The bot at Very Hard is already stronger than most casual players want to face, the rules are correct, and the experience doesn't depend on any of the above to be good. These are directions I'm open to growing into, not gaps to fill.
