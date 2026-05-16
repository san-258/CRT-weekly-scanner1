"""
╔══════════════════════════════════════════════════════════════════╗
║          CRT WEEKLY SCANNER — COMPLETED + DEVELOPING            ║
║                                                                  ║
║  Candle Range Theory (ICT/SMC) — Weekly Timeframe               ║
║  Backtested: 1,540 events | Bullish hit rate: 66.2%             ║
║                                                                  ║
║  Detects:                                                        ║
║   1. Completed weekly CRT (prior weeks, still in play)          ║
║   2. Developing CRT (current week — real time)                  ║
║                                                                  ║
║  Developing CRT Stages:                                          ║
║   Stage 1 — Sweep Phase    (Mon/Tue)                            ║
║   Stage 2 — Reversal Phase (Wed/Thu)                            ║
║   Stage 3 — Confirmation   (Fri close)                          ║
║                                                                  ║
║  Scoring: Weighted 15pt system — midpoint (3pts) is #1 factor   ║
╚══════════════════════════════════════════════════════════════════╝

USAGE:
    python crt_scanner.py                    # uses TICKERS list below
    python crt_scanner.py --file tickers/custom.txt
    python crt_scanner.py --file tickers/sp500.txt
    python crt_scanner.py --tickers AAPL MSFT NVDA

Google Colab:
    # Add at top of cell:
    # !pip install yfinance pandas numpy colorama tabulate --quiet
    # Then paste this entire file and run
"""

import argparse
import sys
import os
import yfinance as yf
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
import warnings
warnings.filterwarnings("ignore")

try:
    from colorama import Fore, Style, init
    init(autoreset=True)
    HAS_COLOR = True
except ImportError:
    HAS_COLOR = False
    class Fore:
        GREEN = CYAN = RED = YELLOW = WHITE = MAGENTA = ""
    class Style:
        BRIGHT = RESET_ALL = DIM = ""

try:
    from tabulate import tabulate
    HAS_TABULATE = True
except ImportError:
    HAS_TABULATE = False


# ════════════════════════════════════════════════════════════════
# 0. CONFIG — Edit these settings
# ════════════════════════════════════════════════════════════════

# Default ticker list (used when no --file or --tickers argument given)
TICKERS = [
    "AMAT", "NVDA", "AAPL", "MSFT", "AMD",
    "TSM",  "ASML", "QCOM", "MU",   "INTC",
    "GOOGL","META", "AMZN", "AVGO", "ARM"
]

# ── CRT Quality Filters ───────────────────────────────────────
SWEEP_MIN_PCT         = 0.003   # Min sweep = 0.3% beyond mother extreme
SWEEP_MAX_PCT         = 0.04    # Max sweep = 4% (deeper = likely failed CRT)
MOTHER_RANGE_MIN      = 0.02    # Mother candle min range = 2% of price
MOTHER_RANGE_MAX      = 0.06    # Mother candle max range = 6% of price
                                # Research: 2-4% = 80.7% hit rate, >8% = 33%

# ── Doji Filter ───────────────────────────────────────────────
# Body ratio = |Open - Close| / (High - Low)
# Doji mothers have weak midpoints — unreliable for CRT
# Soft penalty (not hard disqualify) so setup still shows with lower score
MOTHER_BODY_RATIO_MIN = 0.15    # Below this = flagged as doji
DOJI_SCORE_PENALTY    = 2       # Points deducted for doji mother candle

# ── Direction Alignment ───────────────────────────────────────
# Best CRT: Mother body is OPPOSITE to CRT direction
#   Bullish CRT + Red (bearish) mother = trapped sellers = ideal
#   Bearish CRT + Green (bullish) mother = trapped buyers = ideal
# Same-direction mother = weaker liquidity pool
DIRECTION_MISMATCH_PENALTY = 1  # Points deducted for same-direction mother

# ── Lookback & Volume ─────────────────────────────────────────
LOOKBACK_WEEKS        = 12      # How many prior weeks to scan for completed CRTs
VOL_ELEVATED_RATIO    = 1.05    # Sweep week volume > mother week * this ratio


# ════════════════════════════════════════════════════════════════
# 1. TICKER LOADING
# ════════════════════════════════════════════════════════════════
def load_tickers_from_file(filepath):
    """Load tickers from a text file (one per line, # = comment)."""
    if not os.path.exists(filepath):
        print(f"  ⚠  File not found: {filepath}")
        return []
    tickers = []
    with open(filepath, "r") as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith("#"):
                tickers.append(line.upper())
    return tickers


# ════════════════════════════════════════════════════════════════
# 2. DATA FETCHING
# ════════════════════════════════════════════════════════════════
def fetch_all_data(ticker):
    """Fetch weekly, daily, and 1H OHLCV data for a ticker."""
    end   = datetime.today()
    start = end - timedelta(weeks=LOOKBACK_WEEKS + 4)

    weekly = yf.download(ticker, start=start, end=end,
                         interval="1wk", auto_adjust=True, progress=False)
    daily  = yf.download(ticker, start=start, end=end,
                         interval="1d",  auto_adjust=True, progress=False)

    # 1H only available for last 60 days (yfinance limit)
    h1_start = end - timedelta(days=59)
    hourly = yf.download(ticker, start=h1_start, end=end,
                         interval="1h", auto_adjust=True, progress=False)

    for df in [weekly, daily, hourly]:
        if isinstance(df.columns, pd.MultiIndex):
            df.columns = df.columns.get_level_values(0)
        df.index = pd.to_datetime(df.index).tz_localize(None)

    return weekly, daily, hourly


# ════════════════════════════════════════════════════════════════
# 3. 1H CHOCH DETECTION
# ════════════════════════════════════════════════════════════════
def detect_1h_choch(hourly, since, direction,
                    sweep_low=None, sweep_high=None):
    """
    Detect Change of Character (CHOCH) on the 1H chart after a sweep.

    Bullish CHOCH: After sweeping below mother low, price breaks a
                   prior 1H swing high — structure has flipped bullish.
    Bearish CHOCH: After sweeping above mother high, price breaks a
                   prior 1H swing low — structure has flipped bearish.

    Returns dict with detected flag, level, and timestamp.
    """
    result = {"detected": False, "level": None, "time": None,
              "bars_after_sweep": None}

    h = hourly[hourly.index >= pd.Timestamp(since)].copy()
    if len(h) < 5:
        return result

    highs  = h["High"].values
    lows   = h["Low"].values
    closes = h["Close"].values
    times  = h.index

    if direction == "Bullish" and sweep_low is not None:
        # Look for a 1H swing high that gets broken to the upside
        for i in range(2, len(h)):
            if highs[i-1] > highs[i-2] and highs[i-1] > highs[i]:
                swing_high = highs[i-1]
                for j in range(i, len(h)):
                    if closes[j] > swing_high:
                        return {"detected": True, "level": round(swing_high, 2),
                                "time": times[j], "bars_after_sweep": j}

    elif direction == "Bearish" and sweep_high is not None:
        # Look for a 1H swing low that gets broken to the downside
        for i in range(2, len(h)):
            if lows[i-1] < lows[i-2] and lows[i-1] < lows[i]:
                swing_low = lows[i-1]
                for j in range(i, len(h)):
                    if closes[j] < swing_low:
                        return {"detected": True, "level": round(swing_low, 2),
                                "time": times[j], "bars_after_sweep": j}
    return result


# ════════════════════════════════════════════════════════════════
# 4. DEVELOPING CRT SCANNER (Current Week)
# ════════════════════════════════════════════════════════════════
def scan_developing_crt(ticker, weekly, daily, hourly):
    """
    Analyze the CURRENT incomplete week for a developing CRT setup.
    Scores each stage as it develops in real time.

    Returns list of result dicts (one per direction found), or None.
    """
    if len(weekly) < 2:
        return None

    # Mother candle = last COMPLETED week
    # Current week  = this week's daily bars so far
    last_completed_week = weekly.iloc[-2]
    current_week_start  = weekly.index[-1]

    m_high      = float(last_completed_week["High"])
    m_low       = float(last_completed_week["Low"])
    m_open      = float(last_completed_week["Open"])
    m_close     = float(last_completed_week["Close"])
    m_vol       = float(last_completed_week["Volume"])
    m_range     = m_high - m_low
    m_range_pct = m_range / m_close
    m_mid       = (m_high + m_low) / 2.0

    # ── HARD DISQUALIFY — mother range outside 2-6% ─────────────
    # Research: >8% mother = 33% hit rate (coin flip or worse)
    mother_quality = MOTHER_RANGE_MIN <= m_range_pct <= MOTHER_RANGE_MAX
    if not mother_quality:
        return None

    # ── DOJI DETECTION — soft penalty, not hard disqualify ───────
    # Volatile weeks compress candle bodies into doji/spinning tops.
    # Hard disqualify kills too many valid setups in high-vol environments.
    # Instead: flag it, deduct 2pts, let trader see and decide.
    m_body       = abs(m_open - m_close)
    m_body_ratio = m_body / m_range if m_range > 0 else 0
    m_is_bullish = m_close > m_open    # True = green candle
    m_is_doji    = m_body_ratio < MOTHER_BODY_RATIO_MIN

    # Current week daily bars
    curr_daily = daily[daily.index >= current_week_start].copy()
    if curr_daily.empty:
        return None

    latest_price  = float(curr_daily["Close"].iloc[-1])
    curr_high     = float(curr_daily["High"].max())
    curr_low      = float(curr_daily["Low"].min())
    curr_vol_avg  = float(curr_daily["Volume"].mean())
    num_days      = len(curr_daily)

    # ── Sweep Detection ──────────────────────────────────────────
    bull_sweep_pct = (m_low - curr_low) / m_low   if curr_low  < m_low   else 0
    bear_sweep_pct = (curr_high - m_high) / m_high if curr_high > m_high  else 0

    bull_sweep_ok = SWEEP_MIN_PCT <= bull_sweep_pct <= SWEEP_MAX_PCT
    bear_sweep_ok = SWEEP_MIN_PCT <= bear_sweep_pct <= SWEEP_MAX_PCT

    results = []

    for direction in ["Bullish", "Bearish"]:
        sweep_ok  = bull_sweep_ok  if direction == "Bullish" else bear_sweep_ok
        sweep_pct = bull_sweep_pct if direction == "Bullish" else bear_sweep_pct

        if not sweep_ok:
            continue

        sweep_extreme = curr_low if direction == "Bullish" else curr_high

        # ── STAGE 1: Sweep Phase ─────────────────────────────────
        s1_swept        = True
        s1_sweep_pct_ok = sweep_pct <= 0.02   # <2% sweep = cleaner grab
        s1_vol_elevated = curr_vol_avg > (m_vol / 5) * VOL_ELEVATED_RATIO
        # Note: divide by 5 because daily avg vs full weekly volume

        # Weighted: Sweep<2% = 2pts (most impactful sweep filter)
        stage1_score = (1 * s1_swept +
                        2 * s1_sweep_pct_ok +
                        1 * s1_vol_elevated)
        stage1_max   = 4

        # ── STAGE 2: Reversal Phase ──────────────────────────────
        if direction == "Bullish":
            s2_recovering        = latest_price > m_low
            s2_wicking           = any(
                (float(r["High"]) - float(r["Close"])) /
                max(float(r["High"]) - float(r["Low"]), 0.01) > 0.35
                for _, r in curr_daily.iterrows()
            )
            s2_daily_close_inside = float(curr_daily["Close"].iloc[-1]) > m_low
        else:
            s2_recovering        = latest_price < m_high
            s2_wicking           = any(
                (float(r["Close"]) - float(r["Low"])) /
                max(float(r["High"]) - float(r["Low"]), 0.01) > 0.35
                for _, r in curr_daily.iterrows()
            )
            s2_daily_close_inside = float(curr_daily["Close"].iloc[-1]) < m_high

        choch = detect_1h_choch(
            hourly,
            since      = current_week_start - timedelta(days=1),
            direction  = direction,
            sweep_low  = sweep_extreme if direction == "Bullish" else None,
            sweep_high = sweep_extreme if direction == "Bearish" else None
        )

        # Weighted: CHOCH = 2pts (structural confirmation is critical)
        stage2_score = (1 * s2_recovering +
                        1 * s2_wicking +
                        2 * int(choch["detected"]) +
                        1 * s2_daily_close_inside)
        stage2_max   = 5

        # ── STAGE 3: Confirmation Phase ──────────────────────────
        if direction == "Bullish":
            s3_inside_range   = m_low < latest_price < m_high
            s3_above_midpoint = latest_price > m_mid
            s3_heading_to_open= latest_price > m_close
        else:
            s3_inside_range   = m_low < latest_price < m_high
            s3_above_midpoint = latest_price < m_mid
            s3_heading_to_open= latest_price < m_close

        s3_vol_expanding = (
            float(curr_daily["Volume"].iloc[-1]) >
            float(curr_daily["Volume"].iloc[-2])
        ) if num_days >= 2 else False

        # Weighted: vs Midpoint = 3pts — #1 predictor from 1,540-event backtest
        # ~80% hit rate when child closes above/below midpoint
        # ~20% hit rate when child closes on wrong side
        stage3_score = (1 * s3_inside_range +
                        3 * s3_above_midpoint +
                        1 * s3_vol_expanding +
                        1 * s3_heading_to_open)
        stage3_max   = 6

        # ── Direction Alignment Penalty ───────────────────────────
        # Ideal: Bullish CRT + Red mother (trapped sellers below)
        # Ideal: Bearish CRT + Green mother (trapped buyers above)
        mother_aligned    = (not m_is_bullish) if direction == "Bullish" else m_is_bullish
        direction_penalty = 0 if mother_aligned else DIRECTION_MISMATCH_PENALTY

        # ── Doji Penalty ──────────────────────────────────────────
        doji_penalty = DOJI_SCORE_PENALTY if m_is_doji else 0

        # ── Final Score ───────────────────────────────────────────
        raw_score   = stage1_score + stage2_score + stage3_score
        total_score = max(raw_score - direction_penalty - doji_penalty, 0)
        total_max   = stage1_max + stage2_max + stage3_max
        score_pct   = total_score / total_max * 100

        # ── Stage Label ───────────────────────────────────────────
        if num_days <= 2:
            current_stage = "Stage 1 — Sweep"
        elif num_days <= 3:
            current_stage = "Stage 2 — Reversal"
        else:
            current_stage = "Stage 3 — Confirmation"

        results.append({
            "ticker"             : ticker,
            "type"               : "DEVELOPING",
            "direction"          : direction,
            "current_stage"      : current_stage,
            "days_in"            : num_days,
            "mother_range_pct"   : round(m_range_pct * 100, 2),
            "mother_body_ratio"  : round(m_body_ratio * 100, 1),
            "mother_body_dir"    : "🟢 Green" if m_is_bullish else "🔴 Red",
            "m_is_doji"          : m_is_doji,
            "mother_aligned"     : mother_aligned,
            "direction_penalty"  : direction_penalty,
            "doji_penalty"       : doji_penalty,
            "mother_quality"     : mother_quality,
            "m_high"             : round(m_high, 2),
            "m_low"              : round(m_low, 2),
            "m_mid"              : round(m_mid, 2),
            "m_open"             : round(m_open, 2),
            "current_price"      : round(latest_price, 2),
            "sweep_extreme"      : round(sweep_extreme, 2),
            "sweep_pct"          : round(sweep_pct * 100, 2),
            "target"             : round(m_high if direction == "Bullish" else m_low, 2),
            "s1_swept"           : s1_swept,
            "s1_sweep_pct_ok"    : s1_sweep_pct_ok,
            "s1_vol_elevated"    : s1_vol_elevated,
            "s1_score"           : f"{stage1_score}/{stage1_max}(w)",
            "s2_recovering"      : s2_recovering,
            "s2_wicking"         : s2_wicking,
            "s2_choch"           : choch["detected"],
            "s2_choch_level"     : choch["level"],
            "s2_daily_inside"    : s2_daily_close_inside,
            "s2_score"           : f"{stage2_score}/{stage2_max}(w)",
            "s3_inside_range"    : s3_inside_range,
            "s3_above_mid"       : s3_above_midpoint,
            "s3_vol_expanding"   : s3_vol_expanding,
            "s3_vs_open"         : s3_heading_to_open,
            "s3_score"           : f"{stage3_score}/{stage3_max}(w)",
            "total_score"        : f"{total_score}/{total_max}",
            "score_pct"          : round(score_pct, 0),
        })

    return results if results else None


# ════════════════════════════════════════════════════════════════
# 5. COMPLETED CRT SCANNER (Prior Weeks)
# ════════════════════════════════════════════════════════════════
def scan_completed_crt(ticker, weekly, daily):
    """
    Scan last N completed weekly bars for CRT setups.
    For each found, track whether target has been hit, invalidated,
    or is still in play — using forward daily bars.
    """
    if len(weekly) < 3:
        return []

    results = []
    w = weekly.dropna().reset_index()

    for i in range(1, min(LOOKBACK_WEEKS, len(w) - 1)):
        mother = w.iloc[-(i + 1)]
        child  = w.iloc[-i]

        m_high      = float(mother["High"])
        m_low       = float(mother["Low"])
        m_open      = float(mother["Open"])
        m_close     = float(mother["Close"])
        m_range     = m_high - m_low
        m_range_pct = m_range / m_close
        m_mid       = (m_high + m_low) / 2.0

        c_high  = float(child["High"])
        c_low   = float(child["Low"])
        c_close = float(child["Close"])
        child_date = child["Date"] if "Date" in child.index else child.name

        # Range filter
        if not (MOTHER_RANGE_MIN <= m_range_pct <= MOTHER_RANGE_MAX):
            continue

        # Doji filter — only skip pure dojis (<5%) in completed scanner
        m_body       = abs(m_open - m_close)
        m_body_ratio = m_body / m_range if m_range > 0 else 0
        if m_body_ratio < 0.05:
            continue

        # ── Bullish CRT ──────────────────────────────────────────
        sweep_bull = (m_low - c_low) / m_low
        if sweep_bull >= SWEEP_MIN_PCT and c_close > m_low:
            post_daily = daily[daily.index > pd.Timestamp(child_date)]
            if post_daily.empty:
                continue

            current_price   = float(post_daily["Close"].iloc[-1])
            target_hit      = any(post_daily["High"] >= m_high)
            invalid_hit     = any(post_daily["Low"]  <= c_low * 0.999)
            child_above_mid = c_close > m_mid
            pct_to_target   = (m_high - current_price) / current_price * 100

            if target_hit:
                status = "✅ TARGET HIT"
            elif invalid_hit:
                status = "❌ INVALIDATED"
            else:
                status = "🔄 IN PLAY"

            results.append({
                "ticker"        : ticker,
                "direction"     : "Bullish",
                "child_week"    : str(child_date)[:10],
                "mother_range_%": round(m_range_pct * 100, 2),
                "sweep_%"       : round(sweep_bull * 100, 2),
                "child_vs_mid"  : "✅ Above" if child_above_mid else "⚠️ Below",
                "target"        : round(m_high, 2),
                "current_price" : round(current_price, 2),
                "pct_to_target" : round(pct_to_target, 2),
                "status"        : status,
            })

        # ── Bearish CRT ──────────────────────────────────────────
        sweep_bear = (c_high - m_high) / m_high
        if sweep_bear >= SWEEP_MIN_PCT and c_close < m_high:
            post_daily = daily[daily.index > pd.Timestamp(child_date)]
            if post_daily.empty:
                continue

            current_price   = float(post_daily["Close"].iloc[-1])
            target_hit      = any(post_daily["Low"]  <= m_low)
            invalid_hit     = any(post_daily["High"] >= c_high * 1.001)
            child_above_mid = c_close < m_mid
            pct_to_target   = (current_price - m_low) / current_price * 100

            if target_hit:
                status = "✅ TARGET HIT"
            elif invalid_hit:
                status = "❌ INVALIDATED"
            else:
                status = "🔄 IN PLAY"

            results.append({
                "ticker"        : ticker,
                "direction"     : "Bearish",
                "child_week"    : str(child_date)[:10],
                "mother_range_%": round(m_range_pct * 100, 2),
                "sweep_%"       : round(sweep_bear * 100, 2),
                "child_vs_mid"  : "✅ Below" if child_above_mid else "⚠️ Above",
                "target"        : round(m_low, 2),
                "current_price" : round(current_price, 2),
                "pct_to_target" : round(pct_to_target, 2),
                "status"        : status,
            })

    return results


# ════════════════════════════════════════════════════════════════
# 6. DISPLAY FUNCTIONS
# ════════════════════════════════════════════════════════════════
def score_color(pct):
    if pct >= 75: return Fore.GREEN + Style.BRIGHT
    if pct >= 50: return Fore.YELLOW
    return Fore.RED

def bool_icon(val):
    return "✅" if val else "❌"

def print_developing(r):
    sc         = score_color(r["score_pct"])
    dir_icon   = '🟢' if r['direction'] == 'Bullish' else '🔴'
    align_icon = "✅ Ideal" if r["mother_aligned"] else f"⚠️ Same-dir (-{r['direction_penalty']}pt)"
    doji_icon  = f"⚠️ DOJI (-{r['doji_penalty']}pts)" if r["m_is_doji"] else "✅ Strong body"

    penalties = []
    if r["direction_penalty"]: penalties.append(f"dir-{r['direction_penalty']}")
    if r["doji_penalty"]:      penalties.append(f"doji-{r['doji_penalty']}")
    penalty_str = f"  [{', '.join(penalties)}]" if penalties else ""

    print(f"\n{'═'*60}")
    print(f"  {Fore.CYAN}{Style.BRIGHT}{r['ticker']}{Style.RESET_ALL}  "
          f"{dir_icon} {r['direction'].upper()} CRT — {r['current_stage']}")
    print(f"  Day {r['days_in']} of week  |  Mother Range: {r['mother_range_pct']}%")
    print(f"  Mother Body : {r['mother_body_ratio']}% of range  "
          f"| {r['mother_body_dir']}  | {doji_icon}")
    print(f"  Alignment   : {align_icon}")
    print(f"{'─'*60}")
    print(f"  Mother:  Low {r['m_low']}  |  Mid {r['m_mid']}  |  High {r['m_high']}")
    print(f"  Current: {r['current_price']}  |  Sweep to: {r['sweep_extreme']} ({r['sweep_pct']}%)")
    print(f"  Target:  {r['target']}  |  "
          f"Dist: {abs(r['target']-r['current_price']):.2f} "
          f"({abs(r['target']-r['current_price'])/r['current_price']*100:.1f}%)")
    print(f"{'─'*60}")

    # Stage 1
    s1n = int(r['s1_score'].split('/')[0])
    s1c = score_color(s1n / 4 * 100)
    print(f"  {s1c}STAGE 1 — Sweep Phase          [{r['s1_score']}]{Style.RESET_ALL}")
    print(f"    {bool_icon(r['s1_swept'])}  Swept beyond mother extreme          (1pt)")
    print(f"    {bool_icon(r['s1_sweep_pct_ok'])}  Sweep < 2%  ({r['sweep_pct']}%)               (2pts ★)")
    print(f"    {bool_icon(r['s1_vol_elevated'])}  Volume elevated on sweep             (1pt)")

    # Stage 2
    s2n = int(r['s2_score'].split('/')[0])
    s2c = score_color(s2n / 5 * 100)
    print(f"\n  {s2c}STAGE 2 — Reversal Phase        [{r['s2_score']}]{Style.RESET_ALL}")
    print(f"    {bool_icon(r['s2_recovering'])}  Price recovering into mother range   (1pt)")
    print(f"    {bool_icon(r['s2_wicking'])}   Long wicks / price slowing down     (1pt)")
    choch_txt = f"@ {r['s2_choch_level']}" if r['s2_choch_level'] else ""
    print(f"    {bool_icon(r['s2_choch'])}  1H CHOCH formed {choch_txt:<16}   (2pts ★)")
    print(f"    {bool_icon(r['s2_daily_inside'])}  Daily close back inside range        (1pt)")

    # Stage 3
    s3n = int(r['s3_score'].split('/')[0])
    s3c = score_color(s3n / 6 * 100)
    print(f"\n  {s3c}STAGE 3 — Confirmation Phase    [{r['s3_score']}]{Style.RESET_ALL}")
    print(f"    {bool_icon(r['s3_inside_range'])}  Currently inside mother range        (1pt)")
    print(f"    {bool_icon(r['s3_above_mid'])}  {'Above' if r['direction']=='Bullish' else 'Below'} "
          f"midpoint ({r['m_mid']})           (3pts ★★★)")
    print(f"    {bool_icon(r['s3_vol_expanding'])}  Volume expanding on recovery days    (1pt)")
    print(f"    {bool_icon(r['s3_vs_open'])}  Reclaimed mother candle close ({r['m_open']})  (1pt)")

    print(f"\n  {sc}  TOTAL SCORE: {r['total_score']}  ({r['score_pct']:.0f}%){penalty_str}{Style.RESET_ALL}")

    if r["score_pct"] >= 75:
        print(f"{Fore.GREEN}{Style.BRIGHT}  🔥 HIGH CONVICTION — All stages aligning{Style.RESET_ALL}")
    elif r["score_pct"] >= 50:
        print(f"{Fore.YELLOW}  ⚡ DEVELOPING — Watch closely{Style.RESET_ALL}")
    else:
        print(f"{Fore.RED}  👀 EARLY / WEAK — Do not enter yet{Style.RESET_ALL}")


def print_completed_table(completed_list):
    in_play = [r for r in completed_list if r["status"] == "🔄 IN PLAY"]
    if not in_play:
        return
    print(f"\n{'═'*60}")
    print(f"  {Fore.CYAN}{Style.BRIGHT}COMPLETED CRT — STILL IN PLAY{Style.RESET_ALL}")
    print(f"{'═'*60}")
    headers = ["Ticker","Dir","Child Week","M.Range","Sweep",
               "Child vs Mid","Target","Price","To Target","Status"]
    rows = [[r["ticker"], r["direction"], r["child_week"],
             f"{r['mother_range_%']}%", f"{r['sweep_%']}%",
             r["child_vs_mid"], r["target"], r["current_price"],
             f"{r['pct_to_target']}%", r["status"]] for r in in_play]
    if HAS_TABULATE:
        print(tabulate(rows, headers=headers, tablefmt="simple"))
    else:
        print("  " + " | ".join(f"{h:12}" for h in headers))
        for row in rows:
            print("  " + " | ".join(f"{str(x):12}" for x in row))


def print_summary_table(all_developing):
    print(f"\n{'═'*60}")
    print(f"  {Fore.CYAN}{Style.BRIGHT}DEVELOPING CRT — QUICK SUMMARY{Style.RESET_ALL}")
    print(f"{'═'*60}")
    if not all_developing:
        return
    rows = []
    for r in all_developing:
        rows.append([
            r["ticker"], r["direction"],
            r["current_stage"].replace("Stage 3 — ","S3-")
                              .replace("Stage 2 — ","S2-")
                              .replace("Stage 1 — ","S1-"),
            r["s1_score"], r["s2_score"], r["s3_score"],
            r["total_score"], f"{r['score_pct']:.0f}%",
            r["sweep_pct"], f"{r['mother_body_ratio']}%",
            "✅" if r["mother_aligned"] else "⚠️",
            "✅" if r["s3_above_mid"] else "❌",
            "✅" if r["s2_choch"] else "❌",
        ])
    headers = ["Ticker","Dir","Stage","S1","S2","S3",
               "Total","Score%","Sweep%","Body%","Align","vsMid","CHOCH"]
    if HAS_TABULATE:
        print(tabulate(rows, headers=headers, tablefmt="simple"))
    else:
        print("  " + " | ".join(f"{h:8}" for h in headers))
        for row in rows:
            print("  " + " | ".join(f"{str(x):8}" for x in row))


def print_legend():
    print(f"\n{'═'*60}")
    print(f"  {Fore.WHITE}LEGEND — WEIGHTED SCORING (max 15pts):{Style.RESET_ALL}")
    print(f"  S1 (max 4w) : Swept=1 | Sweep<2%=2 | Vol elevated=1")
    print(f"  S2 (max 5w) : Recovering=1 | Wicking=1 | CHOCH=2 | Daily inside=1")
    print(f"  S3 (max 6w) : Inside range=1 | vs MID=3★ | Vol expand=1 | Past close=1")
    print(f"  ★ Midpoint weighted 3x — #1 predictor from 1,540-event backtest")
    print(f"")
    print(f"  HARD DISQUALIFIED (never shown):")
    print(f"    ✗ Mother range outside 2-6%")
    print(f"    ✗ Sweep outside 0.3-4%")
    print(f"")
    print(f"  SOFT PENALTIES (shown with deductions):")
    print(f"    ⚠️  Doji/near-doji mother (<15% body)    = -2pts")
    print(f"    ⚠️  Same-direction mother candle         = -1pt")
    print(f"    → Ideal: Bullish CRT + Red mother | Bearish CRT + Green mother")
    print(f"")
    print(f"  {Fore.GREEN}Score ≥75%  = 🔥 High Conviction — actionable{Style.RESET_ALL}")
    print(f"  {Fore.YELLOW}Score 50-74% = ⚡ Developing — watch list{Style.RESET_ALL}")
    print(f"  {Fore.RED}Score <50%  = 👀 Too early — do not enter{Style.RESET_ALL}")
    print(f"{'═'*60}\n")


# ════════════════════════════════════════════════════════════════
# 7. MAIN RUNNER
# ════════════════════════════════════════════════════════════════
def run_scanner(tickers):
    now = datetime.now()
    print(f"\n{'═'*60}")
    print(f"  {Fore.CYAN}{Style.BRIGHT}CRT WEEKLY SCANNER{Style.RESET_ALL}")
    print(f"  Run time : {now.strftime('%Y-%m-%d %H:%M')}")
    print(f"  Tickers  : {len(tickers)}")
    print(f"  Filters  : Mother 2-6% | Sweep 0.3-4% | Vol >{VOL_ELEVATED_RATIO}")
    print(f"{'═'*60}")

    all_developing = []
    all_completed  = []

    for ticker in tickers:
        try:
            weekly, daily, hourly = fetch_all_data(ticker)
            dev  = scan_developing_crt(ticker, weekly, daily, hourly)
            if dev:
                all_developing.extend(dev)
            comp = scan_completed_crt(ticker, weekly, daily)
            all_completed.extend(comp)
        except Exception as e:
            print(f"  ⚠  {ticker}: {e}")

    all_developing.sort(key=lambda x: x["score_pct"], reverse=True)

    if all_developing:
        print(f"\n{'═'*60}")
        print(f"  {Fore.GREEN}{Style.BRIGHT}DEVELOPING CRT THIS WEEK "
              f"({len(all_developing)} found){Style.RESET_ALL}")
        for r in all_developing:
            print_developing(r)
    else:
        print(f"\n  {Fore.YELLOW}No developing CRT setups found this week.{Style.RESET_ALL}")
        print(f"  Likely cause: Last week's candle was outside 2-6% range")
        print(f"  (common after extreme volatility weeks)")

    print_completed_table(all_completed)
    print_summary_table(all_developing)
    print_legend()

    return all_developing, all_completed


# ════════════════════════════════════════════════════════════════
# 8. ENTRY POINT
# ════════════════════════════════════════════════════════════════
def main():
    parser = argparse.ArgumentParser(
        description="CRT Weekly Scanner — Candle Range Theory (ICT/SMC)",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  python crt_scanner.py
  python crt_scanner.py --file tickers/sp500.txt
  python crt_scanner.py --file tickers/custom.txt
  python crt_scanner.py --tickers AAPL MSFT NVDA AMAT
        """
    )
    parser.add_argument("--file", type=str,
                        help="Path to ticker file (one ticker per line)")
    parser.add_argument("--tickers", nargs="+",
                        help="Space-separated list of tickers")

    args = parser.parse_args()

    if args.tickers:
        tickers = [t.upper() for t in args.tickers]
    elif args.file:
        tickers = load_tickers_from_file(args.file)
        if not tickers:
            print("No tickers loaded from file. Check the file path.")
            sys.exit(1)
    else:
        tickers = TICKERS

    run_scanner(tickers)


if __name__ == "__main__":
    main()
