# Balanced Mode

**Strategy:** Trend Following + Multi-Confirmation + Diversified | Full Autonomy | Quality Over Quantity

## Target

Agent chooses target at start: **+25% to +50%**

| Account Size | Target Range | Reasoning |
|--------------|--------------|----------|
| < $100 | +50% | Grow small stack steadily |
| $100-1000 | +40% | Solid compounding |
| > $1000 | +25% | Protect larger capital |

Minimum target: **+25%** — patient growth.

Agent runs until target reached or stopped manually.

```
Example:
Starting Balance: $500
Agent picks: +40% target
Target Balance: $700

Agent trades patiently until $700 hit.
```

## Parameters

| Parameter | Value |
|-----------|-------|
| Target | +25% to +50% (agent chooses) |
| Leverage | 3-5x |
| Position Size | 8-12% of account |
| Stop Loss | -3% to -5% |
| Take Profit | +8% to +12% (min 2× SL) |
| Max Positions | 4 (diversified) |
| Scan Interval | 2 hours |
| Daily Loss Limit | -8% (hard stop) |

## Dynamic Stop-Loss Levels

| Position P&L | Move Stop To | Locked Profit |
|--------------|--------------|---------------|
| +3% | Breakeven (-0.2%) | Risk-free |
| +5% | +2% | +2% guaranteed |
| +8% | +5% | +5% guaranteed |
| +10%+ | Trail 3% below max | Ride the trend |

## Setup Flow

### Step 0: Set Target

```javascript
// Get starting balance
hyperliquid_get_balance({})
STARTING_BALANCE = accountValue

// Agent chooses target based on account size
if (STARTING_BALANCE < 100) {
  TARGET_PCT = 50   // +50% for small accounts
} else if (STARTING_BALANCE < 1000) {
  TARGET_PCT = 40   // +40% for medium
} else {
  TARGET_PCT = 25   // +25% for larger
}

TARGET_BALANCE = STARTING_BALANCE * (1 + TARGET_PCT / 100)

// Log
"⚖️ Balanced mode: $X → Target: $Y (+Z%)"
```

### Step 1: Create Event Hub Webhook

```javascript
event_create_webhook({
  label: "hyperliquid_balanced"
})
// Save webhook_id and webhook_url
```

### Step 2: Market Research (Multi-Confirmation)
Always include chat_id as a prefix
```javascript
market_deepresearch({
  context_memory_id: "{chat_id}_balanced_session",
  message: `Quick scan (1-2 min max): Find the best momentum trade RIGHT NOW on Hyperliquid perpetuals.

Requirements - ALL must be met:
1. TREND: Clear trend on 4h/daily timeframe (up or down)
2. MOMENTUM: Recent price action confirms trend (not reversing)
3. FUNDING: Not extreme against our direction (avoid crowded trades)
4. CATALYST: Bonus if news/event supports the move

I need up to 2 trade recommendations (for diversification):
- Coin (prefer top 30, good liquidity)
- Direction (LONG only if uptrend, SHORT only if downtrend)
- Confirmations (list what's aligned)
- Confidence (must be 7+ out of 10)

IMPORTANT:
- Skip if trend unclear
- Skip if signals conflict
- Quality over quantity
- It's OK to recommend nothing if no good setups exist

Current positions: [LIST CURRENT POSITIONS]
Avoid same coins, aim for diversification.`
})
```

**Trade Requirement:**
- Confidence >= 7
- At least 3 confirmations aligned
- OK to skip if nothing qualifies

### Step 3: Validate Coins on Hyperliquid

```javascript
// For each recommended coin:
hyperliquid_get_meta({ coin: "COIN" })
hyperliquid_get_all_prices({ coins: ["COIN"] })
hyperliquid_get_funding_rates({ coin: "COIN" })

// Check constraints:
// - Max leverage >= 5x
// - Not already in portfolio
// - Funding not extreme against position
```

### Step 4: Execute Trade (Diversified)

```javascript
// Check current positions
hyperliquid_get_positions({})

// Portfolio rules:
// - Max 4 positions total
// - Max 2 LONG, Max 2 SHORT (balance)
// - Max 15% margin per position

if (positions.length >= 4) {
  "Portfolio full, waiting for exits"
  SKIP
}

// Set leverage (3-5x)
LEVERAGE = Math.min(maxLeverage, 5)
LEVERAGE = Math.max(LEVERAGE, 3)

hyperliquid_update_leverage({
  coin: "COIN",
  leverage: LEVERAGE,
  is_cross: true
})

// Calculate position size (8-12% of account)
MARGIN = accountValue * 0.10  // 10% default

// Calculate TP/SL with enforced 2:1 minimum R:R
const SL_PCT = 4  // -4%
const TP_PCT = Math.max(SL_PCT * 2, 8)  // Minimum 2× SL = +8%

const SL_PRICE = ENTRY_PRICE * (is_buy ? (1 - SL_PCT/100) : (1 + SL_PCT/100))
const TP_PRICE = ENTRY_PRICE * (is_buy ? (1 + TP_PCT/100) : (1 - TP_PCT/100))

// Place bracket order with tight stops
hyperliquid_place_bracket_order({
  coin: "COIN",
  is_buy: true,  // or false for SHORT
  size: POSITION_SIZE,
  entry_price: ENTRY_PRICE,
  take_profit_price: TP_PRICE,  // +8-12% (2× SL minimum)
  stop_loss_price: SL_PRICE     // -3-5%
})
```

### Step 5: Setup Monitoring

```javascript
// Subscribe Hyperliquid to send events to Event Hub
// Note: If multiple coins, duplicate this call for each coin
hyperliquid_subscribe_webhook({
  webhook_url: WEBHOOK_URL,
  coins: ["COIN"],
  events: ["fills", "orders"],
  position_alerts: [
    { coin: "COIN", condition: "pnl_pct_gt", value: 3 },   // Breakeven trigger
    { coin: "COIN", condition: "pnl_pct_gt", value: 5 },   // +2% lock trigger
    { coin: "COIN", condition: "pnl_pct_gt", value: 8 },   // +5% lock trigger
    { coin: "COIN", condition: "pnl_pct_gt", value: 10 },  // Trailing start
    { coin: "COIN", condition: "pnl_pct_lt", value: -2 }   // Danger zone
  ]
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
  delay: 7200,  // 2 hours - patient
  message: "2hr scan: Check portfolio, look for quality setups"
})
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

// 3. Manage dynamic stops for ALL positions
for (const position of positions) {
  await manage_dynamic_stop(position, 'balanced')
}

// 4. Report ALL positions to Telegram
telegram_send_message({
  chat_id: TELEGRAM_CHAT_ID,
  text: `📊 Portfolio: ${positions.length}/4
${positions.map(p => `• ${p.coin} ${p.direction}: ${p.pnl_pct}%`).join('\n')}
Balance: $${balance} / $${target} (${progress}% progress)`
})

// 5. Check target
if (accountValue >= TARGET_BALANCE) {
  cleanup()
  STOP
}
```

### On Trade Event (fill/order)

1. Report what happened (TP/SL hit?)
2. If closed by trailing stop with profit → check re-entry opportunity
3. If position closed → slot opens for new trade

### On Position Alert

| Alert | Action |
|-------|--------|
| +3% | Move stop to breakeven |
| +5% | Lock +2% profit |
| +8% | Lock +5% profit |
| +10%+ | Trail 3% behind price |
| -2% | Monitor closely, DO NOT move stop down |

### On Schedule (2hr scan)

1. Report portfolio status (see above)
2. If < 4 positions AND confidence 7+ setup → Research new trade
3. Check LONG/SHORT balance

### LAST STEP (every event, never skip)

```javascript
// Always re-schedule before ending
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 7200,  // 2 hours
  message: "2hr scan"
})

// If schedule fails (subscription expired):
// → Run Step 5-6 again to recreate subscription + schedule
```

**If you don't re-schedule, the agent dies.**

## Cleanup (on stop or target reached)

```javascript
// Close all positions
for each position:
  hyperliquid_market_close({ coin: position.coin })

// Cancel schedule
cancel_schedule({ schedule_id: SCHEDULE_ID })

// Unsubscribe from events
event_unsubscribe({ subscription_id: SUBSCRIPTION_ID })

// Remove Hyperliquid webhook
hyperliquid_unsubscribe_webhook({})

// Final report with stats
hyperliquid_get_balance({})
"Session ended: $STARTING → $FINAL (±X%)
 Trades: Y total, W wins, L losses
 Win rate: Z%"
```

## Position Sizing

```
Account Balance: $X
Max Positions: 4
Per Position: 8-12% margin (10% default)
Leverage: 3-5x (4x default)
Total Max Exposure: 40% of account in margin

Example ($500 account):
- Per position: $50 margin (10%)
- Leverage: 4x
- Notional per position: $200
- Max 4 positions = $200 margin (40% account)
- Total notional: $800
- Max loss per position (4% SL): $2
- Max loss all positions: $8 (1.6% of account)
```

## Risk Controls

| Control | Value |
|---------|-------|
| Daily Loss Limit | -8% → Stop for day |
| 3 Consecutive Losses | Cooldown 4-6 hours, size -50% |
| 5 Losses in Day | Stop for 24 hours |
| Drawdown -15% | Reduce size 50%, pause 2 hours |
| Drawdown -20% | **HALT ALL TRADING** |
| Min R:R | 2:1 (enforced) |

## Confirmation Checklist

Before every trade, verify:

```
□ Trend clear on higher timeframe?
□ Price moving WITH trend?
□ Funding not extreme against us?
□ Confidence 7+ ?
□ Portfolio has room (< 4 positions)?
□ Not doubling up on same coin?
□ Balanced LONG/SHORT exposure?

If ANY checkbox = NO → SKIP
```

## Notifications

- "⚖️ Balanced: ${STARTING} → Target: ${TARGET} (+{PCT}%) | Scan: 2hr"
- "🔍 No setup found | Next: 2hr"
- "🎯 {DIRECTION} {COIN} @ ${ENTRY} | {LEV}x | Conf: {CONF}/10 | Next: 2hr"
- "📊 Portfolio: {POS}/{MAX} | +{TOTAL_PCT}% | Next: 2hr"
- "📈 {COIN} +{PROFIT_PCT}%, trailing active | Next: 2hr"
- "✅ WIN +${PNL} | {W}W/{L}L | Next: 2hr"
- "❌ LOSS ${PNL} | Streak: {N} | Next: 2hr"
- "🎉 TARGET! ${STARTING} → ${FINAL} (+{RETURN}%)"

Patient, diversified, disciplined.