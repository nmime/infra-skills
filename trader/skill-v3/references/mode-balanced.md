# Balanced Mode

**Strategy:** Trend Following + Multi-Confirmation + Diversified | Full Autonomy | Quality Over Quantity | KV State Tracking

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

### Step 0: Initialize Session (KV Storage)

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

// Initialize all KV state
await init_session(chat_id, 'balanced', STARTING_BALANCE, TARGET_BALANCE)

// Notify
telegram_send_message({
  text: `⚖️ *Balanced mode started*
Balance: $${STARTING_BALANCE}
Target: $${TARGET_BALANCE} (+${TARGET_PCT}%)
Scan: every 2hr`
})
```

### Step 1: Create Event Hub Webhook

```javascript
event_create_webhook({
  label: "hyperliquid_balanced"
})
// Save to session KV
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.webhook_id = webhook_id
session.webhook_url = webhook_url
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
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

### Step 3b: Pre-Trade Checks

```javascript
// 1. Check liquidity & spread
const liquidity = await check_liquidity(COIN, MARGIN, 'balanced')
if (!liquidity.ok) SKIP

// 2. Check BTC alignment (required for balanced)
const btc_check = await check_btc_alignment(COIN, DIRECTION)
if (!btc_check.aligned) {
  telegram_send_message({ text: `⚠️ ${COIN} ${DIRECTION} against BTC trend. Skipping.` })
  SKIP  // Balanced mode: strict BTC alignment
}

// 3. Check time conditions
const time_check = check_trading_conditions()
let SIZE_MULTIPLIER = time_check.multiplier

// 4. Check drawdown
const balance = await hyperliquid_get_balance({})
const drawdown = await check_drawdown_circuit_breaker(chat_id, balance.accountValue)
if (drawdown.halt) STOP
SIZE_MULTIPLIER *= drawdown.size_multiplier

// 5. Check consecutive losses
const loss_check = await check_consecutive_losses(chat_id)
if (loss_check.stop_24h) STOP
if (loss_check.cooldown) SIZE_MULTIPLIER *= loss_check.size_multiplier

// 6. Check daily loss limit
const daily = await check_daily_loss_limit(chat_id, 'balanced')
if (daily.exceeded) SKIP

// 7. Final confidence check
if (CONFIDENCE < 7) SKIP
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

// Calculate position size with multipliers
let MARGIN = accountValue * 0.10
MARGIN *= SIZE_MULTIPLIER

// Calculate TP/SL with enforced 2:1 minimum R:R
const SL_PCT = 4
const TP_PCT = Math.max(SL_PCT * 2, 8)

const SL_PRICE = ENTRY_PRICE * (is_buy ? (1 - SL_PCT/100) : (1 + SL_PCT/100))
const TP_PRICE = ENTRY_PRICE * (is_buy ? (1 + TP_PCT/100) : (1 - TP_PCT/100))

// Use slippage-protected order
const entry_result = await place_protected_order(COIN, is_buy, POSITION_SIZE, 'balanced')

// Place TP/SL after entry
hyperliquid_place_order({ coin: "COIN", order_type: "take_profit", trigger_price: TP_PRICE, reduce_only: true })
hyperliquid_place_order({ coin: "COIN", order_type: "stop_loss", trigger_price: SL_PRICE, reduce_only: true })
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

// Save subscription_id to session
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.subscription_id = SUBSCRIPTION_ID
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
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
const drawdown = await check_drawdown_circuit_breaker(chat_id, accountValue)
if (drawdown.halt) {
  await cleanup_session(chat_id)
  STOP
}

// 3. Check daily loss limit
const daily = await check_daily_loss_limit(chat_id, 'balanced')

// 4. Check consecutive losses
const loss_check = await check_consecutive_losses(chat_id)
if (loss_check.stop_24h) {
  schedule({ subscription_id: SUBSCRIPTION_ID, delay: 86400, message: "24h cooldown ended" })
  STOP
}

// 5. Manage dynamic stops and partial takes
for (const position of positions) {
  await manage_dynamic_stop(position, 'balanced')
  await manage_partial_takes(chat_id, position, 8)  // tp_pct = 8% for balanced
}

// 6. Report ALL positions to Telegram
const stats = JSON.parse(await splox_kv_get({ key: `${chat_id}_stats` }) || '{}')
telegram_send_message({
  chat_id: TELEGRAM_CHAT_ID,
  text: `📊 Portfolio: ${positions.length}/4
${positions.map(p => `• ${p.coin} ${p.direction}: ${p.pnl_pct}%`).join('\n')}
Balance: $${balance} / $${target} (${progress}% progress)
Stats: ${stats.wins || 0}W/${stats.losses || 0}L`
})

// 7. Check target
if (accountValue >= TARGET_BALANCE) {
  await cleanup_session(chat_id)
  STOP
}
```

### On Trade Event (fill/order)

```javascript
if (payload.closedPosition) {
  const closed = payload.closedPosition

  // Record the trade
  await record_trade(chat_id, {
    coin: closed.coin,
    direction: closed.direction,
    entry_price: closed.entryPx,
    exit_price: closed.exitPx,
    pnl_pct: closed.pnl_pct,
    pnl_usd: closed.pnl_usd,
    duration_min: (Date.now() - closed.openTime) / 60000,
    exit_reason: closed.exit_reason
  })

  // Clear partials tracking
  await clear_partials(chat_id, closed.coin)

  // Check re-entry if trailing stop exit
  if (closed.exit_reason === 'trailing_stop' && closed.pnl_pct > 0) {
    const reentry = await check_reentry_opportunity(closed)
    if (reentry.reentry) {
      // Execute re-entry trade
    }
  }

  // Position closed → slot opens for new trade
}
```

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
await cleanup_session(chat_id)
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
| Drawdown -10% | Size reduced 30% |
| Drawdown -15% | Pause 2 hours, size -50% |
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
- "📊 Portfolio: {POS}/{MAX} | +{TOTAL_PCT}% | {W}W/{L}L | Next: 2hr"
- "📈 {COIN} +{PROFIT_PCT}%, trailing active | Next: 2hr"
- "💰 {COIN} Partial #1: 30% closed at +{PNL}%"
- "✅ WIN +${PNL} | {W}W/{L}L | Next: 2hr"
- "❌ LOSS ${PNL} | Streak: {N} | Next: 2hr"
- "🎉 TARGET! ${STARTING} → ${FINAL} (+{RETURN}%) | {W}W/{L}L"

Patient, diversified, disciplined.
