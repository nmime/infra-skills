# Degen Mode

**Strategy:** Momentum + News | Full Autonomy | Any Coin | KV State Tracking

## Target

Agent chooses target at start: **+100% to +300%**

| Account Size | Target Range | Reasoning |
|--------------|--------------|----------|
| < $100 | +200-300% | Small stack, swing big |
| $100-1000 | +150-200% | Moderate growth |
| > $1000 | +100-150% | Preserve gains |

Minimum target: **+100% (2x)** — never lower.

Agent runs until target reached or stopped manually.

```
Example:
Starting Balance: $50
Agent picks: +250% target
Target Balance: $175

Agent trades until $175 hit.
```

## Parameters

| Parameter | Value |
|-----------|-------|
| Target | +100% to +300% (agent chooses) |
| Leverage | 15-25x (max 25x) |
| Position Size | 20-30% of account |
| Stop Loss | -8% to -12% |
| Take Profit | +20% to +30% (min 2× SL) |
| Max Positions | 3 |
| Scan Interval | 10 minutes |
| Daily Loss Limit | -15% |

## Dynamic Stop-Loss Levels

| Position P&L | Move Stop To | Locked Profit |
|--------------|--------------|---------------|
| +5% | Breakeven (-0.3%) | Risk-free |
| +10% | +5% | +5% guaranteed |
| +15% | +10% | +10% guaranteed |
| +20%+ | Trail 5% below max | Ride the trend |

## Setup Flow

### Step 0: Initialize Session (KV Storage)

```javascript
// Get starting balance
hyperliquid_get_balance({})
STARTING_BALANCE = accountValue

// Agent chooses target based on account size
if (STARTING_BALANCE < 100) {
  TARGET_PCT = 250  // +250% for small accounts
} else if (STARTING_BALANCE < 1000) {
  TARGET_PCT = 175  // +175% for medium
} else {
  TARGET_PCT = 125  // +125% for larger
}

TARGET_BALANCE = STARTING_BALANCE * (1 + TARGET_PCT / 100)

// Initialize all KV state
await init_session(chat_id, 'degen', STARTING_BALANCE, TARGET_BALANCE)

// Notify
telegram_send_message({
  text: `🎰 *Degen mode started*
Balance: $${STARTING_BALANCE}
Target: $${TARGET_BALANCE} (+${TARGET_PCT}%)
Scan: every 10min`
})
```

### Step 1: Create Event Hub Webhook

```javascript
event_create_webhook({
  label: "hyperliquid_degen"
})
// Save webhook_id and webhook_url to session
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.webhook_id = webhook_id
session.webhook_url = webhook_url
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
```

### Step 2: Market Research

Always include chat_id as a prefix
```javascript
market_deepresearch({
  context_memory_id: "{chat_id}_degen_trading_session",
  message: `Quick scan (1-2 min max): Find the best momentum trade RIGHT NOW on Hyperliquid perpetuals.

Check:
1. Top movers in last 1-4 hours (biggest pumps/dumps)
2. Any breaking news or catalysts for crypto coins
3. Extreme funding rates (crowded trades to fade or ride)

I need ONE trade recommendation:
- Coin (available on Hyperliquid perps)
- Direction (LONG or SHORT)
- Why (catalyst/momentum reason)
- Confidence (1-10)

Looking for volatile plays - shitcoins welcome.`
})
```

### Step 3: Validate Coin on Hyperliquid

```javascript
// Check if coin exists and get max leverage
hyperliquid_get_meta({ coin: "COIN" })

// Get current price
hyperliquid_get_all_prices({ coins: ["COIN"] })

// Check funding rate
hyperliquid_get_funding_rates({ coin: "COIN" })
```

### Step 3b: Pre-Trade Checks

```javascript
// 1. Check liquidity & spread (see SKILL.md)
const liquidity = await check_liquidity(COIN, MARGIN, 'degen')
if (!liquidity.ok) SKIP

// 2. Check BTC alignment (see SKILL.md)
const btc_check = await check_btc_alignment(COIN, DIRECTION)
CONFIDENCE += btc_check.confidence_penalty  // -2 if misaligned

// 3. Check time conditions
const time_check = check_trading_conditions()
let SIZE_MULTIPLIER = time_check.multiplier

// 4. Check drawdown - get multiplier
const balance = await hyperliquid_get_balance({})
const drawdown = await check_drawdown_circuit_breaker(chat_id, balance.accountValue)
if (drawdown.halt) STOP
SIZE_MULTIPLIER *= drawdown.size_multiplier

// 5. Check consecutive losses
const loss_check = await check_consecutive_losses(chat_id)
if (loss_check.stop_24h) STOP
if (loss_check.cooldown) SIZE_MULTIPLIER *= loss_check.size_multiplier

// 6. Check daily loss limit
const daily = await check_daily_loss_limit(chat_id, 'degen')
if (daily.exceeded) SKIP

// 7. Final confidence check after adjustments
if (CONFIDENCE < 5) {
  telegram_send_message({ text: `⚠️ ${COIN} confidence too low: ${CONFIDENCE}/10. Skipping.` })
  SKIP
}
```

### Step 4: Execute Trade

```javascript
// Set leverage (15-25x, cap at 25x)
const LEVERAGE = Math.min(maxLeverage, 25)

hyperliquid_update_leverage({
  coin: "COIN",
  leverage: LEVERAGE,
  is_cross: true
})

// Calculate position size with all multipliers
let MARGIN = accountValue * 0.25  // 25% default
MARGIN *= SIZE_MULTIPLIER  // Apply drawdown/time/loss adjustments

// Calculate TP/SL with enforced 2:1 minimum R:R
const SL_PCT = 10  // -10%
const TP_PCT = Math.max(SL_PCT * 2, 20)  // Minimum 2× SL = +20%

const SL_PRICE = ENTRY_PRICE * (is_buy ? (1 - SL_PCT/100) : (1 + SL_PCT/100))
const TP_PRICE = ENTRY_PRICE * (is_buy ? (1 + TP_PCT/100) : (1 - TP_PCT/100))

// Use slippage-protected order (see SKILL.md)
const entry_result = await place_protected_order(COIN, is_buy, POSITION_SIZE, 'degen')

if (!entry_result.filled || entry_result.fill_pct < 0.7) {
  telegram_send_message({ text: `⚠️ Order only ${(entry_result.fill_pct * 100).toFixed(0)}% filled. Retrying...` })
  // Retry with wider slippage or skip
}

// Place TP/SL orders after entry confirmed
hyperliquid_place_order({
  coin: "COIN",
  is_buy: !is_buy,
  size: FILLED_SIZE,
  order_type: "take_profit",
  trigger_price: TP_PRICE,
  reduce_only: true
})

hyperliquid_place_order({
  coin: "COIN",
  is_buy: !is_buy,
  size: FILLED_SIZE,
  order_type: "stop_loss",
  trigger_price: SL_PRICE,
  reduce_only: true
})
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
    { coin: coin, condition: "pnl_pct_gt", value: 10 },  // +5% lock trigger
    { coin: coin, condition: "pnl_pct_gt", value: 15 },  // +10% lock trigger
    { coin: coin, condition: "pnl_pct_gt", value: 20 },  // Trailing start
    { coin: coin, condition: "pnl_pct_lt", value: -5 }   // Danger zone
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

// Save subscription_id to session
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.subscription_id = SUBSCRIPTION_ID
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
```

### Step 6: Schedule Periodic Scans

```javascript
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 600,  // 10 minutes
  message: "10min scan: Check positions, scan for new momentum plays"
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

// 2. CHECK DRAWDOWN CIRCUIT BREAKER (before anything else!)
const drawdown = await check_drawdown_circuit_breaker(chat_id, accountValue)
if (drawdown.halt) {
  await cleanup_session(chat_id)
  STOP
}

// 3. Check daily loss limit
const daily = await check_daily_loss_limit(chat_id, 'degen')

// 4. Check consecutive losses
const loss_check = await check_consecutive_losses(chat_id)
if (loss_check.stop_24h) {
  // Schedule wake-up in 24h instead
  schedule({ subscription_id: SUBSCRIPTION_ID, delay: 86400, message: "24h cooldown ended" })
  STOP
}

// 5. Manage dynamic stops and partial takes for ALL positions
for (const position of positions) {
  await manage_dynamic_stop(position, 'degen')
  await manage_partial_takes(chat_id, position, 20)  // tp_pct = 20% for degen
}

// 6. Report ALL positions to Telegram
const stats = JSON.parse(await splox_kv_get({ key: `${chat_id}_stats` }) || '{}')
telegram_send_message({
  chat_id: TELEGRAM_CHAT_ID,
  text: `📊 Positions:
${positions.map(p => `• ${p.coin} ${p.direction}: ${p.roe}% ROE`).join('\n')}
Balance: $${balance}
Progress: ${progress}% to target
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
// 1. Determine what happened
const event_type = payload.type
const coin = payload.coin

// 2. If position closed
if (event_type === 'fill' && payload.closedPosition) {
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
    exit_reason: closed.exit_reason  // 'tp', 'sl', 'trailing'
  })

  // Clear partials tracking for this coin
  await clear_partials(chat_id, closed.coin)

  // 3. Check re-entry if trailing stop exit
  if (closed.exit_reason === 'trailing_stop' && closed.pnl_pct > 0) {
    const reentry = await check_reentry_opportunity(closed)
    if (reentry.reentry) {
      // Execute re-entry trade (Steps 3-4)
    }
  }

  // 4. Research new trade if slot available
  if (positions.length < 3) {
    // Go to Step 2: Market Research
  }
}
```

### On Position Alert

Dynamic stops are managed automatically, but on specific alerts:

| Alert | Action |
|-------|--------|
| +5% | Move stop to breakeven |
| +10% | Lock +5% profit |
| +15% | Lock +10% profit |
| +20%+ | Trail 5% behind price |
| -5% | Watch closely, DO NOT move stop down |

### On Schedule (10min scan)

1. Report position status (see above)
2. If < 3 positions AND not in cooldown AND daily limit not hit → Research new trade
3. If 3 positions → Just monitor

### LAST STEP (every event, never skip)

```javascript
// Always re-schedule before ending
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 600,  // 10 minutes
  message: "10min scan"
})

// If schedule fails (subscription expired):
// → Run Step 5-6 again to recreate subscription + schedule
```

**If you don't re-schedule, the agent dies.**

## Cleanup (on stop or target reached)

```javascript
await cleanup_session(chat_id)
// This handles:
// - Closing all positions
// - Cancelling subscriptions
// - Final report with full stats
```

## Position Sizing

```
Account Balance: $X
Position Size: 20-30% of $X (25% default)
Max Leverage: 15-25x (capped at 25x)
Notional Value: Position Size × Leverage

Example ($100 account, 3 positions):
- Per position: $25 margin (25%)
- Leverage: 20x
- Notional per position: $500
- Total exposure: $1,500 (3 positions)
- Max loss per position (10% SL): $2.50
- Max loss all positions: $7.50 (7.5% of account)
```

## Risk Controls

| Control | Value |
|---------|-------|
| Daily Loss Limit | -15% → Stop for day |
| 3 Consecutive Losses | Cooldown 4-6 hours, size -50% |
| 5 Losses in Day | Stop for 24 hours |
| Drawdown -10% | Size reduced 30% |
| Drawdown -15% | Pause 2 hours, size -50% |
| Drawdown -20% | **HALT ALL TRADING** |
| Min R:R | 2:1 (enforced) |

## Notifications

All actions are logged. User sees:
- "🎰 Degen mode started: $100 → Target: $275 (+175%)"
- "Opened 20x LONG on SOL @ $95, catalyst: ETF momentum"
- "🔒 SOL +15%, stop moved to +10%"
- "💰 SOL Partial take #1: 30% closed at +12%"
- "✅ Closed SOL @ $100, P&L: +$15 (+15%)"
- "📊 Progress: $115 / $275 (15% to target) | 3W/1L"
- "⚠️ Drawdown -12%, size reduced"
- "🎉 TARGET REACHED! $100 → $280 (+180%) | 8W/3L"

No questions asked. Execute and report.
