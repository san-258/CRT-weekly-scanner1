# 📊 CRT Weekly Scanner — Candle Range Theory

A research-backed weekly stock scanner that detects **Candle Range Theory (CRT)** setups using ICT/Smart Money Concepts methodology.

Backtested on **1,540 weekly CRT events** across 7 major tech/semi stocks (2015–2026).

---

## 🧠 What is Candle Range Theory (CRT)?

CRT is an ICT (Inner Circle Trader) concept based on **2-candle weekly price patterns**:

```
Week 1 — Mother Candle : Establishes a price range
Week 2 — Child Candle  : Sweeps liquidity beyond Mother, then reverses

Bullish CRT:  Child sweeps BELOW Mother low → recovers back above it
Bearish CRT:  Child sweeps ABOVE Mother high → recovers back below it

Target = the OTHER side of the Mother candle (opposite the sweep)
```

The logic: institutions manufacture a liquidity grab (the sweep), then drive price toward the opposite side of the range where their real orders sit.

---

## 📈 Backtest Results (1,540 Events, 2015–2026)

| Direction | Hit Rate | Avg Days to Target | Median Days |
|---|---|---|---|
| **Bullish CRT** | **66.2%** | 6.2 days | 5 days |
| Bearish CRT | 52.0% | 5.3 days | 5 days |

### Key Research Findings

| Finding | Detail |
|---|---|
| **Best mother candle size** | 2–4% range = 80.7% hit rate |
| **Worst mother candle size** | >8% range = 33% hit rate |
| **#1 predictor** | Child closing ABOVE midpoint → ~80% hit rate |
| **Child below midpoint** | ~20% hit rate — near coin flip |
| **Time in market** | 80% of winners hit target within 10 daily bars |
| **Doji mothers** | Unreliable — midpoint loses meaning |

---

## 🔍 Scanner Features

### Two Detection Modes

**1. Developing CRT** — Real-time current week analysis
- Detects sweep happening NOW (Mon/Tue)
- Tracks reversal forming (Wed/Thu)  
- Scores confirmation (Fri close)
- Detects 1H Change of Character (CHOCH) automatically

**2. Completed CRT** — Historical in-play setups
- Scans last 12 weeks for completed CRTs
- Shows which are still active (not yet hit target or invalidated)
- Flags child close vs midpoint quality

### Weighted Scoring System (15 points max)

```
STAGE 1 — Sweep Phase (max 4pts)
  Swept beyond mother extreme    1pt
  Sweep distance < 2%           2pts ★★  (deeper sweep = lower hit rate)
  Volume elevated on sweep       1pt

STAGE 2 — Reversal Phase (max 5pts)  
  Price recovering into range    1pt
  Long wicks / price slowing     1pt
  1H CHOCH confirmed            2pts ★★  (structural flip)
  Daily close back inside range  1pt

STAGE 3 — Confirmation Phase (max 6pts)
  Currently inside mother range  1pt
  Above/below midpoint          3pts ★★★ (#1 predictor from backtest)
  Volume expanding on recovery   1pt
  Reclaimed mother candle close  1pt
```

### Quality Filters

**Hard Disqualify (never shown):**
- Mother candle range outside 2–6% of price
- Sweep distance outside 0.3–4%

**Soft Penalties (shown with deductions):**
- Doji/near-doji mother candle (<15% body ratio) = **–2pts**
- Same-direction mother candle = **–1pt**
  - Ideal: Bullish CRT + Red (bearish) mother
  - Ideal: Bearish CRT + Green (bullish) mother

**Score Thresholds:**
- ≥75% = 🔥 High Conviction — actionable
- 50–74% = ⚡ Developing — watch list  
- <50% = 👀 Too early — do not enter

---

## 🚀 Quick Start

### Option A — Google Colab (Recommended, No Install)

1. Go to [colab.research.google.com](https://colab.research.google.com)
2. File → New notebook
3. In the first cell, run:

```python
!pip install yfinance pandas numpy colorama tabulate --quiet
```

4. Copy the contents of `crt_scanner.py` into the next cell
5. Edit the `TICKERS` list at the top
6. Run the cell

### Option B — Local Python

```bash
# 1. Clone the repo
git clone https://github.com/YOUR_USERNAME/crt-scanner.git
cd crt-scanner

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run the scanner
python crt_scanner.py
```

---

## ⚙️ Configuration

Open `crt_scanner.py` and edit the **CONFIG section** at the top:

```python
# ── YOUR TICKER LIST ─────────────────────────────────────────
TICKERS = [
    "AMAT", "NVDA", "AAPL", "MSFT", "AMD",
    "TSM",  "ASML", "QCOM", "MU",   "INTC",
    # Add any US stock ticker here
]

# ── FILTERS ──────────────────────────────────────────────────
SWEEP_MIN_PCT         = 0.003   # Min sweep = 0.3% beyond mother extreme
SWEEP_MAX_PCT         = 0.04    # Max sweep = 4% (deeper = likely failed CRT)
MOTHER_RANGE_MIN      = 0.02    # Mother candle min range = 2% of price
MOTHER_RANGE_MAX      = 0.06    # Mother candle max range = 6% of price
MOTHER_BODY_RATIO_MIN = 0.15    # Min body ratio (below = flagged as doji)
DOJI_SCORE_PENALTY    = 2       # Points deducted for doji mother
DIRECTION_MISMATCH_PENALTY = 1  # Points deducted for same-direction mother

# ── LOOKBACK ──────────────────────────────────────────────────
LOOKBACK_WEEKS        = 12      # Weeks to scan for completed CRTs

# ── VOLUME ───────────────────────────────────────────────────
VOL_ELEVATED_RATIO    = 1.05    # Child volume must be > mother * this ratio
```

---

## 📖 How to Read the Output

### Developing CRT Block

```
════════════════════════════════════════════════════════════
  PFE  🟢 BULLISH CRT — Stage 3 — Confirmation
  Day 5 of week  |  Mother Range: 5.42%
  Mother Body : 87.0% of range  | 🔴 Red  | ✅ Strong body
  Alignment   : ✅ Ideal
────────────────────────────────────────────────────────────
  Mother:  Low 26.77  |  Mid 27.5  |  High 28.23
  Current: 27.56  |  Sweep to: 26.68 (0.34%)
  Target:  28.23  |  Dist to target: 0.67 (2.4%)
────────────────────────────────────────────────────────────
  STAGE 1 — Sweep Phase   [3/4(w)]      ← weighted score
    ✅  Swept beyond mother extreme       (1pt)
    ✅  Sweep < 2%  (0.34%)              (2pts ★)
    ❌  Volume elevated on sweep          (1pt)

  STAGE 2 — Reversal Phase [5/5(w)]
    ✅  Price recovering into range       (1pt)
    ✅  Long wicks / slowing down         (1pt)
    ✅  1H CHOCH formed @ 27.57          (2pts ★)
    ✅  Daily close back inside range     (1pt)

  STAGE 3 — Confirmation  [6/6(w)]
    ✅  Inside mother range               (1pt)
    ✅  Above midpoint (27.5)            (3pts ★★★)  ← most important
    ✅  Volume expanding                  (1pt)
    ✅  Reclaimed mother close (28.19)    (1pt)

    TOTAL SCORE: 14/15  (93%)
  🔥 HIGH CONVICTION — All stages aligning
```

**Key things to look for:**
- `Mother Body %` — higher is better, <15% = doji flag
- `Alignment` — Ideal = opposite color mother
- `vs Midpoint` — **most important** single factor
- `Sweep %` — lower is better, ideally <2%
- `CHOCH` — structural confirmation on 1H

### Completed CRT Table

```
Ticker  Dir      Child Week   M.Range  Sweep  Child vs Mid  Target   Price   To Target  Status
PFE     Bullish  2026-04-13   5.42%    0.34%  ✅ Above      28.23    27.56   2.43%      🔄 IN PLAY
```

- `✅ Above/Below` = child closed on correct side of midpoint (~80% hit rate)
- `⚠️ Below/Above` = child closed wrong side (~20% hit rate)
- `🔄 IN PLAY` = target not yet hit, not invalidated

---

## 🎯 Trading Plan Framework

### Entry (1H Timeframe)

Once scanner flags a high conviction setup (≥75%):

```
1. Wait for 1H CHOCH (scanner detects this automatically)
2. After CHOCH, wait for retracement into 1H FVG (Fair Value Gap)
3. Enter at FVG with stop below sweep low (bullish) / above sweep high (bearish)
```

### Targets

```
TP1 → Mother candle midpoint (50% level)
TP2 → Mother candle open
TP3 → Mother candle HIGH (full CRT target) — take 50% here
TP4 → Next HTF draw (runner position)
```

### Stop Loss

```
Bullish CRT: Stop below child candle low (the sweep wick) - 0.5%
Bearish CRT: Stop above child candle high (the sweep wick) + 0.5%
```

### Time-Based Management

From backtest research:
- If no meaningful progress by **Day 7** → tighten stop
- **Day 10** = hard deadline for active management
- **Day 15** = exit regardless if target not hit

---

## 📂 Repository Structure

```
crt-scanner/
│
├── crt_scanner.py          # Main scanner (run this)
├── crt_backtest.py         # Full backtest engine with charts
├── requirements.txt        # Python dependencies
├── tickers/
│   ├── sp500.txt           # S&P 500 ticker list
│   ├── nasdaq100.txt       # NASDAQ 100 tickers
│   └── custom.txt          # Your custom list (edit this)
├── notebooks/
│   └── CRT_Scanner.ipynb   # Google Colab notebook version
└── README.md
```

---

## 📦 Requirements

```
yfinance>=0.2.18
pandas>=1.5.0
numpy>=1.23.0
colorama>=0.4.6
tabulate>=0.9.0
matplotlib>=3.6.0    # for backtest charts only
seaborn>=0.12.0      # for backtest charts only
```

---

## ⚠️ Disclaimer

This scanner is for **educational and research purposes only**.

- Not financial advice
- Past backtest performance does not guarantee future results
- Always do your own due diligence before trading
- Never risk more than you can afford to lose
- CRT is one tool — use it with broader market context

---

## 🙏 Acknowledgements

- **ICT (Inner Circle Trader)** — for the original CRT and SMC concepts
- **yfinance** — for free market data access
- Backtested on 1,540 weekly events across AMAT, NVDA, AAPL, MSFT, AMD, TSM, ASML (2015–2026)

---

## 📬 Contributing

Pull requests welcome. Please:
1. Fork the repo
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Commit your changes (`git commit -m 'Add some feature'`)
4. Push to the branch (`git push origin feature/your-feature`)
5. Open a Pull Request

---

*Built with ❤️ for the ICT/SMC trading community*
