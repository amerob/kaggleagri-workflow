[← 13 Reproducing](13-reproduce.md) | **14 — Glossary** | [Index →](README.md)

---

# 14 — Glossary

Every term used in this documentation set, defined.

---

## 🏆 Competition terms

| Term | Definition |
|---|---|
| **Simulation competition** | A Kaggle format where you submit a *program* that plays a game against other players' programs, rather than predictions to be scored against ground truth. No training data, no metric to fit |
| **Agent / bot** | Your submitted program. Receives the game state each turn and returns actions |
| **Episode** | One complete 720-turn game between two agents |
| **Validation episode** | An automatic self-play game run when you upload, to check your agent doesn't crash. Failure marks the submission `Error` |
| **Skill rating** | Elo-like score. **Starts at 600** for every new submission and moves with wins and losses |
| **Bradley-Terry** | The statistical model used for the final leaderboard after the deadline. Estimates relative strength from head-to-head results |
| **Ladder** | The pool of active agents. Yours is matched against opponents of similar rating |
| **Tracked submissions** | Only your **latest 2** keep playing and count for final evaluation. Older ones stop |
| **Convergence** | The process of a rating settling near true strength. Early games move it in large steps; later games in small ones |

---

## 🎮 Game terms

| Term | Definition |
|---|---|
| **Season** | One full game: 30 days × 24 turns = **720 turns** |
| **Turn / step** | One tick. Every worker acts once |
| **Day** | 24 turns. Growth, watering resets, and weed spawns happen at day boundaries |
| **Quadrant** | One 5×5 section of the 10×10 board. You start owning one; the rest cost $1,000 / $2,000 / $4,000 |
| **Tile** | One board cell. Holds a plant, weed, coop, pasture, or nothing |
| **The shed** | Central storage, **100-item cap**, overflow destroyed at end of day. Not a tile — you interact by standing on one of the four centre tiles |
| **Farmer** | Your permanent worker. 720 actions per season |
| **Farm hand** | A worker hired for one day only. Vanishes at end of day |
| **Inventory** | What a worker is carrying. Distinct from the shed. **`FEED` and `FERTILIZE` draw from inventory, not the shed** |
| **Coop / pasture** | Structures required before placing animals. Coop for geese, pasture for cows and sheep |
| **Weed** | What a plant becomes after two unwatered days, or spawns randomly on empty tiles (0.5%/day). Cleared with `DIG` |
| **One-time crop** | Wheat, carrot, melon — harvested once, then removed |
| **Ongoing crop** | Tomato, strawberry — produce on a schedule, but capped at 4 yields then decay |
| **Bonus window** | The period during which watering adds extra yield to a one-time crop. Starts at `ceil(max_yield_day / 2)` |
| **`max_held`** | Cap on *unharvested* product sitting on an animal tile. **Not** a lifetime limit — animals produce indefinitely if fed |
| **`pending_care_bonus`** | Banked yield from `CARE`, paid out in full on the animal's next scheduled production |

---

## 💹 Market terms

| Term | Definition |
|---|---|
| **`I0`** | Starting market inventory: **10,000 units** per product. The equilibrium point where price equals base |
| **Base price** | What a product sells for at exactly `I0` inventory |
| **Glut** | Market inventory **above** `I0`. Pushes price **down**. Written as a positive number (`+75`) |
| **Scarcity / deficit** | Market inventory **below** `I0`. Pushes price **up**. Written as negative (`−169`) |
| **Crash point** | How many units of glut it takes to drive a product to the $1 floor. Wool crashes at +59; wheat never crashes |
| **`T`** | A calibration constant per product: the production capacity of one 5×5 field over 24 days |
| **`target`** | "Moving `T` units past `I0` shifts the price by `target × base`" |
| **Shape function** | `linear`, `sq`, `sqrt`, or `log` — controls how sharply price responds. **Chosen independently for each side of equilibrium** |
| **Price floor** | $1. Sales at the floor still pay you but do **not** add to market inventory |
| **Marginal value** | What one *additional* unit is really worth, after accounting for the price drop it causes on everything else you sell. Can be **negative** |
| **Cumulative revenue** | Total collected selling N units from equilibrium. Misleading for decisions — use marginal value |
| **Town centre** | Consumes 1 of every non-fertilizer product per day. Flat all season |
| **Town shop** | Unlocks every 3 days (max 8), drawn **with replacement**. Each consumes its products every 4 turns |
| **Drain rate** | How fast the town removes a product from the market. Determines the sustainable sales rate |

---

## 🔬 Analysis terms

| Term | Definition |
|---|---|
| **Open-loop** | An agent that replays fixed pre-recorded actions regardless of board state. The V16-RC5 agent is open-loop |
| **State-driven** | An agent that decides each turn based on what is actually happening. What a competitive agent needs to be |
| **Trace / recording** | The 720-entry list of hardcoded actions inside the open-loop agent |
| **Desync** | When an open-loop agent's recording no longer matches reality, so its scripted actions silently do nothing |
| **No-op** | An action that has no effect — e.g. `WATER` on an empty tile, or any tile action on locked ground |
| **Seat advantage** | Player 0's structural edge. Measured at **~$1,006** here. Why all testing uses both seat orders |
| **Paired testing** | Running each matchup twice — once in each seat — so seat advantage cancels exactly |
| **Mirror match** | An agent playing against a copy of itself. Revenue drops sharply (181k → 124k) because both players crash the same markets |
| **W-T-L** | Wins-Ties-Losses. **The metric that matters**, since rating ignores margin |
| **Margin** | Money difference at game end. Useful for diagnosis, but **irrelevant to rating** |
| **Supply-constrained** | Limited by how much you can produce, not by the price you can get. **The central finding here** |
| **Price-constrained** | Limited by poor selling. The alternative hypothesis, which the evidence rejected |
| **Shed coupling** | The structural fact that market changes affect farming actions, because `PICKUP`/`FEED`/`FERTILIZE` read shed contents. Cause of every experimental failure |
| **Action budget** | The total unit-actions available in a season (6,673 for this agent). The real scarce resource |
| **Fibonacci hiring** | Each additional worker per day costs 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377… Makes labour unaffordable past ~14 hands |

---

## 🐍 Code and API terms

| Term | Definition |
|---|---|
| **`kaggle-environments`** | The pip package containing the game engine. `kaggriculture` ships inside it |
| **`obs`** | The observation dict passed to your agent each turn: `player`, `step`, `day`, `hour`, `farms`, `market`, `town`, `private` |
| **`private`** | The part of the observation only you can see: your `shed`, `seeds`, and worker `inventories` |
| **`farms`** | Public state for **both** players — tiles, money, worker positions. Your opponent's farm is visible; their shed is not |
| **`submission.py`** | The required filename for single-file submissions |
| **`submission.tar.gz`** | The required filename for multi-file bundles, with `main.py` at the root |
| **`agent(obs)`** | The required entry-point function |
| **`starter` / `random` / `pass`** | Built-in opponents. `starter` is a deterministic baseline; `pass` does nothing |
| **`env.run([a, b])`** | Plays one full episode between two agents |
| **`_commit_unit`** | Engine function processing one unit of a market order. **Returns `False` when money runs out, aborting the whole order** |

---

## 🎨 Reading the tables

| Symbol | Meaning |
|---|---|
| 🟢 | Good / favourable / untapped opportunity |
| 🟡 | Mixed / approaching a limit |
| 🟠 | Concerning |
| 🔴 | Bad / a hard constraint / oversupplied |
| ⚫ | Wasted / neutral |
| ✅ ❌ | Verified true / verified false |
| **+75** | Glut — inventory above `I0`, price below base |
| **−169** | Deficit — inventory below `I0`, price **above** base |

---

[← 13 Reproducing](13-reproduce.md) | **14 — Glossary** | [Index →](README.md)
