[← 12 Roadmap](12-roadmap.md) | **13 — Reproducing This Work** | [Next: Glossary →](14-glossary.md)

---

# 13 — Reproducing This Work

![Runtime](https://img.shields.io/badge/full%20reproduction-~30%20min-3498DB?style=flat-square)

Every number in this documentation set, and how to regenerate it.

---

## ⚠️ Environment warning

> [!CAUTION]
> **Installing `kaggle-environments` upgraded numpy to 2.5.2 in the base conda
> environment, breaking `tensorflow-intel 2.18`, `ultralytics`, and `thinc`.** This is
> currently unresolved on this machine.

**Use a virtual environment instead:**

```bash
python -m venv .venv
# Windows
.venv/Scripts/activate
# macOS / Linux
source .venv/bin/activate

pip install -U kaggle-environments kaggle pandas matplotlib
```

Or, to repair the base environment:

```bash
pip install "numpy<2.1"
```

---

## 📦 Setup

```bash
pip install -U kaggle-environments
```

Version used: **1.32.6**. The `kaggriculture` environment ships inside the package.

Verify:

```python
from kaggle_environments import make
env = make('kaggriculture', configuration={'episodeSteps': 720})
env.run(['pass', 'pass'])
print([(i, s.reward) for i, s in enumerate(env.steps[-1])])
# -> [(0, 3000.0), (1, 3000.0)]
```

> [!NOTE]
> Importing `kaggle_environments` prints a large block of OpenSpiel warnings to stdout.
> Harmless. Suppress with `contextlib.redirect_stdout` or filter the output.

### Engine source

Worth reading directly — two findings came from it rather than the rules:

```
<python>/site-packages/kaggle_environments/envs/kaggriculture/kaggriculture.py
```

| Line | What's there |
|---|---|
| ~462 | `FERTILIZE` takes fertilizer from **worker inventory** (killed V18) |
| ~492 | `FEED` takes wheat from **worker inventory** |
| ~531 | `_process_market` — order processing, one unit at a time |
| ~660 | `_commit_unit` — a failed unit **aborts the entire order** |

---

## 🗂️ Repository layout

```
kaggriculture-analysis/
├── README.md                    technical writeup
├── main.py                      the agent (rename to submission.py to submit)
├── build_notebook.py            regenerates the Kaggle notebook
├── notebook/
│   └── kaggriculture-supply-constraint-analysis.ipynb
└── analysis/
    ├── market_model.py          verified price curve + town-demand simulator
    ├── harness.py               paired two-seat duel harness
    └── variants/
        ├── v17_main.py          adaptive reserve-price seller
        ├── v18_main.py          dump melon + fertilizer
        ├── v18b_main.py         dump melon only
        └── v19_main.py          no-op salvage
```

---

## ✅ Verifying the price model

Reproduces all nine rows of the official table:

```bash
cd kaggriculture-analysis
python analysis/market_model.py
```

Expected — every value matching, ending in `True`:

```
VERIFY P(I0+T), P(I0+2T):
  WHEAT         20   19
  CARROT        10    1
  TOMATO        24    9
  STRAWBERRY     1    1
  MELON          1    1
  EGG           40   39
  MILK           1    1
  WOOL           1    1
  FERTILIZER    60   20
```

---

## ⚔️ Running a duel

```python
import sys
sys.path.insert(0, 'analysis')
from harness import duel, play

# single game
print(play('main.py', 'main.py', seed=1))
# -> (124172.0, 123166.0, ['DONE', 'DONE'])

# full paired comparison — 15 seeds x both seat orders
duel('analysis/variants/v19_main.py', 'main.py', range(1, 16), 'V19 vs BASE')
```

> [!WARNING]
> **Always use `duel`, never a single `play`.** Seat 0 carries a ~$1,006 structural
> advantage. A single game — or 15 games all in the same seat — will mislead you.

Runtime: ~7s per game, so ~3.5 min for a 30-game comparison.

---

## 📊 Reproducing each headline number

| Claim | How to reproduce |
|---|---|
| Price model matches all 9 rows | `python analysis/market_model.py` |
| Baseline self-play 124,172 / 123,166 | `play('main.py','main.py',1)` |
| Baseline vs `starter` 131,869 / 3,504 | `play('main.py','starter',1)` |
| Baseline vs `pass` 181,057 | `play('main.py','pass',1)` |
| Seat advantage ~$1,006 | Self-play margin above |
| V17 result 0-0-24 | `duel('analysis/variants/v17_main.py','main.py',range(1,16))` |
| V18 result 2-0-28 | `duel('analysis/variants/v18_main.py','main.py',range(1,16))` |
| V18b result 7-16-7 | `duel('analysis/variants/v18b_main.py','main.py',range(1,16))` |
| V19 result 9-0-21 | `duel('analysis/variants/v19_main.py','main.py',range(1,16))` |

---

## 🔬 Decoding the agent's action trace

```python
import re, base64, zlib, json, collections

src = open('main.py', encoding='utf8').read()
blob = re.search(r"b85decode\('(.*?)'\)", src, re.S).group(1)
A = json.loads(zlib.decompress(base64.b85decode(blob)).decode())

print('steps:', len(A))                       # 720

ops = collections.Counter()
for a in A:
    ops[a['farmer'][0]] += 1
    for h in a.get('hands') or []:
        ops[h[0]] += 1

total = sum(ops.values())
move = sum(ops[k] for k in ('NORTH', 'SOUTH', 'EAST', 'WEST'))
print(f'total {total}, movement {move} ({move/total:.1%}), PASS {ops["PASS"]}')
# -> total 6673, movement 2855 (42.8%), PASS 324
```

---

## 📈 Full-season trajectory diagnostics

This is what produced the central diagnosis. Step the environment manually:

```python
import importlib.util, io, contextlib
from kaggle_environments import make

spec = importlib.util.spec_from_file_location('agent', 'main.py')
mod = importlib.util.module_from_spec(spec); spec.loader.exec_module(mod)

env = make('kaggriculture', configuration={'episodeSteps': 720, 'seed': 1})
env.reset(2)
st, step = env.state, 0
while not env.done and step < 720:
    acts = []
    for i in range(2):
        o = dict(st[i].observation); o['step'] = step; o['player'] = i
        if i == 0:
            acts.append(mod.agent(o))
        else:
            nh = len(st[1].observation['farms'][1]['hands'])
            acts.append({'farmer': ['PASS'], 'hands': [['PASS']]*nh, 'market': []})
    with contextlib.redirect_stdout(io.StringIO()):
        st = env.step(acts)
    step += 1
    if step % 72 == 0:
        p = st[0].observation
        glut = {k: v-10000 for k, v in p['market']['inventory'].items() if v != 10000}
        print(f"day {step//24:2d}  money {p['farms'][0]['money']:>9,.0f}  glut {glut}")
```

> [!TIP]
> **The market glut line is the whole diagnosis.** Negative values mean the town is
> out-draining you and prices are above base. Seven of nine end negative.

---

## 📓 Rebuilding the notebook

```bash
cd kaggriculture-analysis
python build_notebook.py
```

Writes `notebook/kaggriculture-supply-constraint-analysis.ipynb` (18 cells), embedding the
current `main.py` in the final `%%writefile submission.py` cell.

Execute-test it headless:

```bash
MPLBACKEND=Agg python -c "
import json, io, contextlib
nb = json.load(open('notebook/kaggriculture-supply-constraint-analysis.ipynb', encoding='utf8'))
ns = {'display': lambda *a, **k: None}
for i, c in enumerate(nb['cells']):
    if c['cell_type'] != 'code': continue
    src = ''.join(c['source'])
    if src.startswith('%%writefile'): continue
    with contextlib.redirect_stdout(io.StringIO()): exec(src, ns)
    print(f'cell {i}: OK')
"
```

---

## 📤 Submitting

> [!IMPORTANT]
> Single-file submissions **must** be named `submission.py`. Bundles must be
> `submission.tar.gz` with `main.py` at the root.

```bash
cp main.py submission.py
kaggle competitions submit kaggriculture -f submission.py -m "description"
```

Monitoring:

```bash
kaggle competitions submissions kaggriculture
kaggle competitions episodes <SUBMISSION_ID>
kaggle competitions leaderboard kaggriculture -s
kaggle competitions logs <EPISODE_ID> 0
```

Authentication — run in your own terminal (needs a real TTY):

```bash
kaggle auth login
```

Or save a token from <https://www.kaggle.com/settings/api> to `~/.kaggle/access_token`.

---

## 🎲 A note on randomness

Two independent random sources:

| Source | Impact |
|---|---|
| Shop draw (8 shops, with replacement) | **±47% revenue swing** ($169,274–$247,938 across 6 draws) |
| Weed spawn (0.5%/tile/day) | ~7 weeds per season |

> [!WARNING]
> **Any claimed improvement smaller than the shop-draw spread requires many seeds to
> detect.** This is why every comparison here uses 15 seeds × 2 seat orders. Do not trust a
> single-game result.

---

[← 12 Roadmap](12-roadmap.md) | **13 — Reproducing This Work** | [Next: Glossary →](14-glossary.md)
