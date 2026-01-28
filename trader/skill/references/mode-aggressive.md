# Aggressive Mode

**Strategy:** Trend Momentum + Multi-Timeframe Confirmation + ATR-Based Risk | Full Autonomy

## Target

Agent chooses target at start: **+50% to +100%**

| Account Size | Target Range | Reasoning |
|--------------|--------------|----------|
| < $100 | +100% | Small stack, aim to double |
| $100-1000 | +75% | Solid growth |
| > $1000 | +50% | Meaningful gains |

Minimum target: **+50%** — never lower.

## Core Parameters

| Parameter | Value |
|-----------|-------|
| Target | +50% to +100% (agent chooses) |
| Leverage | 15-25x (high momentum) |
| Position Size | Half-Kelly based on confidence |
| Base Risk | 2-3% of account per trade |
| Stop Loss | 2× ATR (volatility-adjusted) |
| Take Profit | 4× ATR (minimum 2:1 R:R) |
| Trailing Stop | 1.5× ATR after 3× ATR profit |
| Max Positions | 3 (diversified focus) |
| Scan Interval | 20 minutes |
| Daily Loss Limit | -12% (hard stop) |
| Consecutive Loss Limit | 3 (then cooldown) |
| Min Confidence | 6+ |

## Risk Management Framework

### Position Sizing: Half-Kelly Criterion

```javascript
// Professional position sizing based on edge
KELLY_PCT = ((WIN_RATE * AVG_WIN) - (LOSS_RATE * AVG_LOSS)) / AVG_WIN
HALF_KELLY = KELLY_PCT / 2  // 75% of optimal growth, 50% less drawdown

// Confidence-adjusted sizing
POSITION_SIZE = BASE_RISK * (confidence / 10) * volatility_factor

// Examples with 2.5% base risk:
// Confidence 6, normal volatility: 2.5% * 0.6 * 1.0 = 1.5% risk
// Confidence 8, low volatility: 2.5% * 0.8 * 1.2 = 2.4% risk
// Confidence 6, high volatility: 2.5% * 0.6 * 0.7 = 1.05% risk
```

### ATR-Based Dynamic Stops

```javascript
// Get 14-period ATR for volatility measurement
ATR_14 = calculate_atr(coin, period=14, timeframe="1h")
ATR_PCT = (ATR_14 / current_price) * 100

// Volatility-adjusted stops
STOP_LOSS_DISTANCE = ATR_PCT * 2.0   // 2× ATR (wider than degen)
TAKE_PROFIT_DISTANCE = ATR_PCT * 4.0 // 4× ATR (2:1 R:R)
TRAILING_TRIGGER = ATR_PCT * 3.0     // Activate after 3× ATR profit
TRAILING_DISTANCE = ATR_PCT * 1.5    // Trail by 1.5× ATR

// Leverage safety check (critical at 15-25x)
MAX_SAFE_SL = (100 / LEVERAGE) * 0.6  // 60% of liquidation distance
if (STOP_LOSS_DISTANCE > MAX_SAFE_SL) {
  STOP_LOSS_DISTANCE = MAX_SAFE_SL
  TAKE_PROFIT_DISTANCE = STOP_LOSS_DISTANCE * 2  // Maintain 2:1 R:R
}

// Reference table:
// 15x: Liq at -6.7%, Safe SL: -4%, TP: +8%
// 20x: Liq at -5%, Safe SL: -3%, TP: +6%
// 25x: Liq at -4%, Safe SL: -2.4%, TP: +4.8%
```

### Multi-Timeframe Confirmation

```javascript
// Check trend alignment across timeframes
TREND_1H = get_trend("1h")   // Short-term
TREND_4H = get_trend("4h")   // Medium-term
TREND_1D = get_trend("1d")   // Long-term

// Scoring system
MTF_SCORE = 0
if (TREND_1H == direction) MTF_SCORE++
if (TREND_4H == direction) MTF_SCORE++
if (TREND_1D == direction) MTF_SCORE++

// Requirements
if (MTF_SCORE < 2) {
  "⚠️ Only " + MTF_SCORE + "/3 timeframes aligned. Skipping."
  SKIP
}

// Bonus confidence for full alignment
if (MTF_SCORE == 3) {
  confidence = confidence + 1
  "✅ All timeframes aligned, confidence boost"
}
```

### Volatility Regime Detection

```javascript
// Compare current ATR to historical average
ATR_CURRENT = calculate_atr(coin, 14, "1h")
ATR_AVERAGE = calculate_atr(coin, 50, "1h")
VOLATILITY_RATIO = ATR_CURRENT / ATR_AVERAGE

if (VOLATILITY_RATIO > 1.5) {
  // HIGH VOLATILITY
  volatility_factor = 0.7
  LEVERAGE = Math.min(LEVERAGE, 15)  // Cap at 15x in high vol
  "⚠️ High volatility, reducing exposure"
} else if (VOLATILITY_RATIO < 0.7) {
  // LOW VOLATILITY - breakout potential
  volatility_factor = 1.1
  "📊 Low volatility, watching for breakout"
} else {
  volatility_factor = 1.0
}
```

### Funding Rate Analysis

```javascript
FUNDING = hyperliquid_get_funding_rates({ coin: "COIN" })
FUNDING_PCT = FUNDING.rate * 100

// More conservative thresholds than degen
if (FUNDING_PCT > 0.03) {  // > 0.03% per hour
  FUNDING_BIAS = "SHORT"
  FUNDING_EDGE = true
  "💰 Positive funding - favor shorts"
} else if (FUNDING_PCT < -0.03) {
  FUNDING_BIAS = "LONG"
  FUNDING_EDGE = true
  "💰 Negative funding - favor longs"
} else {
  FUNDING_BIAS = "NEUTRAL"
  FUNDING_EDGE = false
}

// Penalize trades against funding
if (FUNDING_EDGE && trade_direction != FUNDING_BIAS) {
  confidence = confidence - 1
}
```

### Drawdown Protection (Tiered)

```javascript
DAILY_PNL_PCT = (accountValue - DAY_START_BALANCE) / DAY_START_BALANCE * 100

if (DAILY_PNL_PCT <= -12) {
  // HARD STOP
  "🛑 Daily loss limit (-12%). Stopping for today."
  cleanup()
  STOP
} else if (DAILY_PNL_PCT <= -8) {
  // REDUCED MODE
  "⚠️ Down 8%. Reducing size by 50%, confidence +1 required."
  BASE_RISK = BASE_RISK * 0.5
  MIN_CONFIDENCE = 7
} else if (DAILY_PNL_PCT <= -5) {
  // CAUTION MODE
  "⚠️ Down 5%. Tightening criteria."
  MIN_CONFIDENCE = 7
}
```

### Consecutive Loss Handler

```javascript
if (trade_result == "LOSS") {
  CONSECUTIVE_LOSSES++
} else {
  CONSECUTIVE_LOSSES = 0
}

if (CONSECUTIVE_LOSSES >= 3) {
  "🧊 3 consecutive losses. 45 min cooldown."
  schedule({
    delay: 2700,  // 45 min cooldown (longer than degen)
    message: "Cooldown complete"
  })
  BASE_RISK = BASE_RISK * 0.6
  CONSECUTIVE_LOSSES = 0
}
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   AGGRESSIVE MODE LOOP                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   TRIGGER    │───▶│   VALIDATE   │───▶│   EXECUTE    │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│         │                   │                   │           │
│         ▼                   ▼                   ▼           │
│  • 20min schedule     • Drawdown check     • Half-Kelly     │
│  • Event fires        • MTF confirmation   • ATR stops      │
│                       • Volatility regime  • 8-12x lever    │
│                       • Funding analysis   • Bracket order  │
│                       • Confidence 6+                       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              ENTRY REQUIREMENTS (ALL MUST PASS)       │   │
│  │  1. Daily drawdown < -8% (or reduced mode)           │   │
│  │  2. No 3+ consecutive losses                         │   │
│  │  3. Confidence >= 6                                  │   │
│  │  4. At least 2/3 timeframes aligned                  │   │
│  │  5. Funding not extreme against direction            │   │
│  │  6. Position count < 3                               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              HYBRID EXIT STRATEGY                     │   │
│  │  • Primary: ATR-based TP (4× ATR)                    │   │
│  │  • After 3× ATR profit: Trailing (1.5× ATR)          │   │
│  │  • Stop loss: 2× ATR (max 70% to liquidation)        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Coin Selection

Only trade coins meeting these criteria:

```javascript
ALLOWED_CRITERIA = {
  min_volume_24h: 10_000_000,  // $10M+ daily volume
  max_leverage: 10,            // Must support 10x+
  top_n_by_volume: 50,         // Top 50 coins
  funding_not_extreme: true    // |funding| < 0.05%
}

// Preferred coins (high liquidity, clean trends)
PREFERRED = ["BTC", "ETH", "SOL", "AVAX", "ARB", "OP", "DOGE", "LINK", "MATIC", "APT"]
```

## Setup Flow

### Step 0: Initialize Session

```javascript
hyperliquid_get_balance({})
STARTING_BALANCE = accountValue
DAY_START_BALANCE = accountValue

// Initialize tracking
CONSECUTIVE_LOSSES = 0
TOTAL_TRADES = 0
WINS = 0
LOSSES = 0
BASE_RISK = 0.025  // 2.5% base risk

// Set target
if (STARTING_BALANCE < 100) {
  TARGET_PCT = 100
} else if (STARTING_BALANCE < 1000) {
  TARGET_PCT = 75
} else {
  TARGET_PCT = 50
}

TARGET_BALANCE = STARTING_BALANCE * (1 + TARGET_PCT / 100)

"🚀 Aggressive mode initialized
   Starting: $" + STARTING_BALANCE + "
   Target: $" + TARGET_BALANCE + " (+" + TARGET_PCT + "%)
   Risk per trade: " + (BASE_RISK * 100) + "%
   Scanning every 20 minutes"
```

### Step 1: Create Event Hub Webhook

```javascript
event_create_webhook({
  label: "hyperliquid_aggressive"
})
```

### Step 2: Pre-Trade Validation

```javascript
hyperliquid_get_balance({})
DAILY_PNL_PCT = (accountValue - DAY_START_BALANCE) / DAY_START_BALANCE * 100

// Drawdown check
if (DAILY_PNL_PCT <= -12) {
  cleanup()
  STOP
}

// Adjust based on drawdown
if (DAILY_PNL_PCT <= -8) {
  ADJUSTED_RISK = BASE_RISK * 0.5
  MIN_CONFIDENCE = 7
} else if (DAILY_PNL_PCT <= -5) {
  ADJUSTED_RISK = BASE_RISK * 0.75
  MIN_CONFIDENCE = 7
} else {
  ADJUSTED_RISK = BASE_RISK
  MIN_CONFIDENCE = 6
}

// Consecutive losses
if (CONSECUTIVE_LOSSES >= 3) {
  "🧊 In cooldown"
  SKIP
}

// Position count
hyperliquid_get_positions({})
if (positions.length >= 3) {
  "📊 Max positions (3)"
  SKIP
}
```

### Step 3: Market Research

```javascript
market_deepresearch({
  context_memory_id: "{chat_id}_aggressive_session",
  message: `Find the best momentum trade on Hyperliquid perpetuals.

REQUIREMENTS:
1. Top 50 coins by volume (good liquidity)
2. Clear trend on multiple timeframes (1h, 4h aligned minimum)
3. Recent breakout or trend continuation setup
4. Catalyst or news is a bonus

CHECK:
- Funding rates (avoid extreme crowding)
- Recent price action (momentum confirmation)
- Volume (should be above average)

OUTPUT:
- Coin symbol (top 50 preferred)
- Direction (LONG or SHORT)
- Trend confirmation (which timeframes aligned)
- Entry reason (technical + catalyst if any)
- Confidence (1-10, need 6+ to trade)

Current positions: ` + JSON.stringify(positions) + `
Avoid same coins for diversification.
Account status: ` + DAILY_PNL_PCT.toFixed(1) + `% today`
})
```

### Step 4: Multi-Timeframe Validation

```javascript
// Get coin data
hyperliquid_get_meta({ coin: "COIN" })
hyperliquid_get_all_prices({ coins: ["COIN"] })
hyperliquid_get_funding_rates({ coin: "COIN" })

// Multi-timeframe check (via research or price data analysis)
MTF_ALIGNED = count_aligned_timeframes(COIN, direction)

if (MTF_ALIGNED < 2) {
  "⚠️ Only " + MTF_ALIGNED + "/3 timeframes aligned. SKIP."
  SKIP
}

// Calculate volatility
ATR_PCT = estimate_atr(COIN)

if (ATR_PCT > 4) {
  volatility_factor = 0.7
} else if (ATR_PCT < 1.5) {
  volatility_factor = 1.1
} else {
  volatility_factor = 1.0
}
```

### Step 5: Calculate Position Size

```javascript
CONFIDENCE_FACTOR = confidence / 10

// Risk calculation
RISK_AMOUNT = accountValue * ADJUSTED_RISK * CONFIDENCE_FACTOR * volatility_factor

// Stop distance (wider for aggressive, more room to breathe)
STOP_DISTANCE_PCT = Math.min(ATR_PCT * 2.0, (100 / LEVERAGE) * 0.7)

// Position size from risk
MARGIN = RISK_AMOUNT / (STOP_DISTANCE_PCT / 100)
MARGIN = Math.min(MARGIN, accountValue * 0.35)  // Max 35% per position

// Set leverage (15-25x for aggressive)
LEVERAGE = 20  // Default
if (volatility_factor < 1.0) {
  LEVERAGE = 15  // Reduce in high vol
} else if (MTF_ALIGNED == 3) {
  LEVERAGE = 25  // Max when all timeframes aligned
}

NOTIONAL = MARGIN * LEVERAGE
POSITION_SIZE = NOTIONAL / CURRENT_PRICE

"📊 Position:
   Risk: $" + RISK_AMOUNT.toFixed(2) + " (" + (RISK_AMOUNT/accountValue*100).toFixed(1) + "%)
   Margin: $" + MARGIN.toFixed(2) + "
   Leverage: " + LEVERAGE + "x
   Stop: " + STOP_DISTANCE_PCT.toFixed(2) + "%
   Target: " + (STOP_DISTANCE_PCT * 2).toFixed(2) + "% (2:1 R:R)"
```

### Step 6: Execute Trade

```javascript
hyperliquid_update_leverage({
  coin: "COIN",
  leverage: LEVERAGE,
  is_cross: true
})

// Calculate prices with 2:1 R:R
if (is_buy) {
  ENTRY_PRICE = CURRENT_PRICE * 1.0005
  SL_PRICE = ENTRY_PRICE * (1 - STOP_DISTANCE_PCT / 100)
  TP_PRICE = ENTRY_PRICE * (1 + STOP_DISTANCE_PCT * 2 / 100)
} else {
  ENTRY_PRICE = CURRENT_PRICE * 0.9995
  SL_PRICE = ENTRY_PRICE * (1 + STOP_DISTANCE_PCT / 100)
  TP_PRICE = ENTRY_PRICE * (1 - STOP_DISTANCE_PCT * 2 / 100)
}

hyperliquid_place_bracket_order({
  coin: "COIN",
  is_buy: is_buy,
  size: POSITION_SIZE,
  entry_price: ENTRY_PRICE,
  take_profit_price: TP_PRICE,
  stop_loss_price: SL_PRICE
})

TOTAL_TRADES++

"🎯 " + (is_buy ? "LONG" : "SHORT") + " " + COIN + "
   Entry: $" + ENTRY_PRICE + " (" + LEVERAGE + "x)
   Stop: $" + SL_PRICE + " (-" + STOP_DISTANCE_PCT.toFixed(1) + "%)
   Target: $" + TP_PRICE + " (+" + (STOP_DISTANCE_PCT * 2).toFixed(1) + "%)
   Timeframes aligned: " + MTF_ALIGNED + "/3
   Confidence: " + confidence + "/10"
```

### Step 7: Setup Monitoring

```javascript
// Get all position coins
ALL_COINS = positions.map(p => p.coin).concat([COIN])

hyperliquid_subscribe_webhook({
  webhook_url: WEBHOOK_URL,
  coins: ALL_COINS,
  events: ["fills", "orders"],
  position_alerts: [
    { condition: "pnl_pct_gt", value: STOP_DISTANCE_PCT * 1.5 },  // Profit zone
    { condition: "pnl_pct_lt", value: -STOP_DISTANCE_PCT * 0.5 }  // Warning
  ]
})

event_subscribe({
  webhook_id: WEBHOOK_ID,
  timeout: 86400,
  triggers: [
    { name: "trade_events", filter: "payload.type == 'fill' || payload.type == 'order'", debounce: 5 },
    { name: "position_alerts", filter: "payload.type == 'position_alert'", debounce: 5 }
  ]
})
```

### Step 8: Schedule Scans

```javascript
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 1200,  // 20 minutes
  message: "20min scan: Monitor positions, find setups"
})
```

## Event Handling

### On Trade Event

```javascript
if (event.type == "fill" && event.closedPosition) {
  PNL = event.realizedPnl

  if (PNL > 0) {
    WINS++
    CONSECUTIVE_LOSSES = 0
    "✅ WIN: +$" + PNL.toFixed(2) + " | " + WINS + "W/" + LOSSES + "L"
  } else {
    LOSSES++
    CONSECUTIVE_LOSSES++
    "❌ LOSS: $" + PNL.toFixed(2) + " | Streak: " + CONSECUTIVE_LOSSES
  }

  WIN_RATE = WINS / TOTAL_TRADES

  // Check target
  hyperliquid_get_balance({})
  if (accountValue >= TARGET_BALANCE) {
    cleanup()
    "🎉 TARGET! $" + STARTING_BALANCE + " → $" + accountValue.toFixed(2)
    STOP
  }

  // Consecutive loss check
  if (CONSECUTIVE_LOSSES >= 3) {
    "🧊 3 losses. 45 min cooldown."
    schedule({ delay: 2700, message: "Cooldown complete" })
  }
}
```

### On Position Alert

```javascript
if (alert.condition == "pnl_pct_gt") {
  // In profit - consider trailing
  "📈 " + alert.coin + " in profit zone, trailing stop active"

  hyperliquid_modify_order({
    coin: alert.coin,
    trailing_stop: true,
    trail_distance_pct: ATR_PCT * 1.5
  })
}

if (alert.condition == "pnl_pct_lt") {
  // Early warning - monitor closely
  "⚠️ " + alert.coin + " approaching stop loss"
}
```

### On Schedule (20min scan)

```javascript
// 1. Drawdown check
hyperliquid_get_balance({})
DAILY_PNL_PCT = (accountValue - DAY_START_BALANCE) / DAY_START_BALANCE * 100

if (DAILY_PNL_PCT <= -12) {
  cleanup()
  STOP
}

// 2. Target check
if (accountValue >= TARGET_BALANCE) {
  cleanup()
  "🎉 TARGET!"
  STOP
}

// 3. Portfolio status
hyperliquid_get_positions({})

"📊 Portfolio Status:
   Balance: $" + accountValue.toFixed(2) + " (" + DAILY_PNL_PCT.toFixed(1) + "% today)
   Progress: " + ((accountValue - STARTING_BALANCE) / (TARGET_BALANCE - STARTING_BALANCE) * 100).toFixed(0) + "%
   Positions: " + positions.length + "/3
   Record: " + WINS + "W/" + LOSSES + "L (" + (WIN_RATE * 100).toFixed(0) + "%)"

// 4. Position details
for (pos of positions) {
  "   • " + pos.coin + ": " + (pos.unrealizedPnl > 0 ? "+" : "") + pos.unrealizedPnl.toFixed(2)
}

// 5. Look for new trades if room
if (positions.length < 3 && CONSECUTIVE_LOSSES < 3) {
  // Research new opportunities
}

// 6. Re-schedule
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 1200,
  message: "20min scan"
})
```

## Cleanup

```javascript
// Close all positions
for each position:
  hyperliquid_market_close({ coin: position.coin })

cancel_schedule({ schedule_id: SCHEDULE_ID })
event_unsubscribe({ subscription_id: SUBSCRIPTION_ID })
hyperliquid_unsubscribe_webhook({})

"Session Complete:
   Duration: X hours
   Starting: $" + STARTING_BALANCE + "
   Final: $" + accountValue.toFixed(2) + "
   Return: " + ((accountValue - STARTING_BALANCE) / STARTING_BALANCE * 100).toFixed(1) + "%
   Total trades: " + TOTAL_TRADES + "
   Win rate: " + (WIN_RATE * 100).toFixed(0) + "% (" + WINS + "W/" + LOSSES + "L)
   Avg win: +$X
   Avg loss: -$Y
   Profit factor: " + (TOTAL_WINS / TOTAL_LOSSES).toFixed(2)
```

## Position Sizing Examples

```
Account: $500

Trade 1: Confidence 7, Normal volatility
- Risk: $500 × 2.5% × 0.7 × 1.0 = $8.75
- ATR: 2.5%, Stop: 5% (2× ATR)
- Margin: $8.75 / 0.05 = $175
- Leverage: 10x
- Notional: $1,750

Trade 2: Confidence 8, Low volatility
- Risk: $500 × 2.5% × 0.8 × 1.1 = $11.00
- ATR: 1.5%, Stop: 3% (2× ATR)
- Margin: $11 / 0.03 = $366
- Capped at 35%: $175
- Leverage: 10x
- Notional: $1,750

Trade 3: Confidence 6, High volatility
- Risk: $500 × 2.5% × 0.6 × 0.7 = $5.25
- ATR: 4%, Stop: 5.6% (leverage-safe max)
- Margin: $5.25 / 0.056 = $94
- Leverage: 8x (reduced)
- Notional: $752
```

## Key Principles

1. **Trend is Friend** - Only trade with multi-timeframe alignment
2. **Risk First** - Never exceed calculated risk per trade
3. **ATR Adapts** - Stops scale with volatility automatically
4. **Diversify** - Max 3 positions across different coins
5. **2:1 Minimum** - Every trade must have positive expectancy
6. **Protect Capital** - Tiered drawdown response
7. **No Revenge** - Mandatory cooldown after losing streaks
8. **Let Winners Run** - Trailing stops capture extended moves

## Notifications

- "🚀 Aggressive mode: ${STARTING} → Target: ${TARGET} (+{PCT}%)"
- "📊 {COIN} {MTF_SCORE}/3 timeframes aligned ({TF_DETAIL})"
- "🎯 {DIRECTION} {COIN} @ ${ENTRY}, {LEVERAGE}x | SL: ${SL} (-{SL_PCT}%) | TP: ${TP} (+{TP_PCT}%)"
- "📈 {COIN} +{PROFIT_PCT}%, trailing stop active"
- "✅ WIN +${PNL} | Record: {W}W/{L}L ({WIN_RATE}%)"
- "📊 Portfolio: {POS_COUNT}/{MAX_POS} positions | +${TODAY_PNL} today (+{TODAY_PCT}%)"
- "🧊 3 losses → 45 min cooldown"
- "🎉 TARGET! ${STARTING} → ${FINAL} (+{RETURN}%)"
