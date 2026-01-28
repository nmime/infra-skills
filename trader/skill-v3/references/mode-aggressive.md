# Aggressive Mode

**Strategy:** Momentum + Trend Confirmation | Full Autonomy | Top 50 Coins | KV State Tracking

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

### Step 0: Initialize Session (KV Storage)

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

// Initialize all KV state
await init_session(chat_id, 'aggressive', STARTING_BALANCE, TARGET_BALANCE)

// Notify
telegram_send_message({
  text: `🚀 *Aggressive mode started*
Balance: $${STARTING_BALANCE}
Target: $${TARGET_BALANCE} (+${TARGET_PCT}%)
Scan: every 20min`
})
```

### Step 1: Create Event Hub Webhook

```javascript
event_create_webhook({
  label: "hyperliquid_aggressive"
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
const daily = await check_daily_loss_limit(chat_id, 'aggressive')
if (daily.exceeded) SKIP

// 7. Final confidence check
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
MARGIN *= SIZE_MULTIPLIER

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

// Save subscription_id to session
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.subscription_id = SUBSCRIPTION_ID
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
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
const drawdown = await check_drawdown_circuit_breaker(chat_id, accountValue)
if (drawdown.halt) {
  await cleanup_session(chat_id)
  STOP
}

// 3. Check daily loss limit
const daily = await check_daily_loss_limit(chat_id, 'aggressive')

// 4. Check consecutive losses
const loss_check = await check_consecutive_losses(chat_id)
if (loss_check.stop_24h) {
  schedule({ subscription_id: SUBSCRIPTION_ID, delay: 86400, message: "24h cooldown ended" })
  STOP
}

// 5. Manage dynamic stops and partial takes for ALL positions
for (const position of positions) {
  await manage_dynamic_stop(position, 'aggressive')
  await manage_partial_takes(chat_id, position, 12)  // tp_pct = 12% for aggressive
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
// If position closed
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

  // Research new trade if slot available
  if (positions.length < 3) {
    // Go to Step 2
  }
}
```

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
await cleanup_session(chat_id)
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
| Drawdown -10% | Size reduced 30% |
| Drawdown -15% | Pause 2 hours, size -50% |
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
- "🔒 SOL +8%, stop moved to +4%"
- "💰 SOL Partial #1: 30% closed at +6%"
- "✅ Closed SOL @ $130, P&L: +$18 (+15.6%)"
- "📊 Progress: $118 / $175 (24% to target) | 4W/2L"
- "🎉 TARGET REACHED! $100 → $178 (+78%) | 7W/3L"

No questions asked. Execute and report.
