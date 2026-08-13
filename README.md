# Kaggriculture — Complete Project Walkthrough

**Written for someone with zero prior context.** This document explains what the
competition is, what was in this folder at the start, everything I investigated, what I
found, what I built, what failed, and where things currently stand.

Date of work: **13 August 2026**.

---

## Table of contents

1. [What this competition is](#1-what-this-competition-is)
2. [How the game works](#2-how-the-game-works)
3. [What was in this folder when I started](#3-what-was-in-this-folder-when-i-started)
4. [Step 1 — Reading the rules and building a price model](#4-step-1--reading-the-rules-and-building-a-price-model)
5. [Step 2 — The single most important insight](#5-step-2--the-single-most-important-insight)
6. [Step 3 — Taking the existing agent apart](#6-step-3--taking-the-existing-agent-apart)
7. [Step 4 — Getting the real game running](#7-step-4--getting-the-real-game-running)
8. [Step 5 — The diagnosis](#8-step-5--the-diagnosis)
9. [Step 6 — Four experiments, four failures](#9-step-6--four-experiments-four-failures)
10. [Step 7 — The idea I killed with arithmetic](#10-step-7--the-idea-i-killed-with-arithmetic)
11. [Step 8 — Checking the actual leaderboard](#11-step-8--checking-the-actual-leaderboard)
12. [Mistakes I made and corrected](#12-mistakes-i-made-and-corrected)
13. [What was delivered](#13-what-was-delivered)
14. [Current status](#14-current-status)
15. [What to do next](#15-what-to-do-next)
16. [Glossary](#16-glossary)

---

## 1. What this competition is

**Kaggriculture** is a Kaggle *simulation* competition. It is **not** a machine-learning
competition — there is no training data, no test set, and no metric to fit. You write a
**bot** (an agent) that plays a farming game, and it competes head-to-head against other
people's bots on a live ladder.

| | |
|---|---|
| Started | 29 July 2026 |
| Entry / team-merger deadline | 23 September 2026 |
| **Final submission deadline** | **30 September 2026** |
| Leaderboard finalised | ~15 October 2026 |
| Prizes | **Places 1–10 each win $5,000** |

### How scoring works

This is important and frequently misunderstood.

- Every submission gets a **skill rating** (like a chess Elo). New submissions start at a
  **default rating of 600** and move up or down as they play games.
- Only the **win/loss/tie** matters. If you win by $1 or by $100,000, the rating change is
  identical. Margin is irrelevant.
- You may submit up to **5 agents per day**, but only your **latest 2 submissions** are
  tracked and used for final evaluation. Older ones stop playing.
- On the leaderboard only your **best** bot is displayed.
- On upload, a **validation episode** runs your agent against a copy of itself. If it
  crashes, the submission is marked `Error`.

**Practical consequence:** seeing "600" right after submitting is completely normal. It is
the starting value, not a score your agent earned. It climbs over the following hours as
games are played.

---

## 2. How the game works

Two players, each with their own farm. The season is **30 days × 24 turns = 720 turns**.
Whoever has the most money at the end wins. Unsold goods count for nothing.

### The farm

- A 10×10 grid split into four 5×5 quadrants. You start owning only the north-west one.
- The other three cost **$1,000 / $2,000 / $4,000** to unlock.
- A shed sits at the centre — your storage. It holds **100 items maximum**; anything over
  that at end of day is **thrown away**.
- You start with **$3,000**.

### Your workers

- One **farmer**, permanently.
- Plus **farm hands** you hire each day. They vanish at end of day and must be re-hired.
- Each worker performs **exactly one action per turn**. That is the fundamental currency
  of the game — actions, not money.
- **Hiring cost follows the Fibonacci sequence.** The nth hand hired in a day costs
  `fib(n)`: 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377… This matters enormously
  and is discussed in [section 10](#10-step-7--the-idea-i-killed-with-arithmetic).

### What you produce

| Product | Source | Cost | Base price |
|---|---|---|---|
| Wheat | crop | $10 seed | $25 |
| Carrot | crop | $20 seed | $35 |
| Tomato | crop (repeating) | $50 seed | $60 |
| Strawberry | crop (repeating) | $100 seed | $120 |
| Melon | crop | $80 seed | $250 |
| Egg | goose | $300 | $50 |
| Milk | cow | $400 | $160 |
| Wool | sheep | $500 | $200 |
| Fertilizer | any animal, free, 1/day | — | $100 |

Plants must be **watered daily** or they become weeds. Animals must be **fed wheat daily**
or they escape permanently.

### The market — the heart of the game

Both players sell into **one shared market**. This is the only way the two players
interact. Each product starts with an inventory of 10,000 units.

- **You sell → inventory rises → price falls.**
- **The town consumes → inventory falls → price rises.**

The town is made of shops that unlock every 3 days (up to 8), each consuming specific
products continuously, plus a town centre that takes one of everything per day.

---

## 3. What was in this folder when I started

```
AGENTS.md            official getting-started guide
README.md            official full game rules
desccription.docx    the competition's Kaggle page, saved as Word
notebook score.docx  a screenshot-ish record showing "Public Score 2966.9"
kaggriculture.zip    a zip containing copies of the two .md files
```

Partway through, you added:

```
v16-rc5-high-score-8c-4s-premium-market-lead.ipynb
```

This notebook contained the agent. **The "2966.9" was the headline number I was asked to
beat.** As [section 12](#12-mistakes-i-made-and-corrected) explains, that number turned
out not to belong to this account at all.

---

## 4. Step 1 — Reading the rules and building a price model

The rules describe *how* prices are computed but not how they *behave*. The formula:

```
price(inventory) = base + sign × amp × f(|inventory − 10000|)

  sign = +1 if inventory < 10000   (scarce  → price goes UP)
  sign = −1 if inventory > 10000   (glut    → price goes DOWN)
  amp  = target × base / f(T)
  f    = one of: linear, x², √x, ln(1+x)
```

Every product picks its **own shape and steepness independently for each side**. That is
the subtle part: a product can be violently sensitive to oversupply but barely react to
scarcity, or vice versa.

I reimplemented this in Python. **To confirm I had it right, I checked my implementation
against the nine-row reference table in the official README — and reproduced all nine rows
exactly.** Everything downstream rests on that check passing.

### What the model revealed

The key question for each product: *how many units can you dump before the price collapses
to the $1 floor?*

| Product | Crashes at glut of | Reads as |
|---|---|---|
| **Wool** | +59 | extremely fragile |
| **Strawberry** | +62 | extremely fragile |
| **Milk** | +76 | extremely fragile |
| **Melon** | +158 | fragile |
| **Fertilizer** | +493 | moderate |
| **Tomato** | +529 | moderate |
| **Carrot** | +842 | robust |
| **Wheat** | **never** | still $19 at +1,600 |
| **Egg** | **never** | still $37 at +1,600 |

Wheat and egg use a *logarithmic* glut curve, which flattens out — you essentially cannot
crash them. The expensive products use *linear* or *squared* curves and collapse almost
immediately.

---

## 5. Step 2 — The single most important insight

**The market is a flow system, not a stock.**

Most people intuitively model it as: "there are N units of demand in the season; sell N
units." That is wrong. The town **continuously drains** inventory, and draining inventory
**pushes prices back up**. So the real question is your production *rate* versus the town's
consumption *rate*.

I computed the season-long town drain per product:

| Product | Town drain per season |
|---|---|
| Wheat | 525 |
| Strawberry | 426 |
| Carrot | 327 |
| Milk | 327 |
| Tomato | 228 |
| Egg | 228 |
| Wool | 228 |
| **Melon** | **30** |
| **Fertilizer** | **0** |

**No shop demands melon** (only the town centre, once a day). **Nothing at all consumes
fertilizer.** Those two products never regenerate — they are one-shot pools that both
players race each other to drain. Everything else replenishes continuously.

This single table reframes the whole game.

---

## 6. Step 3 — Taking the existing agent apart

The notebook's agent was not what it appeared. Inside `main.py` was a large
base64-and-compressed blob. I decoded it.

**It is a hardcoded recording of 720 turns of actions.** The agent does not think. On turn
`N` it replays action number `N` from the recording, regardless of what is actually
happening on the board. This is called an **open-loop** agent.

The notebook itself credits the source: the route was reconstructed from public replays of
another Kaggle user (*Nikita Lugovoy*, submission `55440039`).

Around the replay sit two small patches: a weed-repair routine, and a "sell one turn
early" tweak for premium goods.

### Profiling the recording

I counted every action across all 720 turns:

| Action | Count |
|---|---|
| WATER | 1,010 |
| CARE | 967 |
| **NORTH / SOUTH / EAST / WEST** | **2,855** |
| HARVEST | 390 |
| **PASS (do nothing)** | **324** |
| COLLECT_FERTILIZER | 296 |
| FEED | 290 |
| PLANT | 199 |
| PICKUP | 135 |
| FERTILIZE | 72 |
| everything else | ~135 |

**Total: 6,673 actions. 42.8% of them are walking. 4.9% are doing nothing.** Only 52.4% do
productive work. That looked like the biggest opportunity in the game.

What it sells over a season: 479 wheat, 320 milk, 300 fertilizer, 300 strawberry, 154
wool, 126 melon. It buys 8 cows and 4 sheep (hence "8C/4S" in the notebook title), and
buys 2 extra land quadrants.

---

## 7. Step 4 — Getting the real game running

Up to this point everything was theory. I installed the actual game engine:

```bash
pip install -U kaggle-environments
```

The `kaggriculture` environment ships inside that package. A full 720-turn game runs in
about 7 seconds, which makes real experimentation possible.

### The measurement harness

Comparing two bots fairly is harder than it looks. **The player in seat 0 has a built-in
advantage of roughly $1,006** — I measured this by playing the baseline against an
identical copy of itself. If you only test one seat order, that advantage alone can make a
worse bot look better.

So the harness plays **every matchup twice — once in each seat** — across 15 different
random seeds, giving **30 games per comparison**.

### Baseline reference points

| Matchup | Result |
|---|---|
| Baseline vs itself | 124,172 vs 123,166 |
| Baseline vs `starter` bot | 131,869 vs 3,504 |
| Baseline vs `pass` bot (does nothing) | 181,057 |

My offline market simulator had predicted ~$121k for the mirror match. The real engine
gave $124k. **The model was validated against reality.**

---

## 8. Step 5 — The diagnosis

I ran the baseline for a full season and recorded the market state at the end. This is the
result that determined everything afterwards.

| Product | Final price | vs base price | State |
|---|---|---|---|
| Strawberry | $229 | **191%** | starved |
| Wheat | $40 | **160%** | starved |
| Milk | $249 | **156%** | starved |
| Tomato | $78 | **130%** | starved |
| Carrot | $42 | **120%** | starved |
| Wool | $240 | **120%** | starved |
| Egg | $58 | **116%** | starved |
| Melon | $194 | 78% | oversupplied |
| Fertilizer | $64 | 64% | oversupplied |

**Seven of the nine products end the season trading ABOVE their base price.**

Read that carefully. It means the market is *desperate* for goods that nobody is
producing. The agent is not failing to sell well — it is failing to **produce enough**.

And the only two oversupplied products are melon and fertilizer, which are precisely the
two with no town demand regenerating them. The model predicted exactly this.

**Conclusion: the agent is supply-constrained, not price-constrained.** Any effort spent
making it sell more cleverly is wasted effort.

---

## 9. Step 6 — Four experiments, four failures

I tested that conclusion by trying hard to beat it. Each variant was measured over 30
paired games.

| # | Idea | Result | Margin |
|---|---|---|---|
| V17 | Hold stock back for a better price | **0 wins, 24 losses** | −$149,090 |
| V18 | Instantly sell melon + fertilizer | **2 wins, 28 losses** | −$18,402 |
| V18b | Instantly sell melon only | 7-16-7 | **exactly $0** |
| V19 | Use up actions that were doing nothing | 9-0-21 | −$68 |

### V17 — the reserve-price seller (catastrophic)

**Idea:** don't sell into a bad price; wait for the town to drain inventory and lift it.

**Result:** lost every single game, by $149,000 on average.

**Why:** I set the minimum acceptable price slightly above base. Since price *equals* base
at the starting inventory, nothing ever met the threshold, so the agent sold nothing early.
Its money hit **$0 by day 6**. It could then never afford its cows, its extra land, or its
strawberry seeds. The farm produced **nothing at all** for the remaining 24 days.

**Lesson:** those early fertilizer sales on days 2–4 aren't opportunism, they're **working
capital**. The whole plan is financed by them.

### V18 — dump the no-regeneration goods (bad)

**Idea:** melon and fertilizer never regenerate, so their price can only fall. Holding them
is strictly worse than selling now. Sell on sight.

**Result:** lost 28 of 30.

**Why:** I read the game engine's source and found it. `FERTILIZE` takes fertilizer from
**the worker's pockets**, and workers fill their pockets by picking fertilizer up **from
the shed**. By selling fertilizer the instant it appeared, I emptied the shed before
workers could collect it — so all **72 fertilize actions did nothing**, and crop yields
collapsed.

### V18b — melon only (the control experiment)

To prove fertilizer was the culprit, I re-ran with melon alone.

**Result: mean margin exactly $0, and 16 of the 30 games were bit-for-bit identical.**

That confirms two things: fertilizer coupling explained *all* of V18's loss, and the agent
was already selling melon promptly. **There was no headroom to find.**

### V19 — salvaging wasted actions (neutral)

**Idea:** 324 actions do literally nothing, plus more that silently fail because the replay
has drifted from reality. Detect those and substitute a free useful action instead
(water a dry plant, care for an animal, collect fertilizer). This cannot displace intended
behaviour because it only fires when the scripted action provably does nothing.

**Result:** 9-0-21, margin −$68. Essentially neutral.

**Why it didn't help:** the shed is already at its 100-item cap around day 18. Extra
harvested goods just overflow and get discarded.

### The structural lesson

All four failures share one root cause:

> **The selling system and the farming system are coupled through the shed.**
> `PICKUP`, `FEED`, and `FERTILIZE` all read from shed contents. Any change to selling
> silently breaks farming actions downstream.

The sell side is not an independent subsystem you can optimise in isolation. **This rules
out the entire cheap half of the search space** — and that is the genuinely useful output
of these experiments.

---

## 10. Step 7 — The idea I killed with arithmetic

The obvious response to "you need more production" is: buy more land, hire more workers.

I found a clean opening. The agent buys only 3 of the 4 quadrants — **the south-east
quadrant (25 tiles) is never purchased**, so a new operation there could never collide with
the existing route. I also verified that the recording finishes hiring by hour 1 on nearly
every day, so extra workers hired later would slot safely onto the end of the roster.

Then I did the cost arithmetic, and it killed the idea.

Because hiring cost is Fibonacci, and the agent **already hires 10–14 workers a day**, the
next four workers cost:

```
11th hand:  $89
12th hand: $144
13th hand: $233
14th hand: $377
           ------
           $843 per day  →  ~$14,000 over the useful window
```

Against roughly **$8,500** of carrot revenue from those 25 tiles.

**Labor, not land, is the binding cost.** I killed the idea before writing it rather than
spending hours building something the numbers already ruled out.

This is why "just add more production" is not a patch — it requires making *existing*
actions more productive, i.e. attacking that 42.8% walking overhead, which means rewriting
the routing entirely rather than bolting something on.

---

## 11. Step 8 — Checking the actual leaderboard

You reported the score showing as 600 and were concerned it was a regression. I installed
the Kaggle CLI and (after you authenticated) pulled the real data.

**Nothing was wrong.** The findings:

| | |
|---|---|
| Submissions on the account | **exactly one** |
| Submission ID | 55486150 |
| Score at time of check | **1,846.4** (already climbed from 600) |
| Episodes played | 11 completed + 1 validation, **zero errors** |
| **Rank** | **920 of 4,283** |

Leaderboard context: median score is **736.9**, top is **3,236.5**, only **28 teams** are
above 3,000.

The 600 was simply the starting value. Eleven games later it had risen to 1,846.4 and was
still climbing.

**But this check also uncovered something bigger** — see the next section.

---

## 12. Mistakes I made and corrected

Recording these because they matter to how you read the rest.

### Mistake 1 — I believed the 2,966.9 was your score

The task was framed as "optimise the score 2,961.7 / 2,966.9". I took that at face value
and built the entire analysis around beating it.

**It was never this account's score.** The leaderboard check proved the account has exactly
**one** submission. That 2,966.9 is the public score displayed on the **original author's**
notebook — the notebook credits its route to another user's replays, and it arrived here
already carrying its author's number.

A single `kaggle competitions submissions kaggriculture` call at the very start would have
settled this in seconds. I should have verified the baseline before optimising against it.

### Mistake 2 — I overstated a bug

I found that on turn 0 the agent tries to spend about $3,212 when it only has $3,000, so
some orders fail. I initially reported this as a significant deterministic loss of ~3 melon
seeds (~$2,600 per game).

**Then I actually tested it.** The recording **self-heals on turn 1** — it re-buys exactly
the seeds that failed. The real loss is about one worker on day 0. Nearly nothing.

I had reasoned correctly from the engine source but hadn't checked the very next turn.

### Mistake 3 — a side effect on your machine

Installing `kaggle-environments` upgraded numpy to 2.5.2, which **breaks
`tensorflow-intel 2.18`, `ultralytics`, and `thinc`** in your base conda environment. This
is still outstanding and should be fixed by pinning numpy back or moving this work into a
separate virtual environment.

---

## 13. What was delivered

A folder `kaggriculture-analysis/`, initialised as a git repository with two commits:

```
kaggriculture-analysis/
├── README.md                  full technical writeup
├── main.py                    the submission agent
├── build_notebook.py          regenerates the notebook
├── notebook/
│   └── kaggriculture-supply-constraint-analysis.ipynb
└── analysis/
    ├── market_model.py        the verified price curve + town-demand simulator
    ├── harness.py             the two-seat paired duel harness
    └── variants/
        ├── v17_main.py
        ├── v18_main.py
        ├── v18b_main.py
        └── v19_main.py
```

### The notebook

18 cells, publishable to Kaggle. It derives the price model, verifies it against the
official table live, shows the supply-constraint diagnosis with charts, reports all four
failed experiments with explanations, and writes the submission file.

I execute-tested every analysis cell headless — all pass, and the price-model verification
prints `True`.

Per your last instruction, the final cell writes **`submission.py`** (Kaggle requires
single-file submissions to use that name; bundles must be `submission.tar.gz`).

### An important caveat on the agent

**The agent is not original work.** It is the open-loop replay taken from the community
notebook, which itself reconstructs another competitor's route from public replays. I
included an explicit attribution note directly above it in both the notebook and the
README.

Analysing public replays is normal and permitted on Kaggle. But most competition rules
carry an "independently developed" clause, and this competition pays $5,000 for a top-10
finish. **Reading the Kaggriculture rules page before publishing under your name is worth
doing.** The analysis, price model, harness, and four variants are original; the agent is
not.

---

## 14. Current status

| Item | State |
|---|---|
| Live submission | 55486150, score 1,846.4, rank 920/4,283, still converging |
| Errors | none — 11 clean episodes |
| Best agent available | the original V16-RC5 (nothing I built beat it) |
| Repo | built, committed, **not yet pushed to GitHub** |
| Notebook | built and tested, **not yet uploaded to Kaggle** |
| numpy breakage | **unresolved** |

**I did not produce a better agent.** Four attempts, all measured, all rejected. What the
work produced instead is a validated model of the game's economics and a proof of where the
remaining opportunity is and is not.

---

## 15. What to do next

### Immediately

- **Don't submit anything.** The live bot is mid-convergence. A new submission restarts at
  600, and only your latest 2 are tracked.
- **Fix numpy** — pin it back or move this project into its own virtual environment.

### To push the repo

You need to create it on GitHub first (the CLI isn't authenticated here):

```bash
cd "kaggriculture-analysis"
gh repo create kaggriculture-analysis --public --source=. --push
```

### To actually climb the leaderboard

Top 10 pays $5,000 and needs roughly **3,200**. Even fully converged, this agent would land
around rank 39. Incremental patching will not get there — the experiments demonstrated
that directly.

The path that would work is a **state-driven agent written from scratch**, targeting the
two things the analysis identified:

1. **Cut the 42.8% walking overhead.** Animals allow four productive actions on one tile
   (feed, care, harvest, collect fertilizer) with **zero movement**, while every crop costs
   a move per watering. A dense block of animals beside the shed converts walking directly
   into production. This is the single largest line in the action budget.

2. **Open the untapped demand.** Egg, carrot, and tomato together are roughly **780 units
   per season of continuously replenishing demand** that strong agents ignore completely.
   Eggs are the standout — impossible to crash, and geese are the cheapest animal at $300.

That is a multi-day build, not a patch.

---

## 16. Glossary

| Term | Meaning |
|---|---|
| **Agent / bot** | The program you submit; it receives the game state and returns actions |
| **Episode** | One complete 720-turn game between two agents |
| **Open-loop** | An agent that replays fixed pre-recorded actions, ignoring the board |
| **State-driven** | An agent that decides each turn based on what is actually happening |
| **Skill rating** | Elo-like score; starts at 600 and moves with wins and losses |
| **Glut** | Market inventory above the 10,000 starting level; pushes price down |
| **Base price** | The price a product sells for at exactly 10,000 inventory |
| **Seat advantage** | Player 0's structural edge (~$1,006 here); why every test runs both seats |
| **Paired testing** | Running each matchup in both seat orders so the advantage cancels |
| **The shed** | Central storage, 100-item cap; overflow is destroyed at end of day |
| **Supply-constrained** | Limited by how much you can make, not by the price you can get |
| **Fibonacci hiring** | Each extra worker per day costs 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89… |
| **Validation episode** | Automatic self-play run on upload to check the agent doesn't crash |
