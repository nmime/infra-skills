# Aggressive Mode

**Strategy:** Momentum + Trend Confirmation | Full Autonomy | Top 50 Coins

## Target

Agent chooses target at start: **+50% to +100%**

| Account Size | Target Range | Reasoning |
|--------------|--------------|----------|
| < $100 | +100% | Small stack, aim to double |
| $100-1000 | +75% | Solid growth |
| > $1000 | +50% | Meaningful gains |

Minimum target: **+50%** — never lower.

Agent runs until target reached or stopped manually.

```
Example:
Starting Balance: $200
Agent picks: +75% target
Target Balance: $350

Agent trades until $350 hit.
```

## Parameters

| Parameter | Value |
|-----------|-------|
| Target | +50% to +100% (agent chooses) |
| Leverage | 5-15x (max 15x) |
| Position Size | 12-18% of account |
| Stop Loss | -5% to -8% |
| Take Profit | +12% to +20% (min 2× SL) |
| Max Positions | 3 |
| Scan Interval | 20 minutes |
| Daily Loss Limit | -10% |

## Dynamic Stop-Loss Levels

| Position P&L | Move Stop To | Locked Profit |
|--------------|--------------|---------------|
| +5% | Breakeven (-0.3%) | Risk-free |
| +8% | +4% | +4% guaranteed |
| +12% | +8% | +8% guaranteed |
| +15%+ | Trail 4% below max | Ride the trend |

## Setup Flow

### Step 0: Set Target

```javascript
// Get starting balance
hyperliquid_get_balance({})
STARTING_BALANCE = accountValue

// Agent chooses target based on account size
if (STARTING_BALANCE < 100) {
  TARGET_PCT = 100  // +100% for small accounts
} else if (STARTING_BALANCE < 1000) {
  TARGET_PCT = 75   // +75% for medium
} else {
  TARGET_PCT = 50   // +50% for larger
}

TARGET_BALANCE = STARTING_BALANCE * (1 + TARGET_PCT / 100)

// Log
"🚀 Aggressive mode: $X → Target: $Y (+Z%)"
```

### Step 1: Create Event Hub Webhook

```javascript
event_create_webhook({
  label: "hyperliquid_aggressive"
})
// Save webhook_id and webhook_url
```

### Step 2: Market Research

Always include chat_id as a prefix
```javascript
market_deepresearch({
  context_memory_id: "{chat_id}_aggressive_trading_session",
  message: `Quick scan (1-2 min max): Find the best momentum trade RIGHT NOW on Hyperliquid perpetuals.

Requirements:
1. Top 50 coins by volume only (no ultra-low liquidity)
2. Clear trend direction (up or down)
3. Recent catalyst or news preferred
4. Check funding rates for crowd positioning

I need ONE trade recommendation:
- Coin (top 50 on Hyperliquid)
- Direction (LONG or SHORT)
- Why (trend + catalyst)
- Confidence (must be 6+ out of 10 to trade)

Looking for confirmed momentum, not speculation.`
})
```

**Trade Requirement:** Only execute if confidence >= 6

### Step 3: Validate Coin on Hyperliquid

```javascript
// Check if coin exists and get max leverage
hyperliquid_get_meta({ coin: "COIN" })

// Get current price
hyperliquid_get_all_prices({ coins: ["COIN"] })

// Check funding rate
hyperliquid_get_funding_rates({ coin: "COIN" })

// Verify max leverage >= 10x, otherwise skip
if (maxLeverage < 10) {
  "Skipping COIN - leverage too low for aggressive mode"
  // Research another coin
}
```

### Step 3b: Pre-Trade Checks

```javascript
// 1. Check liquidity & spread (see SKILL.md)
const liquidity = await check_liquidity(COIN, MARGIN, 'aggressive')
if (!liquidity.ok) SKIP

// 2. Check BTC alignment
const btc_check = await check_btc_alignment(COIN, DIRECTION)
CONFIDENCE += btc_check.confidence_penalty

// 3. Check time conditions
const time_check = check_trading_conditions()
POSITION_SIZE_MULTIPLIER *= time_check.multiplier

// 4. Final confidence check
if (CONFIDENCE < 6) {
  telegram_send_message({ text: `⚠️ ${COIN} confidence ${CONFIDENCE}/10 after checks. Skipping.` })
  SKIP
}
```

### Step 4: Execute Trade

```javascript
// Set leverage (5-15x, cap at 15x)
LEVERAGE = Math.min(maxLeverage, 15)
LEVERAGE = Math.max(LEVERAGE, 5)

hyperliquid_update_leverage({
  coin: "COIN",
  leverage: LEVERAGE,
  is_cross: true
})

// Calculate position size with multipliers
let MARGIN = accountValue * 0.15
MARGIN *= POSITION_SIZE_MULTIPLIER

// Calculate TP/SL with enforced 2:1 minimum R:R
const SL_PCT = 6
const TP_PCT = Math.max(SL_PCT * 2, 12)

const SL_PRICE = ENTRY_PRICE * (is_buy ? (1 - SL_PCT/100) : (1 + SL_PCT/100))
const TP_PRICE = ENTRY_PRICE * (is_buy ? (1 + TP_PCT/100) : (1 - TP_PCT/100))

// Use slippage-protected order
const entry_result = await place_protected_order(COIN, is_buy, POSITION_SIZE, 'aggressive')

if (entry_result.fill_pct < 0.7) {
  // Retry or skip
}

// Place TP/SL after entry
hyperliquid_place_order({ coin: "COIN", order_type: "take_profit", trigger_price: TP_PRICE, reduce_only: true })
hyperliquid_place_order({ coin: "COIN", order_type: "stop_loss", trigger_price: SL_PRICE, reduce_only: true })
```

### Step 5: Setup Monitoring

```javascript
// Get all current position coins
hyperliquid_get_positions({})
ALL_COINS = positions.map(p => p.coin)

// Subscribe all coins to webhook with dynamic stop levels
hyperliquid_subscribe_webhook({
  webhook_url: WEBHOOK_URL,
  coins: ALL_COINS,
  events: ["fills", "orders"],
  position_alerts: ALL_COINS.map(coin => [
    { coin: coin, condition: "pnl_pct_gt", value: 5 },   // Breakeven trigger
    { coin: coin, condition: "pnl_pct_gt", value: 8 },   // +4% lock trigger
    { coin: coin, condition: "pnl_pct_gt", value: 12 },  // +8% lock trigger
    { coin: coin, condition: "pnl_pct_gt", value: 15 },  // Trailing start
    { coin: coin, condition: "pnl_pct_lt", value: -4 }   // Danger zone
  ]).flat()
})

// Subscribe to Event Hub to wake up on events
event_subscribe({
  webhook_id: WEBHOOK_ID,
  timeout: 86400,  // 24 hours
  triggers: [
    { name: "trade_events", filter: "payload.type == 'fill' || payload.type == 'order'", debounce: 5 },
    { name: "position_alerts", filter: "payload.type == 'position_alert'", debounce: 5 }
  ]
})
// Save subscription_id
```

### Step 6: Schedule Periodic Scans

```javascript
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 1200,  // 20 minutes
  message: "20min scan: Check positions, scan for new momentum plays"
})
// Save schedule_id
```

## Event Handling

**CRITICAL: Every event MUST end with re-schedule. Never skip.**

### On ANY Wake-up (first thing)

```javascript
// 1. Get current state
hyperliquid_get_balance({})
hyperliquid_get_positions({})

// 2. CHECK DRAWDOWN CIRCUIT BREAKER
const drawdown_check = await check_drawdown_circuit_breaker()
if (drawdown_check.halt) {
  STOP
}

// 3. Manage dynamic stops and partial takes for ALL positions
for (const position of positions) {
  await manage_dynamic_stop(position, 'aggressive')
  await manage_partial_takes(position, 'aggressive')
}

// 4. Report ALL positions to Telegram
telegram_send_message({
  chat_id: TELEGRAM_CHAT_ID,
  text: `📊 Positions:
${positions.map(p => `• ${p.coin} ${p.direction}: ${p.roe}% ROE`).join('\n')}
Balance: $${balance} (${sessionChange}%)
Progress: ${progress}% to target`
})

// 5. Check target
if (accountValue >= TARGET_BALANCE) {
  cleanup()
  STOP
}
```

### On Trade Event (fill/order)

1. Report what happened (TP/SL hit?)
2. **Record trade for performance tracking** (see SKILL.md `record_trade`)
3. If closed by trailing stop with profit → check re-entry opportunity
4. If position closed → research new trade (Steps 2-4)

### On Position Alert

| Alert | Action |
|-------|--------|
| +5% | Move stop to breakeven |
| +8% | Lock +4% profit |
| +12% | Lock +8% profit |
| +15%+ | Trail 4% behind price |
| -4% | Watch closely, DO NOT move stop down |

### On Schedule (20min scan)

1. Report position status (see above)
2. If < 3 positions → Research new trade
3. If 3 positions → Just monitor

### LAST STEP (every event, never skip)

```javascript
// Always re-schedule before ending
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 1200,  // 20 minutes
  message: "20min scan"
})

// If schedule fails (subscription expired):
// → Run Step 5-6 again to recreate subscription + schedule
```

**If you don't re-schedule, the agent dies.**

## Cleanup (on stop or target reached)

```javascript
// Close all positions
hyperliquid_get_positions({})
for (pos of positions) {
  hyperliquid_market_close({ coin: pos.coin })
}

// Cancel schedule
cancel_schedule({ schedule_id: SCHEDULE_ID })

// Unsubscribe from events
event_unsubscribe({ subscription_id: SUBSCRIPTION_ID })

// Remove Hyperliquid webhook
hyperliquid_unsubscribe_webhook({})

// Final report
hyperliquid_get_balance({})
"Session ended: $STARTING → $FINAL (±X%)"
```

## Position Sizing

```
Account Balance: $X
Position Size: 12-18% of $X (15% default)
Leverage: 5-15x (10x default)
Notional Value: Position Size × Leverage

Example ($100 account):
- Position Size: $15 margin (15%)
- Leverage: 10x
- Notional: $150
- Max 3 positions = $45 margin total (45% account)
- Max loss per position (6% SL): $0.90
- Max loss all positions: $2.70 (2.7% of account)
```

## Risk Controls

| Control | Value |
|---------|-------|
| Daily Loss Limit | -10% → Stop for day |
| 3 Consecutive Losses | Cooldown 4-6 hours, size -50% |
| 5 Losses in Day | Stop for 24 hours |
| Drawdown -15% | Reduce size 50%, pause 2 hours |
| Drawdown -20% | **HALT ALL TRADING** |
| Min R:R | 2:1 (enforced) |

## Coin Selection

Only trade coins that meet:
- Top 50 by volume on Hyperliquid
- Max leverage >= 10x
- Clear trend or catalyst

Examples: BTC, ETH, SOL, AVAX, ARB, OP, DOGE, LINK, etc.

Avoid: Ultra-low liquidity, brand new listings, < 10x leverage

## Notifications

All actions are logged. User sees:
- "🚀 Aggressive mode started: $100 → Target: $175 (+75%)"
- "Opened 15x LONG on SOL @ $124, trend: breakout above resistance"
- "SOL +8%, trailing stop active (Balance: $116, Target: $175)"
- "Closed SOL @ $130, P&L: +$18 (+15.6%)"
- "Progress: $118 / $175 (24% to target)"
- "🎉 TARGET REACHED! $100 → $178 (+78%)"

No questions asked. Execute and report.
