# Institutional Crypto Futures Signal Engine — Requirements Specification

## Document Version: 1.0
## Date: 2026-05-23
## Target: TradingView Pine Script v5
## Markets: Binance USDT-Perpetual Futures (BTC, ETH, SOL, etc.)
## Timeframes: 1H (primary), 15M (secondary)

---

## Design Decisions (Resolved Ambiguities)

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Pivot lookback | **2L + pivot + 2R** (`ta.pivothigh(high,2,2)`) | More reactive on 1H/15M crypto; 5L/5R too slow for intraday futures |
| 2 | Liquidity sweep | **Wick breaches level, close recovers above/below** | Classic spring/upthrust pattern; a close-through is a breakdown, not a sweep |
| 3 | Impulse threshold | **Impulse move body ≥ 1.0× ATR(14)** AND causes BOS | Filters weak OBs while not being so strict we miss valid setups |
| 4 | FVG inside OB | **Narrow entry zone to FVG portion only** (premium zone) | Tighter entry = better RR geometry; full OB remains visual reference |
| 5 | Daily reset | **00:00 UTC** | Aligns with Binance daily candle close and funding timestamp |
| 6 | TP3 minimum | **TP3 must be beyond TP2**; if nearest liquidity < TP2, skip TP3, close 100% at TP2 | No point running a portion toward a target already captured |
| 7 | HTF filter data | **Confirmed (closed) HTF bar only**, `barmerge.lookahead_off` | Anti-repaint; yes, first 3 bars of 4H use previous closed 4H values — acceptable |
| 8 | Entry execution | **Indicator**: arrow on confirmation candle close. **Strategy**: entry at open[0] of next bar | Reconciles visual clarity with realistic fill simulation |
| 9 | Multi-TF OBs | **Current timeframe OBs only** | Simplicity; HTF OBs add complexity without proportional edge on 1H/15M |
| 10 | Leverage cap | **Max 20x input** (default 10x); signal blocked if required leverage exceeds cap | Prevents suicidal position sizing on volatile pairs |

---

## 1. MATHEMATICAL FORMULAS

### 1.1 Swing Point Detection (5-Bar Fractal)

```
Swing High (SH):
  SH[i] = true IF high[i] > high[i-1] AND high[i] > high[i-2]
                AND high[i] > high[i+1] AND high[i] > high[i+2]
  Confirmed at bar [i+2] (2-bar right confirmation)
  Final confirmation: barstate.isconfirmed == true on bar [i+2]

Swing Low (SL):
  SL[i] = true IF low[i] < low[i-1] AND low[i] < low[i-2]
               AND low[i] < low[i+1] AND low[i] < low[i+2]
  Confirmed at bar [i+2]
```

Implementation: `ta.pivothigh(high, 2, 2)` and `ta.pivotlow(low, 2, 2)`
Offset: pivot values appear at `bar_index - 2` (the actual pivot bar)

### 1.2 Market Structure Classification

```
Let SH_n = most recent confirmed swing high
Let SL_n = most recent confirmed swing low
Let SH_{n-1} = previous swing high
Let SL_{n-1} = previous swing low

BULLISH STRUCTURE:
  SH_n > SH_{n-1}  (Higher High)  AND  SL_n > SL_{n-1}  (Higher Low)

BEARISH STRUCTURE:
  SH_n < SH_{n-1}  (Lower High)  AND  SL_n < SL_{n-1}  (Lower Low)

RANGING/CHOP:
  Any mixed combination (HH+LL, LH+HL, or insufficient data)
```

### 1.3 Break of Structure (BOS)

```
BULLISH BOS:
  close[0] > SH_{n-1}  WHERE current structure was BEARISH
  (Price closes above the most recent swing high in bearish context)

BEARISH BOS:
  close[0] < SL_{n-1}  WHERE current structure was BULLISH
  (Price closes below the most recent swing low in bullish context)
```

### 1.4 Change of Character (CHoCH)

```
CHoCH = the FIRST counter-trend BOS after a sustained trend
  Bullish CHoCH: First close above swing high after ≥2 consecutive LH+LL
  Bearish CHoCH: First close below swing low after ≥2 consecutive HH+HL
  
  Subsequent breaks in same direction = BOS (not CHoCH)
```

### 1.5 Equal Highs / Equal Lows

```
EQH: |SH_a - SH_b| / SH_a ≤ 0.0015  (0.15% tolerance)
  WHERE SH_a and SH_b are any two swing highs within last 50 bars

EQL: |SL_a - SL_b| / SL_a ≤ 0.0015  (0.15% tolerance)
  WHERE SL_a and SL_b are any two swing lows within last 50 bars
```

### 1.6 Order Block Detection

```
BULLISH ORDER BLOCK (BOB):
  Let BOS_bar = bar where bullish BOS occurred
  Let impulse_bars = bars from last swing low to BOS_bar
  
  Conditions:
    1. Bullish BOS confirmed (close > previous SH)
    2. Impulse body: |close[BOS_bar] - open[BOS_bar]| ≥ 1.0 × ATR(14)
    3. OB_candle = last BEARISH candle (close < open) before the impulse
    4. OB_top = open[OB_candle]
    5. OB_bottom = low[OB_candle]
    
BEARISH ORDER BLOCK (BEOB):
    1. Bearish BOS confirmed (close < previous SL)
    2. Impulse body: |open[BOS_bar] - close[BOS_bar]| ≥ 1.0 × ATR(14)
    3. OB_candle = last BULLISH candle (close > open) before the impulse
    4. OB_top = high[OB_candle]
    5. OB_bottom = open[OB_candle]
```

### 1.7 Fair Value Gap (FVG)

```
BULLISH FVG (3-candle sequence):
  Gap exists IF low[0] > high[2]  (current candle low above 2-bars-ago high)
  FVG_top = low[0]
  FVG_bottom = high[2]
  Valid IF (FVG_top - FVG_bottom) > 0

BEARISH FVG:
  Gap exists IF high[0] < low[2]  (current candle high below 2-bars-ago low)
  FVG_top = low[2]
  FVG_bottom = high[0]
```

### 1.8 RSI Calculation

```
RSI = 100 - (100 / (1 + RS))
RS = Average Gain(14) / Average Loss(14)
Using Wilder's smoothing (standard ta.rsi())
```

### 1.9 EMA Calculations

```
EMA(period) = close × (2 / (period + 1)) + EMA_prev × (1 - 2 / (period + 1))

Current TF:  EMA_fast = EMA(21),  EMA_slow = EMA(50)
HTF Filter:  
  If chart = 15M → HTF = 1H:  EMA(21) vs EMA(50) on 1H
  If chart = 1H  → HTF = 4H:  EMA(21) vs EMA(50) on 4H
```

### 1.10 ATR (Average True Range)

```
TR = max(high - low, |high - close[1]|, |low - close[1]|)
ATR(14) = RMA(TR, 14)  (Wilder's smoothing, 14-period)
```

### 1.11 Stop Loss Calculation

```
LONG STOP:
  SL_structure = OB_bottom - (tick_size × 5)
  SL_sweep = min(low of sweep candle) - (tick_size × 5)
  SL_raw = min(SL_structure, SL_sweep)   ← takes the LOWER of both
  
  ATR_min_stop = entry - (0.8 × ATR(14))
  ATR_max_stop = entry - (2.0 × ATR(14))
  
  SL_final:
    IF SL_raw > ATR_min_stop → SL_final = ATR_min_stop  (too tight, use ATR floor)
    IF SL_raw < ATR_max_stop → SKIP TRADE (OB too far, bad geometry)
    ELSE → SL_final = SL_raw
    
  MINIMUM CHECK: IF |entry - SL_final| / entry < 0.004 → SKIP TRADE

SHORT STOP: mirror (above OB_top, above sweep high, ATR inverted)
```

### 1.12 Take Profit Calculation

```
stop_distance = |entry - SL_final|

TP1 = entry + (stop_distance × 1.0)     [1:1 RR]  → close 50%
TP2 = entry + (stop_distance × 2.0)     [1:2 RR]  → close 35%
TP3 = nearest_liquidity_above(entry)     [runner]  → close 15%

  WHERE nearest_liquidity_above = min(PDH, PWH, EQH, next SH) that is > TP2
  IF no liquidity target exists beyond TP2 → TP3 = entry + (stop_distance × 3.0)

PATH CLEAR CHECK:
  major_levels = [PDH, PDL, PWH, PWL, EQH, EQL, unmitigated BEOB top]
  blocked = any(level WHERE entry < level < TP2)
  IF blocked → do NOT fire signal
```

### 1.13 Position Sizing

```
risk_amount = account_size × (risk_percent / 100)
stop_distance_usd = |entry - SL_final|
position_size_units = risk_amount / stop_distance_usd
position_size_usd = position_size_units × entry
leverage_required = position_size_usd / account_size

IF leverage_required > max_leverage_input → BLOCK SIGNAL
```

---

## 2. SIGNAL LOGIC — PSEUDOCODE

### 2.1 Long Signal Confluence

```python
def check_long_signal():
    # Layer 1: Structure
    if market_structure != BULLISH:
        return False
    
    # Layer 2: Location (price in valid OB or FVG)
    in_bull_ob = any(ob WHERE ob.type == BULLISH 
                     AND ob.mitigated == False
                     AND low <= ob.top AND close >= ob.bottom)
    in_bull_fvg = any(fvg WHERE fvg.type == BULLISH
                      AND low <= fvg.top AND close >= fvg.bottom)
    if not (in_bull_ob or in_bull_fvg):
        return False
    
    # Layer 3: Liquidity Sweep
    recent_lows = [EQL levels, swing lows within last 20 bars]
    sweep = any(level WHERE low < level AND close > level)
    if not sweep:
        return False
    
    # Layer 4: Momentum
    if rsi_14 >= 45:
        return False
    if ema_21 <= ema_50:
        return False
    
    # Layer 5: Candle Confirmation
    bullish_engulfing = (close > open[1] 
                         AND body > 0.6 * range 
                         AND close > high - 0.3 * range)
    hammer = (lower_wick > 2 * body AND close > midpoint)
    if not (bullish_engulfing or hammer):
        return False
    
    # Layer 6: Session Filter
    utc_hour = hour(time, "UTC")
    in_london = 7 <= utc_hour < 10
    in_ny = 13 <= utc_hour < 16
    if not (in_london or in_ny):
        return False
    
    # Layer 7: HTF Trend
    if htf_ema21 <= htf_ema50:
        return False
    
    # Invalidation Checks
    if ob_already_mitigated:
        return False
    if rsi_14 > 70:
        return False
    if price_within_0_3_pct_of_major_level:
        return False
    if news_filter_enabled AND news_active:
        return False
    if signals_today >= daily_limit:
        return False
    
    # Stop/TP Validation
    calculate_stop_loss()
    if stop_outside_atr_bounds:
        return False
    if stop_distance < 0.4%:
        return False
    calculate_take_profits()
    if path_blocked:
        return False
    if leverage_required > max_leverage:
        return False
    
    return True  # FIRE LONG SIGNAL
```

### 2.2 Short Signal Confluence (Mirror)

```python
def check_short_signal():
    # Layer 1: Structure = BEARISH (LH + LL confirmed)
    # Layer 2: Price in Bearish OB or Bearish FVG
    # Layer 3: Wick ABOVE EQH or swing high, close below (upside sweep)
    # Layer 4: RSI > 55, EMA 21 < EMA 50
    # Layer 5: Bearish engulfing or shooting star
    # Layer 6: London or NY session only
    # Layer 7: HTF EMA 21 < EMA 50
    # Same invalidation checks (RSI < 30 blocks, etc.)
    # Stop above BEOB high + sweep wick
    # TP targets mirrored below
```

---

## 3. VISUAL ELEMENTS SPECIFICATION

### 3.1 Color Palette

| Element | Color | Transparency | Pine Code |
|---------|-------|-------------|-----------|
| Bullish Structure (BOS/CHoCH lines) | Green | 0 | `color.green` |
| Bearish Structure (BOS/CHoCH lines) | Red | 0 | `color.red` |
| CHoCH lines | Yellow | 0 | `color.yellow` |
| Bullish Order Block box | Green | 80 | `color.new(color.green, 80)` |
| Bearish Order Block box | Red | 80 | `color.new(color.red, 80)` |
| Bullish FVG box | Blue | 85 | `color.new(color.blue, 85)` |
| Bearish FVG box | Blue | 85 | `color.new(color.blue, 85)` |
| EQH / EQL lines | Orange | 0 | `color.orange` |
| PDH / PDL lines | Orange | 20 | `color.new(color.orange, 20)` |
| PWH / PWL lines | Purple | 0 | `color.purple` |
| Asian Session box | Grey | 90 | `color.new(color.gray, 90)` |
| Long signal arrow | Green | 0 | `color.green` |
| Short signal arrow | Red | 0 | `color.red` |
| Stop Loss line | Red | 0 | `color.red` |
| TP1 line | Green | 40 | `color.new(color.green, 40)` |
| TP2 line | Green | 20 | `color.new(color.green, 20)` |
| TP3 line | Green | 60 | `color.new(color.green, 60)` |
| Mitigated OB | Grey | 70 | `color.new(color.gray, 70)` |
| IDM marker | White | 0 | `color.white` |
| Candles in Bull OB zone | Green | 0 | `color.green` |
| Candles in Bear OB zone | Red | 0 | `color.red` |
| Candles in Chop | Grey | 0 | `color.gray` |

### 3.2 Line Styles

| Element | Style | Width |
|---------|-------|-------|
| BOS level | Dashed | 1 |
| CHoCH level | Dashed | 1 |
| EQH / EQL | Dotted | 1 |
| PDH / PDL | Solid | 1 |
| PWH / PWL | Solid | 2 |
| Stop Loss | Solid | 2 |
| TP1 | Dashed | 1 |
| TP2 | Solid | 1 |
| TP3 | Dotted | 1 |

### 3.3 Labels

| Event | Text | Position | Size |
|-------|------|----------|------|
| BOS Bullish | "BOS ↑" | Above bar | Small |
| BOS Bearish | "BOS ↓" | Below bar | Small |
| CHoCH Bullish | "CHoCH ↑" | Above bar | Small |
| CHoCH Bearish | "CHoCH ↓" | Below bar | Small |
| EQH | "EQH 💧" | Right of line | Tiny |
| EQL | "EQL 💧" | Right of line | Tiny |
| IDM | "IDM ▲/▼" | At level | Tiny |
| Long Signal | "LONG ▲ | SL: X% | TP2: 1:2" | Below bar | Normal |
| Short Signal | "SHORT ▼ | SL: X% | TP2: 1:2" | Above bar | Normal |
| Stop Loss | "SL: $X | Y%" | Right of line | Tiny |
| TP1 | "TP1 1:1 | 50% EXIT" | Right of line | Tiny |
| TP2 | "TP2 1:2 | 35% EXIT" | Right of line | Tiny |
| TP3 | "TP3 Runner | 15%" | Right of line | Tiny |

### 3.4 Drawing Limits

| Object | Max Visible | Cleanup Rule |
|--------|-------------|--------------|
| Order Blocks | 5 | Delete oldest when exceeded |
| FVGs | 3 | Delete oldest when exceeded |
| BOS/CHoCH lines | 10 | Delete oldest when exceeded |
| EQH/EQL lines | 6 | Delete oldest when exceeded |
| Signal labels | 5 | Delete oldest when exceeded |

---

## 4. RISK MANAGEMENT CALCULATIONS — WORKED EXAMPLE

### Example: BTCUSDT Long on 1H

```
Given:
  Entry (confirmation candle close) = $67,500
  Bullish OB bottom = $67,100
  Sweep candle low (wick) = $67,050
  ATR(14) = $350
  Account = $1,000
  Risk = 1%

Step 1: Structure Stop
  SL_structure = $67,100 - (5 × $0.10) = $67,099.50

Step 2: Sweep Stop  
  SL_sweep = $67,050 - (5 × $0.10) = $67,049.50

Step 3: Use the LOWER (more protective)
  SL_raw = min($67,099.50, $67,049.50) = $67,049.50

Step 4: ATR Bounds Check
  ATR_min = $67,500 - (0.8 × $350) = $67,220  → SL_raw < this ✓ (not too tight)
  ATR_max = $67,500 - (2.0 × $350) = $66,800  → SL_raw > this ✓ (not too wide)

Step 5: Minimum Distance Check
  distance = ($67,500 - $67,049.50) / $67,500 = 0.667% > 0.4% ✓

Step 6: SL_final = $67,049.50
  Stop distance = $450.50

Step 7: Take Profits
  TP1 = $67,500 + $450.50 = $67,950.50  [1:1]
  TP2 = $67,500 + $901.00 = $68,401.00  [1:2]
  TP3 = PDH at $69,200 (> TP2) ✓

Step 8: Path Clear Check
  Any major level between $67,500 and $68,401?
  PDH = $69,200 → above TP2, no block
  PWL = $65,500 → below entry, irrelevant
  PATH CLEAR ✅

Step 9: Position Sizing
  risk_amount = $1,000 × 0.01 = $10
  position_size = $10 / $450.50 = 0.0222 BTC
  notional = 0.0222 × $67,500 = $1,498.50
  leverage = $1,498.50 / $1,000 = 1.5x ✓ (under 20x cap)

Step 10: Expected Outcomes
  Loss scenario: -$10 (1% of account)
  TP1 hit (50% close): +$5.00 locked, remaining at breakeven
  TP2 hit (35% close): +$5.00 + $6.31 = +$11.31
  Full runner (TP3): +$11.31 + remaining $2.52 ≈ +$13.83
  Effective RR at TP2: 1.13:1 risk-adjusted (accounting for partial exits)
```

---

## 5. INPUT PARAMETERS

| Input | Type | Default | Range | Group |
|-------|------|---------|-------|-------|
| Account Size (USD) | float | 1000 | 100–1,000,000 | Risk Management |
| Risk Per Trade (%) | float | 1.0 | 0.5–3.0 | Risk Management |
| Max Leverage | int | 10 | 1–20 | Risk Management |
| ATR Multiplier (Stop) | float | 1.0 | 0.5–3.0 | Risk Management |
| Daily Signal Limit | int | 2 | 1–5 | Risk Management |
| Show Order Blocks | bool | true | — | Visuals |
| Show FVGs | bool | true | — | Visuals |
| Show Liquidity Levels | bool | true | — | Visuals |
| Show Structure | bool | true | — | Visuals |
| Show Asian Session | bool | true | — | Visuals |
| Show Dashboard | bool | true | — | Visuals |
| Show TP/SL Lines | bool | true | — | Visuals |
| Enable HTF Filter | bool | true | — | Filters |
| Enable Session Filter | bool | true | — | Filters |
| Block Signals Before News | bool | false | — | Filters |
| EQH/EQL Tolerance (%) | float | 0.15 | 0.05–0.50 | Advanced |
| OB Impulse Threshold (ATR×) | float | 1.0 | 0.5–2.0 | Advanced |
| Swing Pivot Bars | int | 2 | 2–5 | Advanced |

---

## 6. ALERT CONDITIONS

| # | Alert Name | Trigger | Message Template |
|---|-----------|---------|-----------------|
| 1 | Long Signal | Long confluence = true | "🟢 LONG SIGNAL — {syminfo.tickerid} | Entry: {close} | SL: {sl_price} ({sl_pct}%) | TP1: {tp1} | TP2: {tp2}" |
| 2 | Short Signal | Short confluence = true | "🔴 SHORT SIGNAL — {syminfo.tickerid} | Entry: {close} | SL: {sl_price} ({sl_pct}%) | TP1: {tp1} | TP2: {tp2}" |
| 3 | TP1 Hit | Price crosses TP1 after signal | "✅ TP1 HIT — {syminfo.tickerid} | Move stop to breakeven | 50% closed" |
| 4 | TP2 Hit | Price crosses TP2 | "✅ TP2 HIT — {syminfo.tickerid} | 85% position closed, trail runner to TP1" |
| 5 | Stop Hit | Price crosses SL | "❌ STOP HIT — {syminfo.tickerid} | Trade closed at loss" |
| 6 | New OB Formed | OB detection fires | "📦 NEW {direction} ORDER BLOCK — {syminfo.tickerid} | Zone: {ob_top}–{ob_bottom}" |
| 7 | Liquidity Sweep | Sweep detected | "💧 LIQUIDITY SWEEP — {syminfo.tickerid} | {direction} sweep at {level}" |

---

## 7. ANTI-REPAINT RULES (Enforced in Code)

| Rule | Implementation |
|------|---------------|
| No intra-bar signals | All signal checks wrapped in `if barstate.isconfirmed` |
| No lookahead | `request.security(..., barmerge.lookahead_off)` everywhere |
| Pivot confirmation delay | Pivots use `[2]` offset — confirmed 2 bars after formation |
| Strategy entry timing | `strategy.entry()` on bar after signal (process_orders_on_close=false) |
| No close[0] in signals | All signal logic uses confirmed bar data only |
| Realistic fills | Commission: 0.05%, Slippage: 2 ticks |

---

## 8. ESTIMATED SCRIPT SIZE

| Component | Estimated Lines |
|-----------|----------------|
| Inputs & declarations | 80 |
| Helper functions | 120 |
| Market structure engine | 200 |
| Liquidity mapping | 150 |
| Order block detection | 180 |
| FVG detection | 80 |
| Signal confluence engine | 250 |
| Stop/TP calculations | 120 |
| Visual rendering | 300 |
| Dashboard table | 150 |
| Alert conditions | 60 |
| **TOTAL (Indicator)** | **~1,690 lines** |
| Strategy wrapper additions | ~200 lines |
| **TOTAL (Strategy)** | **~1,890 lines** |

---

## 9. FILE STRUCTURE

```
/pine-script-strategy/
├── requirements.md              ← This document
├── indicator_v1.pine            ← Phase 3: Full indicator/study
├── strategy_v1.pine             ← Phase 4: Backtest strategy
└── DEPLOYMENT_GUIDE.md          ← Phase 5: Setup instructions
```

---

## 10. APPROVAL CHECKPOINT

**This specification is complete and ready for review.**

Key architectural decisions:
1. Single-file indicator (~1,700 lines) — no library dependencies
2. All drawing objects managed via arrays with size caps
3. HTF data fetched once via `request.security()` with proper anti-repaint
4. Signal state machine: WAITING → READY → FIRED → TRACKING (TP/SL)
5. Dashboard updates on every bar but only redraws table cells (performance)

**Awaiting approval to proceed to Phase 3 (Indicator Code).**
