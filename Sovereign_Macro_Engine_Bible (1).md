# Sovereign Macro Execution Engine — Reference Guide

*A permanent, section-by-section explanation of every number, badge, and decision rule on your dashboard. Every worked example below uses your actual live reading from July 1, 2026, 11:53 AM (screenshot reference).*

---

## Live snapshot used throughout this guide

| Signal | Value | Trend |
|---|---|---|
| Liquidity | $5.79T | -0.88% ▼ |
| 10Y Yield | 4.38% | -4.17% ▼ |
| USD Index (Fed Broad-AFE) | 114.21 | +2.46% ▲ |
| Credit Spread | 2.80% | -0.56% ▼ |
| Regime | **HARD_PIVOT** | |
| Credit Condition | **STABLE** | |
| System Phase | **NORMAL** | |
| Liquidity Momentum | **CONTRACTING (bottoming)** | accel: +0.12% |
| DCA Mode | **LOW / PAUSE** | |
| Risk Status | **ACTIVE** | |

Keep this table handy — every calculation below traces back to it.

---

## SECTION 1 — The Hero Banner (top of page)

This is the single most important read on the whole dashboard — everything else is supporting detail. It shows three things together: **System Phase**, **Regime**, and **DCA Mode**.

**Why it's first:** System Phase is your risk gate. Regime tells you the flavor of the macro backdrop. DCA Mode tells you what to actually do with money. Read left to right in that priority order.

---

## SECTION 2 — Macro Chokepoints (the four raw inputs)

These are the four data streams everything else in the engine is built from. Each card shows two things: the **level** (where it is right now) and the **trend** (how fast it's moving, %, over a defined lookback window).

### 2.1 — Liquidity

**What it is:** Net cash actually available to markets and banks — the Fed's own "print money vs. drain money" scoreboard.

**Exact formula:**
```
net_liquidity = Fed_Balance_Sheet (WALCL) − Reverse_Repo (RRPONTSYD) − Treasury_General_Account (WTREGEN)
```
All three series are unit-corrected to millions of dollars and resampled to weekly (Wednesday-anchored, matching the Fed's own publishing schedule) before subtracting, so nothing is silently averaged across mismatched units or frequencies.

**Trend formula:**
```
liq_trend = 4-week % change in net_liquidity, smoothed with a 2-week rolling average
```
(4 weeks ≈ 1 month, matching the "monthly" lookback used everywhere else, but expressed in weekly periods since liquidity data is inherently weekly.)

**Your live reading:** $5.79T, trending **-0.88%** (▼ red). Liquidity is contracting — the Fed's effective cash injection into the system is shrinking over the last month.

**Card color logic:**
```python
if delta_pct > 0:  accent = green
elif delta_pct < 0: accent = red
else:               accent = gray
```
→ your card is red-accented because -0.88% < 0.

---

### 2.2 — 10Y Yield

**What it is:** DGS10 — the benchmark 10-year Treasury yield. Drives mortgage rates, corporate borrowing costs, and how expensive/cheap it is to discount future risk-asset cash flows.

**Trend formula:** 30-day % change, smoothed with a 5-day rolling average (same standardized window as USD and Credit — see Section 2.5).

**Your live reading:** 4.38%, trending **-4.17%** (▼ red). Yields are falling sharply — a real, fast move, not noise.

---

### 2.3 — USD Index (Fed Broad-AFE)

**What it is:** DTWEXAFEGS — the Fed's free ~26-currency broad dollar index. **This is NOT the ICE DXY** you'd see quoted on CNBC/Bloomberg (that's a 6-currency, EUR-dominated, proprietary paid index with a different base year). The two move in the same direction most of the time, but their raw levels are never comparable — only read this card's *trend*, never compare its level to a remembered "DXY is around 100" anchor.

**Your live reading:** 114.21, trending **+2.46%** (▲ green). The dollar is strengthening broadly.

---

### 2.4 — Credit Spread

**What it is:** BAMLH0A0HYM2 — the ICE BofA High Yield Option-Adjusted Spread. The extra yield junk-rated (BB and below) corporate bonds pay over Treasuries. Widening = markets pricing more default/liquidity risk. Narrowing = risk appetite improving.

**Your live reading:** 2.80%, trending **-0.56%** (▼ red, but this is a *good* red — spreads narrowing means credit markets are calm, not stressed).

> **Important nuance:** on this one card, "down" (red arrow) is actually the healthy direction. The arrow color logic is purely mechanical (red = negative number), it doesn't know that negative is good here — that judgment call is yours to make when reading it.

---

### 2.5 — Why all four cards use consistent math

```python
TREND_WINDOW = 30   # days
SMOOTH_WINDOW = 5    # days

def trend(series, window=30, smooth=5):
    raw = series.pct_change(window)
    smoothed = raw.rolling(smooth).mean()
    return smoothed.iloc[-1]
```
Yield, USD, and Credit all run through this exact same function, so their trend numbers are directly comparable to each other (same lookback, same smoothing). Liquidity uses its own weekly-adjusted version of the same idea (4-week window, 2-week smooth) because its underlying data publishes weekly, not daily — using the daily 30/5 window on weekly data would mean looking back 30 *weeks* instead of 30 days, a completely different timeframe.

---

## SECTION 3 — System State (the four classifier outputs)

This is where the four raw signals get turned into judgments. This section is entirely if/elif/else logic — no new data, just decision rules applied to what you already saw in Section 2.

### 3.1 — Regime

**Purpose:** classifies the *yield + dollar* combination into one of five macro "weather patterns."

**The exact code:**
```python
REGIME_MAGNITUDE_THRESHOLD = 0.02   # 2% — a move must clear this to count as "real"

def classify_regime(y, d, threshold=0.02):
    y_sig = y if abs(y) >= threshold else 0   # yield_trend, zeroed out if too small to matter
    d_sig = d if abs(d) >= threshold else 0   # dxy_trend, same treatment

    if y_sig > 0 and d_sig > 0:
        return "QT"              # yields rising + dollar rising = tightening
    elif y_sig < 0 and d_sig < 0:
        return "SOFT_PIVOT"       # yields falling + dollar falling = classic dovish ease
    elif y_sig < 0 and d_sig > 0:
        return "HARD_PIVOT"       # yields falling + dollar rising = flight-to-quality / fear, NOT stimulus
    else:
        return "TRANSITION"       # anything else, or below the 2% magnitude bar
```

**Why the 2% threshold exists:** without it, a yield move of +0.0001% (pure noise) would get slapped with a full "QT" label, same as a real 2%+ repricing. The threshold means: *your yield AND your dollar trend both have to move by at least 2% before the engine will commit to a directional label.* Anything smaller falls through to `TRANSITION` — "no clear signal yet," which is the honest answer when moves are too small to trust.

**Your live math, step by step:**
1. `yield_trend = -4.17%` → `abs(-0.0417) = 0.0417 ≥ 0.02` → passes the threshold → `y_sig = -4.17%`
2. `dxy_trend = +2.46%` → `abs(0.0246) = 0.0246 ≥ 0.02` → passes the threshold → `d_sig = +2.46%`
3. `y_sig < 0` and `d_sig > 0` → matches the third branch → **`HARD_PIVOT`**

**What HARD_PIVOT actually means here — and the trap:** the name sounds bullish ("the Fed is pivoting dovish!"), but that's only true if yields are falling *because of Fed easing*. Given a strengthening dollar at the same time, this combination more often reflects **flight-to-quality fear** — people bidding up Treasuries (pushing yields down) and the dollar (safe-haven demand) simultaneously, not a policy-driven stimulus signal. Always cross-check HARD_PIVOT against System Phase and Liquidity Momentum before treating it as "risk-on."

---

### 3.2 — Credit Condition

**Purpose:** classifies whether credit markets are calm, deteriorating, or in acute stress — using the credit spread's *trend*, not its level.

**The exact code:**
```python
def credit_state(val):
    if val > 0.15:      # spread trend widened more than 15% over the window
        return "STRESS SPIKE"
    elif val > 0:        # spread trend widened, but under 15%
        return "WIDENING"
    else:                 # spread trend flat or narrowing
        return "STABLE"
```

**Your live math:** `credit_trend_val = -0.56%` → not `> 0.15` → not `> 0` either → falls to the `else` → **`STABLE`**

Credit spreads narrowing is the market's way of saying "we're not worried about defaults right now" — this is the calmest of the three possible states.

---

### 3.3 — System Phase

**Purpose:** the master risk-gate. Combines Liquidity, Dollar, and Credit Condition together — this is the only classifier that looks at all three at once, which is why it sits in the Hero Banner.

**The exact code, in the order it's actually checked (first match wins — this order matters):**
```python
def detect_system_phase(liq, dxy, credit):

    # --- downside / stress states, checked first (highest priority) ---
    if liq < 0 and dxy > 0 and credit == "STRESS SPIKE":
        return "SYSTEM BREAK"        # worst case: contraction + dollar surge + credit panic

    if liq < 0 and dxy > 0 and credit == "WIDENING":
        return "FRACTURE"            # contraction + dollar surge + credit starting to crack

    if credit == "STRESS SPIKE":
        return "CREDIT STRESS"       # credit panicking on its own, even without liq/dxy alignment

    # --- upside / expansion states ---
    if liq > 0 and dxy < 0 and credit == "STABLE":
        return "LIQUIDITY EXPANSION" # textbook easing: liquidity up, dollar down, credit calm

    if liq > 0 and dxy < 0 and credit == "WIDENING":
        return "EXPANSION (credit lagging)"  # expansion signal present, credit hasn't confirmed

    if liq > 0 and dxy > 0:
        return "FRAGILE EXPANSION"   # liquidity AND dollar both rising — an unstable combo,
                                       # often safe-haven flows offsetting real stimulus

    return "NORMAL"                   # everything else falls here
```

**Your live math, checked branch by branch:**
1. `liq_trend = -0.88%` (liq < 0 ✓), `dxy_trend = +2.46%` (dxy > 0 ✓), but `credit_status = STABLE` (not `STRESS SPIKE`) → **branch 1 fails**
2. Same liq/dxy, but `credit_status = STABLE` (not `WIDENING`) → **branch 2 fails**
3. `credit_status = STABLE` (not `STRESS SPIKE`) → **branch 3 fails**
4. `liq_trend < 0`, so `liq > 0` is false → **branch 4 fails**
5. Same reason → **branch 5 fails**
6. `liq_trend < 0`, so `liq > 0` is false → **branch 6 fails**
7. Nothing matched → falls through to the final `return` → **`NORMAL`**

**This is the single most important lesson on the whole dashboard:** liquidity is contracting AND the dollar is strengthening — the *exact* ingredients FRACTURE and SYSTEM BREAK need — but because **Credit Condition is STABLE**, neither stress state fires. **Credit is the confirmation gate.** The engine is telling you: "the setup for stress exists, but the credit market — historically the most reliable canary — hasn't confirmed it yet." That's why Regime says `HARD_PIVOT` (a fear-tinged label) while System Phase calmly says `NORMAL`. They're not contradicting each other — they're measuring different things, and System Phase is the one that gates your risk decisions.

---

### 3.4 — Liquidity Momentum

**Purpose:** answers a question System Phase and Regime can't — is the liquidity trend itself *speeding up or slowing down*? This is the second derivative (acceleration), not just the first derivative (trend).

**The exact code:**
```python
liq_acceleration = liq_impulse.diff().iloc[-1]   # change in liq_trend from the prior period

def liq_momentum_state(trend_val, accel_val):
    if trend_val > 0 and accel_val > 0:
        return "EXPANDING (accelerating)"    # liquidity rising, and rising faster
    elif trend_val > 0 and accel_val <= 0:
        return "EXPANDING (losing steam)"     # liquidity still rising, but slowing down
    elif trend_val <= 0 and accel_val > 0:
        return "CONTRACTING (bottoming)"       # liquidity still falling, but the fall is slowing
    else:
        return "CONTRACTING (accelerating)"     # liquidity falling, and falling faster
```

**Your live math:** `liq_trend = -0.88%` (≤ 0 ✓), `liq_acceleration = +0.12%` (> 0 ✓) → second branch's condition (`trend_val <= 0 and accel_val > 0`) matches → **`CONTRACTING (bottoming)`**

**Plain English:** liquidity is still shrinking month-over-month, but the *pace* of shrinkage is decelerating — the contraction is losing force. This is often (not always) an early tell that a liquidity trough is forming before the trend actually flips positive. It's a leading hint, not a confirmed reversal — don't act on this alone.

---

## SECTION 4 — Execution (what to actually do)

### 4.1 — DCA Mode

**Purpose:** turns liquidity momentum into a concrete "how aggressively should I deploy capital this month" signal — deliberately **relative to the liquidity trend's own history**, not a fixed number, because what counts as "strong" liquidity expansion changes across different macro eras.

**The exact code:**
```python
def compute_dca_mode(current_trend, trend_history, min_history=10):
    if len(trend_history.dropna()) < min_history:
        return "LOW / PAUSE (insufficient history)"   # not enough data to rank confidently

    hist = trend_history.dropna()
    percentile = (hist < current_trend).mean()   # what fraction of ALL history is below today's reading?

    if percentile >= 0.70:
        return "HIGH DCA"      # today's liquidity trend is stronger than 70%+ of its own history
    elif percentile >= 0.40:
        return "MEDIUM DCA"
    else:
        return "LOW / PAUSE"   # today's reading is in the weak bottom 40% of its own history
```

**Your live reading: `LOW / PAUSE`** — your current `liq_trend` of -0.88% ranks in the bottom 40% of its own historical distribution. Since the number is negative, this is expected — negative liquidity trends will nearly always rank low.

**Why percentile-based, not a fixed number like "> 0.05":** a fixed threshold treats a 5% swing the same whether liquidity has historically been calm (where 5% is enormous) or volatile (where 5% is unremarkable). Ranking against history self-corrects for that — "HIGH DCA" always means "genuinely strong for this series," regardless of era.

---

### 4.2 — Risk Status

**Purpose:** a simple binary gate derived directly from System Phase — this is the "should I be defensive right now" flag.

**The exact code:**
```python
STRESS_PHASES = {"SYSTEM BREAK", "FRACTURE", "CREDIT STRESS"}
risk_status = "RISK OFF" if system_phase in STRESS_PHASES else "ACTIVE"
```

Any of the three stress-family phases trips this to `RISK OFF`. Everything else — `NORMAL`, `LIQUIDITY EXPANSION`, `EXPANSION (credit lagging)`, `FRAGILE EXPANSION` — reads as `ACTIVE`.

**Your live reading:** `system_phase = NORMAL`, which is not in `STRESS_PHASES` → **`ACTIVE`**.

---

## SECTION 5 — How to read the whole board together (the actual skill)

The engine is built so that reading any single card in isolation is a mistake. Here's the discipline, worked through your live example:

1. **Start with System Phase** (the Hero Banner) — this is your risk gate. Today: `NORMAL`, so `Risk Status = ACTIVE`. Nothing here says "get defensive."
2. **Check Regime for flavor** — `HARD_PIVOT` tells you *why* things are moving (yields down, dollar up), and flags that this specific flavor is fear-driven rather than stimulus-driven. This is context, not an action signal by itself.
3. **Check Liquidity Momentum for the leading edge** — `CONTRACTING (bottoming)` says the liquidity drag may be easing off, worth monitoring for a turn.
4. **Check Credit Condition as your confirmation layer** — `STABLE` is the reason System Phase hasn't escalated despite the liq/dollar setup looking stress-adjacent. If Credit Condition flips to `WIDENING` next month while liq stays negative and dxy stays positive, System Phase will immediately jump to `FRACTURE` — watch this transition closely.
5. **Let DCA Mode drive the actual money decision** — `LOW / PAUSE` today, because liquidity trend ranks weak against its own history. This is your execution signal, not the narrative signals above it.

**The one-sentence version to remember:** *Regime tells you the story, Credit Condition tells you whether to believe it, System Phase tells you whether to worry, and DCA Mode tells you what to actually do with your money.*

---

## Quick-reference: every threshold in the engine, in one place

| Rule | Threshold | Where |
|---|---|---|
| Regime magnitude filter | ±2% minimum move on both yield_trend and dxy_trend | `classify_regime()` |
| Credit stress spike | credit_trend_val > 15% | `credit_state()` |
| Credit widening | 0% < credit_trend_val ≤ 15% | `credit_state()` |
| Early pivot override | liq_trend > 1% AND yield_trend < 0 AND regime == TRANSITION | Regime post-processing |
| DCA high tier | liq_trend in top 30% of its own history (percentile ≥ 0.70) | `compute_dca_mode()` |
| DCA medium tier | liq_trend in 40th–70th percentile of its own history | `compute_dca_mode()` |
| Liquidity trend window | 4 weeks, 2-week smoothing | `LIQ_WINDOW_WEEKS`, `LIQ_SMOOTH_WEEKS` |
| Yield/USD/Credit trend window | 30 days, 5-day smoothing | `TREND_WINDOW`, `SMOOTH_WINDOW` |

Bookmark this table — every time a badge changes color, one of these rules is what moved it.

---
---

# PART 2 — Trigger Overlay & The Sovereign 15-Year Engine

*Everything below was added after Part 1. It sits on top of everything above — it doesn't replace it. Since I can't run the live app from here, every number in this Part uses an **illustrative worked example** (clearly labeled as such) rather than a real screenshot — the formulas are exact, only the input numbers are constructed for teaching. Plug in your own live readings and the same logic applies unchanged.*

## Illustrative example used throughout Part 2

| Signal | Example value |
|---|---|
| liq_trend (short, ~1mo) | -0.88% |
| liq_acceleration | +0.12% |
| yield_trend | -4.17% |
| dxy_trend | +2.46% |
| credit_trend_val | -0.56% (credit_status = STABLE) |
| system_phase | NORMAL |
| Your reported current allocation (sidebar slider) | 55% |
| liq_pct / credit_pct / yield_pct / dxy_pct (constructed percentiles, explained below) | 0.35 / 0.60 / 0.20 / 0.75 |

---

## SECTION 6 — Trigger Signals (the heuristic second lens)

This section sits below "Execution" on your dashboard. It is explicitly labeled **unvalidated, not backtested** in the UI itself — that label is load-bearing, not decoration. Every trigger here is a *new boolean combination of signals you already have*, not a new data source.

### 6.1 — Distribution Trap

**The exact code:**
```python
def distribution_trap(dxy_trend_val, credit_trend_val_, liq_trend_val):
    return dxy_trend_val > 0 and credit_trend_val_ <= 0 and liq_trend_val < 0
```

**Reading it:** dollar strengthening (`dxy_trend > 0`), credit spread trend flat-or-narrowing (`credit_trend_val ≤ 0`), AND liquidity contracting (`liq_trend < 0`) — all three at once.

**Worked check:** `dxy_trend = +2.46%` (>0 ✓), `credit_trend_val = -0.56%` (≤0 ✓), `liq_trend = -0.88%` (<0 ✓) → all three true → **`distribution_trap = True`**

**Why this deliberately disagrees with System Phase:** `detect_system_phase()` (Section 3.3, Part 1) requires credit to be `WIDENING` or `STRESS SPIKE` before it will call anything a stress state — it demands *confirmation*. This heuristic takes the opposite philosophy: it treats calm credit *combined with* the other two red flags as itself suspicious ("why isn't credit reacting yet?"), not reassuring. Both readings are valid ways to think about the same data — the dashboard shows you both, on purpose, rather than picking one and hiding the disagreement.

### 6.2 — Forced Liquidation Signal

**The exact code:**
```python
FORCED_LIQUIDATION_CREDIT_THRESHOLD = 0.10  # 10%, lower than the 15% STRESS SPIKE bar

def forced_liquidation_signal(credit_trend_val_, liq_acceleration_val, threshold=0.10):
    return credit_trend_val_ > threshold and liq_acceleration_val < 0
```

**Reading it:** credit spreads widening past 10% (a lower, earlier-warning bar than the 15% that triggers official `STRESS SPIKE`) AND liquidity contraction still accelerating (getting worse, not bottoming).

**Worked check:** `credit_trend_val = -0.56%` → not `> 0.10` → **`forced_liquidation_signal = False`**

### 6.3 — Front-Run Pivot Signal

**The exact code:**
```python
def front_run_pivot(liq_trend_val, liq_acceleration_val, yield_trend_val):
    return liq_trend_val > 0 and liq_acceleration_val > 0 and yield_trend_val <= 0
```

**Reading it:** liquidity expanding AND that expansion accelerating, while yields aren't rising (no fear-driven Treasury selloff working against it).

**Worked check:** `liq_trend = -0.88%` → not `> 0` → **`front_run_pivot = False`**

### 6.4 — Resolved Stance (priority resolution)

**The exact code:**
```python
def resolve_action():
    if is_forced_liquidation:
        return ("FORCED LIQUIDATION", "AGGRESSIVE BUY SEQUENCE", "...")
    if is_distribution_trap:
        return ("DISTRIBUTION TRAP", "HOLD -- DO NOT ADD RISK", "...")
    if is_pivot_signal:
        return ("PIVOT SIGNAL", "ACCUMULATE RISK EARLY", "...")
    playbook = execution_playbook(system_phase)
    return ("SYSTEM PHASE", playbook["stance"], "No override trigger fired...")
```

**Worked check, in order:** `is_forced_liquidation = False` → skip. `is_distribution_trap = True` → **first match wins** → **Resolved Trigger = `DISTRIBUTION TRAP`, Stance = `HOLD — DO NOT ADD RISK`**. (`is_pivot_signal` never even gets checked, because Distribution Trap already matched first.)

**The one rule to remember:** this is strict priority order, top to bottom, first `True` wins. If both Forced Liquidation and Distribution Trap were true simultaneously, Forced Liquidation always wins the headline — it's ranked as the more urgent signal.

---

## SECTION 7 — The Sovereign 15-Year Engine

This is the newest layer, sitting at the bottom of your dashboard. Read the disclaimer built into its own UI caption first: *"a policy framework built from the signals above, not a new data source and not an autonomous trading system... treat the output as a consistent readout of YOUR policy, not a discovered truth."* Every threshold in this section is an editable constant in the code — none of it is a proven law of markets.

### 7.1 — Multi-Horizon Liquidity

**The exact code:**
```python
def liquidity_trend_series(net_liq_series, window_weeks, smooth_weeks):
    raw = net_liq_series.pct_change(window_weeks)
    return raw.rolling(smooth_weeks).mean()

liq_short_series  = liquidity_trend_series(net_liquidity, 4, 2)    # ~1 month
liq_medium_series = liquidity_trend_series(net_liquidity, 13, 4)   # ~1 quarter
liq_long_series   = liquidity_trend_series(net_liquidity, 52, 8)   # ~1 year
```

**What it's for:** the original `liq_trend` badge only ever showed you one lens (~1 month). This adds two more, so you can see whether a short-term wobble is part of a longer contraction or a longer expansion. A short-horizon dip inside a long-horizon uptrend reads very differently than a short-horizon dip inside a long-horizon downtrend, even though the "short" number looks identical in both cases.

**Honest data-window caveat:** "15-year" describes the *holding horizon you're planning for* — it does **not** mean the engine has 15 years of liquidity history to compute against. FRED's `WALCL`/`RRPONTSYD`/`WTREGEN` series only go back to `start_date = "2015-01-01"` in this script, so as of today the "long" (~1 year) window is well-supported, but anything claiming a true 15-year lookback would need data this pipeline doesn't have yet.

---

### 7.2 — Macro Composite Score (-100 to +100)

This is the single number the rest of Section 7 is built from. It compresses four already-familiar signals into one score by first ranking each against its own history, then combining with fixed weights.

**Step 1 — percentile rank each signal against its own history:**
```python
def percentile_rank(current, hist_series, min_history=10):
    hist = hist_series.dropna()
    if len(hist) < min_history:
        return 0.5  # not enough history -> treat as neutral
    return (hist < current).mean()
```
This is the exact same technique `compute_dca_mode()` uses (Part 1, Section 4.1) — "what fraction of this series' own past is below today's reading?"

**Worked example (constructed percentiles for this walkthrough):**
- `liq_pct = 0.35` → today's medium-horizon liquidity trend is weaker than 65% of its own history
- `credit_pct = 0.60` → today's credit trend ranks moderately high (i.e. closer to widening) versus its own history, even though the absolute level is still `STABLE`
- `yield_pct = 0.20` → today's yield trend (falling hard, -4.17%) ranks in the bottom 20% of its own history — a genuinely large move
- `dxy_pct = 0.75` → today's dollar strengthening ranks in the top 75% of its own history — also a large move

**Step 2 — convert each percentile to a signed -1..+1 contribution:**
```python
score_components = {
    "liquidity": (liq_pct - 0.5) * 2,
    "credit":   -(credit_pct - 0.5) * 2,
    "yield":    -(yield_pct - 0.5) * 2,
    "dollar":   -(dxy_pct - 0.5) * 2,
}
```
Notice the sign flips on credit/yield/dollar but not liquidity: high liquidity percentile is good for risk (positive), but high credit/yield/dollar percentile means those are running hot in the *tightening* direction, so they get negated.

**Worked math:**
- liquidity: `(0.35 - 0.5) × 2 = -0.30`
- credit: `-(0.60 - 0.5) × 2 = -0.20`
- yield: `-(0.20 - 0.5) × 2 = +0.60`
- dollar: `-(0.75 - 0.5) × 2 = -0.50`

**Step 3 — weight and combine:**
```python
MACRO_SCORE_WEIGHTS = {"liquidity": 0.40, "credit": 0.30, "yield": 0.15, "dollar": 0.15}

def compute_macro_score(components, weights):
    return sum(components[k] * weights[k] for k in weights) * 100
```
**Worked math:** `(-0.30×0.40) + (-0.20×0.30) + (0.60×0.15) + (-0.50×0.15)`
`= -0.12 + -0.06 + 0.09 + -0.075 = -0.165`
`macro_score = -0.165 × 100 = -16.5`

**Why liquidity and credit are weighted heaviest (40%/30% vs 15%/15% each):** this reflects the earlier discussion in this conversation that liquidity direction and credit stress are the more reliable *leading* signals, while yield/dollar pivots are noisier to trust for timing (the HARD_PIVOT-vs-flight-to-quality ambiguity from earlier is a good example of that noise). This is a stated assumption baked into `MACRO_SCORE_WEIGHTS` — change the dict if you weigh things differently.

---

### 7.3 — Confidence

**The exact code:**
```python
def compute_confidence(components, score):
    if score == 0:
        return "LOW"
    agree = sum(1 for c in components.values() if (c > 0) == (score > 0))
    frac = agree / len(components)
    if frac >= 0.75:
        return "HIGH"
    elif frac >= 0.5:
        return "MEDIUM"
    return "LOW"
```

**Worked check:** `macro_score = -16.5` (negative). Check each component's sign against the score's sign:
- liquidity `-0.30` → negative, **agrees**
- credit `-0.20` → negative, **agrees**
- yield `+0.60` → positive, **disagrees**
- dollar `-0.50` → negative, **agrees**

`agree = 3`, `frac = 3/4 = 0.75` → `0.75 ≥ 0.75` → **`confidence = HIGH`**

**Plain English:** even though the yield signal is pointing the *opposite* direction from the overall score, 3 of 4 signals agree, which clears the 75% bar for HIGH confidence. If it had been 2 of 4 (50%), you'd land on MEDIUM instead — the score would still compute the same way, but you'd trust it less.

---

### 7.4 — Macro Phase (v2)

**The exact code:**
```python
def map_score_to_phase(score):
    if score >= 40:
        return "STRONG EXPANSION"
    elif score >= 15:
        return "EXPANSION"
    elif score > -15:
        return "NEUTRAL"
    elif score > -40:
        return "CONTRACTION"
    return "STRONG CONTRACTION"
```

**Worked check:** `macro_score = -16.5`. Walk the branches top to bottom: `≥ 40`? No. `≥ 15`? No. `> -15`? Is `-16.5 > -15`? **No** (it's more negative). `> -40`? Is `-16.5 > -40`? **Yes** → matches this branch → **`macro_phase_v2 = "CONTRACTION"`**

Note this is a *different* phase label from `system_phase` (Part 1, Section 3.3) — `system_phase` outputs things like `NORMAL`/`FRACTURE`; `macro_phase_v2` outputs a five-tier scale from `STRONG CONTRACTION` to `STRONG EXPANSION`. They're two independent classifiers looking at overlapping but not identical inputs — don't expect them to always say the same thing.

---

### 7.5 — Target Allocation Policy

**The exact code (an editable lookup table, not a discovered rule):**
```python
ALLOCATION_POLICY = {
    ("STRONG EXPANSION", "HIGH"): 0.90, ("STRONG EXPANSION", "MEDIUM"): 0.80, ("STRONG EXPANSION", "LOW"): 0.70,
    ("EXPANSION", "HIGH"): 0.80,        ("EXPANSION", "MEDIUM"): 0.70,        ("EXPANSION", "LOW"): 0.60,
    ("NEUTRAL", "HIGH"): 0.60,          ("NEUTRAL", "MEDIUM"): 0.55,          ("NEUTRAL", "LOW"): 0.50,
    ("CONTRACTION", "HIGH"): 0.40,      ("CONTRACTION", "MEDIUM"): 0.45,      ("CONTRACTION", "LOW"): 0.50,
    ("STRONG CONTRACTION", "HIGH"): 0.20, ("STRONG CONTRACTION", "MEDIUM"): 0.30, ("STRONG CONTRACTION", "LOW"): 0.40,
}

def capital_allocation_target(phase, conf):
    return ALLOCATION_POLICY.get((phase, conf), 0.55)
```

**Worked check:** key = `("CONTRACTION", "HIGH")` → looked up directly → **`allocation_target = 0.40`** (40%)

**How to read the table's shape:** for a given phase, HIGH confidence pushes the target *further* in that phase's direction (more aggressive in expansions, more defensive in contractions) — except notice `CONTRACTION` and `STRONG CONTRACTION` actually have their HIGH-confidence number *lower* than MEDIUM/LOW. That's intentional: when the model is confidently bearish, the policy leans more defensive, not less — LOW confidence in a contraction phase gets pulled back toward the neutral 50% because you shouldn't fully trust a low-confidence bearish read either. This asymmetry is a design choice, not an error — but it's exactly the kind of thing you should feel free to redesign in `ALLOCATION_POLICY` if you disagree with it.

---

### 7.6 — Liquidity Kill-Switch

**The exact code:**
```python
def liquidity_kill_switch(liq_short, liq_accel, threshold=-0.03):
    return liq_short < threshold and liq_accel < 0
```

**Worked check:** `liq_short = liq_multi["short"] ≈ -0.88% = -0.0088`. Is `-0.0088 < -0.03`? **No** (-0.88% is a smaller-magnitude contraction than the -3% bar) → **`kill_switch_active = False`**, regardless of what `liq_acceleration` is (the `and` short-circuits).

**Why -3%, not just "any negative number":** without a real threshold, the kill-switch would trip on almost any month with mild contraction, making it useless as a circuit breaker. -3% on the *short* (1-month) horizon is meant to represent a genuinely sharp, not routine, liquidity drain.

---

### 7.7 — Discipline Tracker

**The exact code:**
```python
def discipline_check(target, actual):
    deviation = abs(target - actual)
    if deviation > 0.20:
        return "VIOLATION"
    elif deviation > 0.10:
        return "DRIFT"
    return "ALIGNED"
```

**Worked check:** `target = allocation_target = 0.40`, `actual = current_alloc = 0.55` (your sidebar slider input in this example). `deviation = |0.40 - 0.55| = 0.15`. Is `0.15 > 0.20`? No. Is `0.15 > 0.10`? **Yes** → **`discipline_status = "DRIFT"`**

**Critical honesty note:** `current_alloc` comes entirely from a manual slider you set in the sidebar — the engine has **no independent way to verify your real brokerage position**. This tracker is only ever as honest as the number you type in. It's a discipline *mirror*, not a discipline *auditor*.

---

### 7.8 — System Governor

**The exact code:**
```python
def system_governor(target_alloc, kill_switch, discipline):
    if kill_switch:
        return target_alloc * 0.5, "Liquidity kill-switch active: suggested target halved..."
    if discipline == "VIOLATION":
        return target_alloc * 0.8, "Large deviation from your own reported allocation: suggested target trimmed..."
    return target_alloc, "No override applied -- policy target stands as-is."
```

**Worked check:** `kill_switch_active = False` → skip first branch. `discipline_status = "DRIFT"` — note the check is specifically for the string `"VIOLATION"`, and `"DRIFT"` is **not equal to** `"VIOLATION"` → skip second branch. → falls through to the final return → **`governed_target = target_alloc = 0.40` (unchanged), note = "No override applied"**

**The rule worth memorizing:** the governor only intervenes at the two most severe states — an active kill-switch, or a full `VIOLATION` (>20% deviation). `DRIFT` (10–20% deviation) is visible to you as a warning badge, but does **not** by itself trigger an automatic haircut. That's a deliberate design choice: minor drift is normal and shouldn't trigger constant automatic adjustments, only large deviations do.

---

### 7.9 — Rebalance Suggestion

**The exact code:**
```python
def rebalance_suggestion(current, target, band=0.05):
    diff = target - current
    if abs(diff) < band:
        return "NO ACTION -- within tolerance band"
    direction = "INCREASING" if diff > 0 else "REDUCING"
    return f"Consider {direction} exposure by ~{abs(diff)*100:.1f}%"
```

**Worked check:** `current = 0.55`, `target = governed_target = 0.40`. `diff = 0.40 - 0.55 = -0.15`. Is `|-0.15| < 0.05`? No → `diff > 0`? No (`-0.15` is negative) → `direction = "REDUCING"` → **`rebalance_note = "Consider REDUCING exposure by ~15.0%"`**

---

### 7.10 — Dynamic Portfolio Split (Tilted Grid)

This is the one piece of Section 7 that connects back to your *actual* uploaded allocation sheet, not a generic placeholder.

**The exact code:**
```python
BASE_GRID = {
    "INFRA": 0.15, "ENERGY_COMMODITY": 0.23, "AI_SEMIS": 0.10,
    "EM": 0.07, "BTC": 0.25, "GOLD": 0.10, "CASH": 0.10,
}
HIGH_BETA_SLEEVES = {"AI_SEMIS", "BTC", "EM"}
DEFENSIVE_SLEEVES = {"GOLD", "CASH"}
MAX_TILT = 0.30  # hard cap

def tilt_grid(base_grid, score, high_beta, defensive, max_tilt=0.30):
    tilt_factor = (max(-100, min(100, score)) / 100) * max_tilt
    tilted = {}
    for name, weight in base_grid.items():
        if name in high_beta:
            tilted[name] = weight * (1 + tilt_factor)
        elif name in defensive:
            tilted[name] = weight * (1 - tilt_factor)
        else:
            tilted[name] = weight
    total = sum(tilted.values())
    return {k: v / total for k, v in tilted.items()}
```

**Worked math, step by step:**
1. Clamp `macro_score` to [-100, 100] (already within range at -16.5), then scale: `tilt_factor = (-16.5 / 100) × 0.30 = -0.0495` (≈ -4.95%)
2. High-beta sleeves (AI_SEMIS, BTC, EM) get multiplied by `(1 + tilt_factor) = 0.9505` — i.e. **shrunk** ~4.95%, because the score is negative:
   - AI_SEMIS: `0.10 × 0.9505 = 0.09505`
   - BTC: `0.25 × 0.9505 = 0.23763`
   - EM: `0.07 × 0.9505 = 0.06654`
3. Defensive sleeves (GOLD, CASH) get multiplied by `(1 - tilt_factor) = 1.0495` — i.e. **grown** ~4.95%:
   - GOLD: `0.10 × 1.0495 = 0.10495`
   - CASH: `0.10 × 1.0495 = 0.10495`
4. INFRA (0.15) and ENERGY_COMMODITY (0.23) are untouched — neither high-beta nor defensive in this scheme.
5. Sum everything: `0.09505 + 0.23763 + 0.06654 + 0.10495 + 0.10495 + 0.15 + 0.23 = 0.98912`
6. Renormalize each by dividing by that total so the whole grid sums back to exactly 100%:

| Sleeve | Base | Tilted (this example) |
|---|---|---|
| INFRA | 15.0% | 15.16% |
| ENERGY_COMMODITY | 23.0% | 23.25% |
| AI_SEMIS | 10.0% | 9.61% |
| EM | 7.0% | 6.73% |
| BTC | 25.0% | 24.03% |
| GOLD | 10.0% | 10.61% |
| CASH | 10.0% | 10.61% |

**Why capped at ±30% (`MAX_TILT`):** without a cap, an extreme score (±100) could theoretically zero out or double a sleeve. The cap guarantees the *macro tilt* can only ever nudge your base policy, never override it entirely — your original allocation grid stays the dominant structure.

---

### 7.11 — Alerts Panel

**The exact code:**
```python
def generate_alerts(sys_phase, trigger, kill_switch, discipline):
    alerts = []
    if kill_switch:
        alerts.append("🚨 Liquidity kill-switch active...")
    if sys_phase in STRESS_PHASES:
        alerts.append(f"⚠️ System Phase is {sys_phase}...")
    if trigger == "FORCED LIQUIDATION":
        alerts.append("⚠️ Forced Liquidation heuristic fired...")
    if discipline == "VIOLATION":
        alerts.append("🧭 Your reported current allocation deviates >20%...")
    if not alerts:
        alerts.append("✅ No active alerts.")
    return alerts
```

**Worked check, condition by condition:**
- `kill_switch_active = False` → no alert
- `system_phase = "NORMAL"`, and `STRESS_PHASES = {"SYSTEM BREAK", "FRACTURE", "CREDIT STRESS"}` — `"NORMAL"` is not in that set → no alert
- `resolved_trigger = "DISTRIBUTION TRAP"`, not `"FORCED LIQUIDATION"` → no alert
- `discipline_status = "DRIFT"`, not `"VIOLATION"` → no alert
- None of the four fired → falls to the final `if not alerts` → **`alerts = ["✅ No active alerts."]`**

**The subtle point worth remembering:** Distribution Trap being `True` (Section 6.1) does **not**, by itself, generate an entry in this Alerts list — it only shows up in the Trigger Signals badges and the Resolved Stance banner higher up the page. Alerts is deliberately a *narrower*, higher-severity list than the full set of things the dashboard tracks. Don't rely on Alerts alone to catch everything; it's the "only the loudest stuff" view.

---

### 7.12 — History Log & Drawdown Protection

**The exact code:**
```python
HISTORY_FILE = "sovereign_engine_history.csv"

def log_snapshot(as_of, score, phase, target, actual, portfolio_value=None):
    row = pd.DataFrame([{...}])
    write_header = not os.path.exists(HISTORY_FILE)
    row.to_csv(HISTORY_FILE, mode="a", header=write_header, index=False)

def drawdown_protection(value_series, threshold=0.20):
    vals = value_series.dropna()
    if vals.empty:
        return "NO DATA", 0.0
    peak = vals.cummax()
    dd = (vals - peak) / peak
    current_dd = dd.iloc[-1]
    status = "DRAWDOWN PROTECTION ACTIVE" if current_dd <= -threshold else "NORMAL"
    return status, current_dd
```

**How it works:** clicking "Log today's snapshot" in the sidebar appends one row (date, macro score, phase, target/current allocation, and optionally your typed-in portfolio value) to a local CSV. `drawdown_protection()` then looks at your logged portfolio-value history, tracks the running peak (`cummax()`), and computes how far below that peak you currently sit. If you're down 20% or more from your own logged peak, the status flips to `DRAWDOWN PROTECTION ACTIVE`.

**The most important caveat in this entire document:** this CSV lives on the app's local filesystem. **On free/ephemeral hosting (including Streamlit Community Cloud's free tier), this file can be wiped on every redeploy or app restart.** This is explicitly *not* durable 15-year storage as currently built — it's a working mechanism you can test today, but real long-horizon tracking needs a real backing store (a Google Sheet via `gspread`, Airtable, or a hosted database on persistent storage). Don't trust this log to survive a year untouched until you've swapped the backend.

---

## Quick-reference: Part 2 thresholds, in one place

| Rule | Threshold | Where |
|---|---|---|
| Forced liquidation credit bar | credit_trend_val > 10% (lower than the 15% STRESS SPIKE bar) | `forced_liquidation_signal()` |
| Trigger priority order | forced liquidation → distribution trap → pivot signal → system phase | `resolve_action()` |
| Macro score weights | liquidity 40% / credit 30% / yield 15% / dollar 15% | `MACRO_SCORE_WEIGHTS` |
| Confidence HIGH | ≥75% of the 4 components agree in sign with the score | `compute_confidence()` |
| Confidence MEDIUM | 50–74% agreement | `compute_confidence()` |
| Macro phase bands | ≥40 Strong Exp / ≥15 Exp / >-15 Neutral / >-40 Contraction / else Strong Contraction | `map_score_to_phase()` |
| Liquidity kill-switch | short-horizon liq_trend < -3% AND liq_acceleration < 0 | `liquidity_kill_switch()` |
| Discipline DRIFT | 10–20% deviation between target and your reported allocation | `discipline_check()` |
| Discipline VIOLATION | >20% deviation | `discipline_check()` |
| Governor override | only fires on kill-switch (halve target) or VIOLATION (×0.8 target) — DRIFT alone does not trigger it | `system_governor()` |
| Rebalance tolerance band | <5% deviation = no action suggested | `rebalance_suggestion()` |
| Portfolio tilt cap | ±30% relative to base sleeve weight, regardless of how extreme the score gets | `MAX_TILT` |
| Drawdown protection trigger | ≥20% decline from your logged portfolio-value peak | `drawdown_protection()` |

Same rule as Part 1: every time a Section 7 badge changes, one of these rows is what moved it. If a number ever looks unfamiliar months from now, trace it back through this table before assuming something's broken.

