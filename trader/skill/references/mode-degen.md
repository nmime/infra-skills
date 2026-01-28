# Degen Mode

**Strategy:** High-Frequency Momentum + Funding Rate Edge + ATR-Based Risk | Full Autonomy

## Target

Agent chooses target at start: **+100% to +300%**

| Account Size | Target Range | Reasoning |
|--------------|--------------|----------|
| < $100 | +200-300% | Small stack, swing big |
| $100-1000 | +150-200% | Aggressive growth |
| > $1000 | +100-150% | Preserve while growing |

Minimum target: **+100% (2x)** — never lower.

## Core Parameters

| Parameter | Value |
|-----------|-------|
| Target | +100% to +300% (agent chooses) |
| Leverage | 25-50x (high risk, tight stops) |
| Position Size | Half-Kelly based on confidence |
| Base Risk | 3-5% of account per trade |
| Stop Loss | 1.5× ATR (volatility-adjusted) |
| Take Profit | 3× ATR (minimum 1:2 R:R) |
| Trailing Stop | 1× ATR after 2× ATR profit |
| Max Positions | 2 (agent decides: concentrate or diversify) |
| Scan Interval | 10 minutes |
| Daily Loss Limit | -20% (hard stop) |
| Consecutive Loss Limit | 3 (then cooldown) |
| Min Confidence | 5+ |

## Risk Management Framework

### Position Sizing: Half-Kelly Criterion

```javascript
// Professional position sizing based on edge
KELLY_PCT = ((WIN_RATE * AVG_WIN) - (LOSS_RATE * AVG_LOSS)) / AVG_WIN
HALF_KELLY = KELLY_PCT / 2  // 75% of growth, 50% less drawdown

// Confidence-adjusted sizing
// Higher confidence = closer to half-kelly
// Lower confidence = smaller position
POSITION_SIZE = BASE_RISK * (confidence / 10) * volatility_factor

// Examples:
// Confidence 5, normal volatility: 3% * 0.5 * 1.0 = 1.5% risk
// Confidence 8, low volatility: 3% * 0.8 * 1.2 = 2.9% risk
// Confidence 5, high volatility: 3% * 0.5 * 0.7 = 1.05% risk
```

### ATR-Based Dynamic Stops

```javascript
// Get 14-period ATR for volatility measurement
ATR_14 = calculate_atr(coin, period=14, timeframe="1h")
ATR_PCT = (ATR_14 / current_price) * 100  // ATR as percentage

// Volatility-adjusted stops (adapts to market conditions)
STOP_LOSS_DISTANCE = ATR_PCT * 1.5   // 1.5× ATR
TAKE_PROFIT_DISTANCE = ATR_PCT * 3.0 // 3× ATR (2:1 R:R minimum)
TRAILING_TRIGGER = ATR_PCT * 2.0     // Activate trailing after 2× ATR profit
TRAILING_DISTANCE = ATR_PCT * 1.0    // Trail by 1× ATR

// Leverage-adjusted safety check
MAX_SAFE_SL = (100 / LEVERAGE) * 0.6  // 60% of liquidation distance
if (STOP_LOSS_DISTANCE > MAX_SAFE_SL) {
  STOP_LOSS_DISTANCE = MAX_SAFE_SL
  // Recalculate TP to maintain 2:1 R:R
  TAKE_PROFIT_DISTANCE = STOP_LOSS_DISTANCE * 2
}

// Quick reference for 25-50x leverage:
// 25x: Liq -4%, Safe SL -2.4%, TP +4.8%
// 30x: Liq -3.3%, Safe SL -2%, TP +4%
// 40x: Liq -2.5%, Safe SL -1.5%, TP +3%
// 50x: Liq -2%, Safe SL -1.2%, TP +2.4%
```

### Volatility Regime Detection

```javascript
// Compare current ATR to 50-period average
ATR_CURRENT = calculate_atr(coin, 14, "1h")
ATR_AVERAGE = calculate_atr(coin, 50, "1h")
VOLATILITY_RATIO = ATR_CURRENT / ATR_AVERAGE

// Adapt parameters to regime
if (VOLATILITY_RATIO > 1.5) {
  // HIGH VOLATILITY - reduce exposure
  volatility_factor = 0.6
  LEVERAGE = Math.min(LEVERAGE, 25)  // Cap at 25x in high vol
  "⚠️ High volatility detected, reducing position size"
} else if (VOLATILITY_RATIO < 0.7) {
  // LOW VOLATILITY - can increase slightly
  volatility_factor = 1.2
  "📊 Low volatility, normal sizing"
} else {
  // NORMAL
  volatility_factor = 1.0
}
```

### Funding Rate Edge

```javascript
// Get current funding rate
FUNDING = hyperliquid_get_funding_rates({ coin: "COIN" })
FUNDING_PCT = FUNDING.rate * 100

// Funding rate signals
if (FUNDING_PCT > 0.05) {  // > 0.05% per hour = extreme positive
  // Longs are paying shorts heavily
  // EDGE: Fade longs (go SHORT) or avoid longs
  FUNDING_BIAS = "SHORT"
  FUNDING_EDGE = true
  "💰 Extreme positive funding - short bias (longs paying)"
} else if (FUNDING_PCT < -0.05) {  // < -0.05% per hour = extreme negative
  // Shorts are paying longs heavily
  // EDGE: Fade shorts (go LONG) or avoid shorts
  FUNDING_BIAS = "LONG"
  FUNDING_EDGE = true
  "💰 Extreme negative funding - long bias (shorts paying)"
} else {
  FUNDING_BIAS = "NEUTRAL"
  FUNDING_EDGE = false
}

// If research direction conflicts with funding edge, reduce confidence
if (FUNDING_EDGE && trade_direction != FUNDING_BIAS) {
  confidence = confidence - 2
  "⚠️ Trade against funding edge, reducing confidence"
}
```

### Drawdown Protection (Tiered)

```javascript
// Track daily P&L
DAILY_PNL_PCT = (accountValue - DAY_START_BALANCE) / DAY_START_BALANCE * 100

// Tiered response
if (DAILY_PNL_PCT <= -20) {
  // HARD STOP
  "🛑 Daily loss limit hit (-20%). STOPPING."
  cleanup()
  STOP
} else if (DAILY_PNL_PCT <= -15) {
  // REDUCE RISK
  "⚠️ Down 15% today. Reducing position sizes by 50%."
  BASE_RISK = BASE_RISK * 0.5
  MAX_POSITIONS = 1
} else if (DAILY_PNL_PCT <= -10) {
  // CAUTION MODE
  "⚠️ Down 10% today. Increasing confidence threshold to 7+."
  MIN_CONFIDENCE = 7
}
```

### Consecutive Loss Handler

```javascript
// Track consecutive losses
if (trade_result == "LOSS") {
  CONSECUTIVE_LOSSES++
} else {
  CONSECUTIVE_LOSSES = 0
}

// Anti-tilt mechanism
if (CONSECUTIVE_LOSSES >= 3) {
  "🧊 3 consecutive losses. Cooldown for 30 minutes."
  schedule({
    delay: 1800,  // 30 min cooldown
    message: "Cooldown complete. Resuming with reduced size."
  })
  BASE_RISK = BASE_RISK * 0.5  // Halve risk after cooldown
  CONSECUTIVE_LOSSES = 0
}
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     DEGEN MODE LOOP                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   TRIGGER    │───▶│   VALIDATE   │───▶│   EXECUTE    │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│         │                   │                   │           │
│         ▼                   ▼                   ▼           │
│  • 10min schedule     • Drawdown check     • Half-Kelly size│
│  • Event fires        • Funding rate       • ATR stops      │
│                       • Volatility regime  • 15-25x lever   │
│                       • Confidence 5+      • Bracket order  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              ENTRY REQUIREMENTS                       │   │
│  │  1. Daily drawdown < -15% (or reduced mode)          │   │
│  │  2. No 3+ consecutive losses (or in cooldown)        │   │
│  │  3. Confidence >= 5 (research score)                 │   │
│  │  4. Funding rate not extreme against direction       │   │
│  │  5. Position count < 2                               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              HYBRID EXIT STRATEGY                     │   │
│  │  • Primary: ATR-based take profit (3× ATR)           │   │
│  │  • After 2× ATR profit: Activate trailing (1× ATR)   │   │
│  │  • Stop loss: 1.5× ATR (max 60% to liquidation)      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Setup Flow

### Step 0: Initialize Session

```javascript
// Get starting balance
hyperliquid_get_balance({})
STARTING_BALANCE = accountValue
DAY_START_BALANCE = accountValue

// Initialize tracking
CONSECUTIVE_LOSSES = 0
TOTAL_TRADES = 0
WINS = 0
LOSSES = 0
BASE_RISK = 0.04  // 4% base risk per trade

// Set target based on account size
if (STARTING_BALANCE < 100) {
  TARGET_PCT = 250
} else if (STARTING_BALANCE < 1000) {
  TARGET_PCT = 175
} else {
  TARGET_PCT = 125
}

TARGET_BALANCE = STARTING_BALANCE * (1 + TARGET_PCT / 100)

"🎰 Degen mode initialized
   Starting: $" + STARTING_BALANCE + "
   Target: $" + TARGET_BALANCE + " (+" + TARGET_PCT + "%)
   Risk per trade: " + (BASE_RISK * 100) + "%
   Scanning every 10 minutes"
```

### Step 1: Create Event Hub Webhook

```javascript
event_create_webhook({
  label: "hyperliquid_degen"
})
// Save webhook_id and webhook_url
```

### Step 2: Pre-Trade Validation

```javascript
// Check all conditions before researching
hyperliquid_get_balance({})
DAILY_PNL_PCT = (accountValue - DAY_START_BALANCE) / DAY_START_BALANCE * 100

// Drawdown check
if (DAILY_PNL_PCT <= -20) {
  cleanup()
  STOP
}

// Adjust risk based on drawdown
if (DAILY_PNL_PCT <= -15) {
  ADJUSTED_RISK = BASE_RISK * 0.5
  MIN_CONFIDENCE = 7
} else if (DAILY_PNL_PCT <= -10) {
  ADJUSTED_RISK = BASE_RISK * 0.75
  MIN_CONFIDENCE = 6
} else {
  ADJUSTED_RISK = BASE_RISK
  MIN_CONFIDENCE = 5
}

// Consecutive loss check
if (CONSECUTIVE_LOSSES >= 3) {
  "🧊 In cooldown after 3 losses"
  SKIP
}

// Position count check
hyperliquid_get_positions({})
if (positions.length >= 2) {
  "📊 Max positions reached (2)"
  SKIP
}
```

### Step 3: Market Research

```javascript
market_deepresearch({
  context_memory_id: "{chat_id}_degen_session",
  message: `RAPID SCAN (1 min): Find the best momentum trades NOW on Hyperliquid.

SCAN FOR:
1. Top movers in last 1-4 hours (biggest % moves)
2. Breaking news or catalysts
3. Extreme funding rates (opportunity to fade)
4. Volume spikes (unusual activity)

REQUIREMENTS:
- Must be on Hyperliquid perps
- Clear momentum direction
- Recent catalyst or technical breakout
- CHECK MAX LEVERAGE AVAILABLE (need 15x+ for degen mode)

OUTPUT (up to 2 trades for diversification):
For EACH opportunity:
- Coin symbol
- Max leverage available on Hyperliquid
- Direction (LONG or SHORT)
- Entry reason (catalyst + momentum)
- Confidence (1-10, need 5+ to trade)
- Current funding rate

MULTI-POSITION GUIDANCE:
If 2+ setups found, YOU DECIDE:
- Go all-in on best setup (higher risk, higher reward)
- Split across setups (lower risk, diversified)
- Choose allocation % based on your conviction

Trust your research. Make the call.

Current funding bias: ` + FUNDING_BIAS + `
Account status: ` + DAILY_PNL_PCT + `% today`
})
```

### Step 4: Validate Coin & Calculate ATR

```javascript
// Get coin metadata - CHECK LEVERAGE FIRST
hyperliquid_get_meta({ coin: "COIN" })
MAX_LEVERAGE = meta.maxLeverage

// LEVERAGE CHECK - Agent decides how to handle
if (MAX_LEVERAGE < 15) {
  "⚠️ " + COIN + " max leverage: " + MAX_LEVERAGE + "x"
  // Agent decides:
  // - Proceed anyway if setup is strong (funding edge, catalyst)
  // - Skip and find higher-leverage alternative
  // - Use max available and adjust expectations
}

// Get current price
hyperliquid_get_all_prices({ coins: ["COIN"] })
CURRENT_PRICE = prices.COIN

// Get funding rate
hyperliquid_get_funding_rates({ coin: "COIN" })
FUNDING_RATE = funding.rate

// Calculate ATR (approximate using recent price data)
ATR_PCT = estimated_volatility  // e.g., 2-5% for volatile coins

// Volatility regime
if (ATR_PCT > 5) {
  volatility_factor = 0.6
  "⚠️ Extremely volatile, reducing size"
} else if (ATR_PCT > 3) {
  volatility_factor = 0.8
} else {
  volatility_factor = 1.0
}

// Adjust leverage to what's available
LEVERAGE = Math.min(MAX_LEVERAGE, 50)
LEVERAGE = Math.max(LEVERAGE, 15)  // Prefer 15x+ for degen
if (MAX_LEVERAGE < 15) {
  LEVERAGE = MAX_LEVERAGE  // Use max if below threshold
  "📊 Using " + LEVERAGE + "x (coin max)"
}
```

### Step 4b: Position Strategy (Agent Decides)

```javascript
// Agent has full autonomy on allocation strategy
// Options when multiple setups found:

// OPTION A: Concentrate (max conviction play)
if (SETUP_1.confidence >= 8) {
  "Going all-in on " + SETUP_1.coin + " - highest conviction"
  ALLOCATION = 1.0
}

// OPTION B: Diversify (spread risk)
if (want_diversification) {
  "Splitting across " + SETUP_1.coin + " and " + SETUP_2.coin
  // Agent chooses split ratio based on confidence gap
}

// OPTION C: Sequential (one now, one later)
if (prefer_sequential) {
  "Starting with " + SETUP_1.coin + ", will add " + SETUP_2.coin + " on next scan"
}

// Agent makes the call based on:
// - Confidence levels
// - Correlation between coins
// - Current market conditions
// - Gut feeling from research
```

### Step 5: Calculate Position Size (Half-Kelly)

```javascript
// Confidence-based sizing
CONFIDENCE_FACTOR = confidence / 10  // 0.5 to 1.0

// Calculate risk amount
RISK_AMOUNT = accountValue * ADJUSTED_RISK * CONFIDENCE_FACTOR * volatility_factor

// Calculate stop distance
STOP_DISTANCE_PCT = Math.min(ATR_PCT * 1.5, (100 / LEVERAGE) * 0.6)

// Position size from risk
MARGIN = RISK_AMOUNT / (STOP_DISTANCE_PCT / 100)

// Position size cap - agent can adjust based on conviction
MAX_MARGIN_PCT = 0.50  // Guideline: 50% max per position
// Agent may go higher (up to 80%) for very high conviction plays
// Agent may go lower for uncertain setups
MARGIN = Math.min(MARGIN, accountValue * MAX_MARGIN_PCT)

// Calculate actual position (25-50x for degen)
LEVERAGE = Math.min(MAX_LEVERAGE, 50)
LEVERAGE = Math.max(LEVERAGE, 25)
NOTIONAL = MARGIN * LEVERAGE
POSITION_SIZE = NOTIONAL / CURRENT_PRICE

"📊 Position calculated:
   Risk: $" + RISK_AMOUNT + " (" + (RISK_AMOUNT/accountValue*100).toFixed(1) + "%)
   Margin: $" + MARGIN + "
   Leverage: " + LEVERAGE + "x
   Stop: " + STOP_DISTANCE_PCT.toFixed(2) + "%"
```

### Step 6: Execute Trade with ATR Stops

```javascript
// Set leverage
hyperliquid_update_leverage({
  coin: "COIN",
  leverage: LEVERAGE,
  is_cross: true
})

// Calculate prices
if (is_buy) {
  ENTRY_PRICE = CURRENT_PRICE * 1.001  // Slight slippage buffer
  SL_PRICE = ENTRY_PRICE * (1 - STOP_DISTANCE_PCT / 100)
  TP_PRICE = ENTRY_PRICE * (1 + STOP_DISTANCE_PCT * 2 / 100)  // 2:1 R:R
} else {
  ENTRY_PRICE = CURRENT_PRICE * 0.999
  SL_PRICE = ENTRY_PRICE * (1 + STOP_DISTANCE_PCT / 100)
  TP_PRICE = ENTRY_PRICE * (1 - STOP_DISTANCE_PCT * 2 / 100)
}

// Place bracket order
hyperliquid_place_bracket_order({
  coin: "COIN",
  is_buy: is_buy,
  size: POSITION_SIZE,
  entry_price: ENTRY_PRICE,
  take_profit_price: TP_PRICE,
  stop_loss_price: SL_PRICE
})

TOTAL_TRADES++

"🎯 Trade executed:
   " + (is_buy ? "LONG" : "SHORT") + " " + COIN + " @ $" + ENTRY_PRICE + "
   Size: " + POSITION_SIZE + " (" + LEVERAGE + "x)
   SL: $" + SL_PRICE + " (-" + STOP_DISTANCE_PCT.toFixed(1) + "%)
   TP: $" + TP_PRICE + " (+" + (STOP_DISTANCE_PCT * 2).toFixed(1) + "%)
   Risk: $" + RISK_AMOUNT
```

### Step 7: Setup Monitoring

```javascript
// Subscribe Hyperliquid events
hyperliquid_subscribe_webhook({
  webhook_url: WEBHOOK_URL,
  coins: [COIN],
  events: ["fills", "orders"],
  position_alerts: [
    { coin: COIN, condition: "pnl_pct_gt", value: STOP_DISTANCE_PCT * 2 },  // TP zone
    { coin: COIN, condition: "pnl_pct_lt", value: -STOP_DISTANCE_PCT * 0.5 }  // Early warning
  ]
})

// Subscribe to Event Hub
event_subscribe({
  webhook_id: WEBHOOK_ID,
  timeout: 86400,
  triggers: [
    { name: "trade_events", filter: "payload.type == 'fill' || payload.type == 'order'", debounce: 3 },
    { name: "position_alerts", filter: "payload.type == 'position_alert'", debounce: 3 }
  ]
})
```

### Step 8: Schedule Fast Scans

```javascript
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 600,  // 10 minutes
  message: "10min scan: Check positions, find momentum"
})
```

## Event Handling

### On Trade Event (fill/order)

```javascript
// Determine result
if (event.type == "fill" && event.closedPosition) {
  PNL = event.realizedPnl

  if (PNL > 0) {
    WINS++
    CONSECUTIVE_LOSSES = 0
    "✅ WIN: +" + PNL + " | Record: " + WINS + "W/" + LOSSES + "L"
  } else {
    LOSSES++
    CONSECUTIVE_LOSSES++
    "❌ LOSS: " + PNL + " | Consecutive: " + CONSECUTIVE_LOSSES
  }

  // Update win rate for Kelly calculation
  WIN_RATE = WINS / TOTAL_TRADES

  // Check target
  hyperliquid_get_balance({})
  if (accountValue >= TARGET_BALANCE) {
    cleanup()
    "🎉 TARGET REACHED! $" + STARTING_BALANCE + " → $" + accountValue
    STOP
  }

  // Check consecutive losses
  if (CONSECUTIVE_LOSSES >= 3) {
    "🧊 3 losses in a row. 30 min cooldown."
    schedule({ delay: 1800, message: "Cooldown complete" })
  }
}
```

### On Position Alert (profit zone)

```javascript
// If in profit zone (2× ATR), consider trailing stop
if (alert.condition == "pnl_pct_gt") {
  "📈 In profit zone, activating trailing stop"

  // Modify to trailing stop
  hyperliquid_modify_order({
    coin: COIN,
    trailing_stop: true,
    trail_distance_pct: ATR_PCT * 1.0  // Trail by 1× ATR
  })
}
```

### On Schedule (10min scan)

```javascript
// 1. Check drawdown
hyperliquid_get_balance({})
DAILY_PNL_PCT = (accountValue - DAY_START_BALANCE) / DAY_START_BALANCE * 100

if (DAILY_PNL_PCT <= -20) {
  cleanup()
  STOP
}

// 2. Check target
if (accountValue >= TARGET_BALANCE) {
  cleanup()
  "🎉 TARGET!"
  STOP
}

// 3. Status report
"📊 Status:
   Balance: $" + accountValue + " (" + DAILY_PNL_PCT.toFixed(1) + "% today)
   Progress: " + ((accountValue - STARTING_BALANCE) / (TARGET_BALANCE - STARTING_BALANCE) * 100).toFixed(0) + "% to target
   Record: " + WINS + "W/" + LOSSES + "L (" + (WIN_RATE * 100).toFixed(0) + "%)
   Positions: " + positions.length + "/2"

// 4. If room for trade, research
if (positions.length < 2 && CONSECUTIVE_LOSSES < 3) {
  // Run research flow
}

// 5. Re-schedule
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 600,
  message: "10min scan"
})
```

## Cleanup

```javascript
// Close all positions
for each position:
  hyperliquid_market_close({ coin: position.coin })

// Cancel schedule
cancel_schedule({ schedule_id: SCHEDULE_ID })

// Unsubscribe
event_unsubscribe({ subscription_id: SUBSCRIPTION_ID })
hyperliquid_unsubscribe_webhook({})

// Final report
"Session complete:
   Duration: X hours
   Starting: $" + STARTING_BALANCE + "
   Final: $" + accountValue + "
   Return: " + ((accountValue - STARTING_BALANCE) / STARTING_BALANCE * 100).toFixed(1) + "%
   Trades: " + TOTAL_TRADES + " (" + WINS + "W/" + LOSSES + "L)
   Win Rate: " + (WIN_RATE * 100).toFixed(0) + "%
   Best trade: +$X
   Worst trade: -$Y"
```

## Key Principles

1. **Risk First** - Never risk more than calculated, regardless of confidence
2. **ATR Adapts** - Stops adjust to volatility, not fixed percentages
3. **Funding Edge** - Use extreme funding as contrarian signal
4. **Kelly Sizing** - Position size based on edge, not emotion
5. **Drawdown Limits** - Live to trade another day
6. **No Revenge** - Cooldown after consecutive losses
7. **2:1 Minimum** - Never take trade with worse than 2:1 R:R

## Notifications

- "🎰 Degen mode: ${STARTING} → Target: ${TARGET} (+{PCT}%) | Scan: 10min"
- "📊 High volatility (ATR {ATR}%), reducing size | Next: 10min"
- "💰 Extreme funding on {COIN}, {BIAS} bias | Next: 10min"
- "🎯 {DIRECTION} {COIN} @ ${ENTRY} | SL: ${SL} | TP: ${TP} | Next: 10min"
- "✅ WIN +${PNL} | {W}W/{L}L ({WIN_RATE}%) | Next: 10min"
- "❌ LOSS ${PNL} | Streak: {N} | Next: 10min"
- "🔍 No setup | Next: 10min"
- "🧊 3 losses → cooldown 30min"
- "⚠️ Down {DD}% → reducing size | Next: 10min"
- "🎉 TARGET! ${STARTING} → ${FINAL} (+{RETURN}%)"
