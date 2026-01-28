# Conservative Mode

**Strategy:** Macro Trends + Capital Preservation + Long-Term Compounding | Full Autonomy | Top 10 Coins Only | KV State Tracking

## Target

**Goal: +20% annually**

Agent runs indefinitely until stopped. Checks progress quarterly.

```
Example:
Starting Balance: $1,000
Annual Target: $1,200 (+20%)

Quarterly milestones:
  Q1: $1,050 (+5%)
  Q2: $1,100 (+10%)
  Q3: $1,150 (+15%)
  Q4: $1,200 (+20%) → Target hit, continue or stop
```

If target hit early → Report success, keep running (compound gains).

## Parameters

| Parameter | Value |
|-----------|-------|
| Target | +20% annually |
| Leverage | 1-2x (near spot) |
| Position Size | 5-8% of account |
| Stop Loss | -2% to -3% |
| Take Profit | +5% to +8% (min 2× SL) |
| Max Positions | 6 (highly diversified) |
| Scan Interval | 3 days (72 hours) |
| Daily Loss Limit | -5% (hard stop) |

## Dynamic Stop-Loss Levels

| Position P&L | Move Stop To | Locked Profit |
|--------------|--------------|---------------|
| +2.5% | Breakeven (-0.1%) | Risk-free |
| +4% | +2% | +2% guaranteed |
| +6%+ | Trail 2% below max | Ride the trend |

## Allowed Coins

Top 10 only: BTC, ETH, SOL, BNB, XRP, ADA, AVAX, DOGE, LINK, DOT

## Setup Flow

### Step 0: Initialize Long-Term Session (KV Storage)

```javascript
// Get starting balance
hyperliquid_get_balance({})
STARTING_BALANCE = accountValue
START_DATE = now()

// Calculate annual target
TARGET_PCT = 20
TARGET_BALANCE = STARTING_BALANCE * 1.20

// Calculate quarterly milestones
Q1_TARGET = STARTING_BALANCE * 1.05  // +5%
Q2_TARGET = STARTING_BALANCE * 1.10  // +10%
Q3_TARGET = STARTING_BALANCE * 1.15  // +15%
Q4_TARGET = STARTING_BALANCE * 1.20  // +20%

// Initialize all KV state
await init_session(chat_id, 'conservative', STARTING_BALANCE, TARGET_BALANCE)

// Store quarterly targets
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.quarterly_targets = { Q1: Q1_TARGET, Q2: Q2_TARGET, Q3: Q3_TARGET, Q4: Q4_TARGET }
session.start_date = START_DATE
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })

// Notify
telegram_send_message({
  text: `🛡️ *Conservative mode started*
Starting: $${STARTING_BALANCE}
Annual target: $${TARGET_BALANCE} (+20%)
Scanning every 3 days
This is a marathon, not a sprint.`
})
```

### Step 1: Create Event Hub Webhook

```javascript
event_create_webhook({
  label: "hyperliquid_conservative"
})
// Save webhook_id and webhook_url to session
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.webhook_id = webhook_id
session.webhook_url = webhook_url
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
```

### Step 2: Market Research (Macro Focus)

Always include chat_id as a prefix
```javascript
market_deepresearch({
  context_memory_id: "{chat_id}_conservative_session",
  message: `Quick scan (1-2 min max): Find the best momentum trade RIGHT NOW on Hyperliquid perpetuals.

Analyze:
1. BTC weekly trend - Is the macro direction clear?
2. Top 10 coins only: BTC, ETH, SOL, BNB, XRP, ADA, AVAX, DOGE, LINK, DOT
3. Weekly/monthly chart trends
4. Are we in accumulation, markup, distribution, or markdown phase?
5. Any major macro events coming? (halving, ETF, regulation)

Rules:
- Only LONG in uptrends, only SHORT in confirmed downtrends
- Follow BTC - don't fight the king
- Skip if unclear - most scans should result in no trade
- Confidence must be 8+ to trade

I need:
- Market assessment (bullish / bearish / unclear)
- If clear: Up to 2 trade ideas from top 10
- If unclear: Recommend waiting

Current positions: [LIST CURRENT POSITIONS]
Cash available: [CASH %]

Remember: It's OK to do nothing. Capital preservation first.`
})
```

**Trade Requirement:**
- Confidence >= 8
- Macro trend clear
- BTC aligned
- Most scans = no trade (expected)

### Step 3: Validate Coins

```javascript
// Only allow top 10
ALLOWED_COINS = ["BTC", "ETH", "SOL", "BNB", "XRP", "ADA", "AVAX", "DOGE", "LINK", "DOT"]

if (!ALLOWED_COINS.includes(coin)) {
  "Coin not in top 10, skipping"
  SKIP
}

hyperliquid_get_meta({ coin: "COIN" })
hyperliquid_get_all_prices({ coins: ["COIN"] })
hyperliquid_get_funding_rates({ coin: "COIN" })
```

### Step 3b: Pre-Trade Checks (Strict)

```javascript
// 1. Check liquidity & spread (strictest requirements)
const liquidity = await check_liquidity(COIN, MARGIN, 'conservative')
if (!liquidity.ok) SKIP

// 2. Check BTC alignment (REQUIRED - no exceptions)
const btc_check = await check_btc_alignment(COIN, DIRECTION)
if (!btc_check.aligned) {
  telegram_send_message({ text: `⚠️ ${COIN} against BTC trend. Conservative mode requires alignment. Skipping.` })
  SKIP
}

// 3. Check time conditions
const time_check = check_trading_conditions()
if (time_check.is_weekend) {
  telegram_send_message({ text: `⚠️ Weekend - Conservative mode pauses weekend trading.` })
  SKIP  // Conservative: no weekend trading
}
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
const daily = await check_daily_loss_limit(chat_id, 'conservative')
if (daily.exceeded) SKIP

// 7. Final confidence check (must be 8+)
if (CONFIDENCE < 8) SKIP
```

### Step 4: Execute Trade (Small & Safe)

```javascript
// Check portfolio constraints
hyperliquid_get_positions({})

// Rules:
// - Max 6 positions
// - Max 60% total margin (keep 40% cash)
// - Max 10% per position

current_margin_pct = totalMarginUsed / accountValue
if (current_margin_pct > 0.60) {
  "Portfolio at max allocation (60%), keeping cash reserve"
  SKIP
}

if (positions.length >= 6) {
  "Max positions reached (6), waiting for exits"
  SKIP
}

// Set low leverage (1-2x)
LEVERAGE = 2

hyperliquid_update_leverage({
  coin: "COIN",
  leverage: LEVERAGE,
  is_cross: true
})

// Small position size with multipliers
let MARGIN = accountValue * 0.06
MARGIN *= SIZE_MULTIPLIER

// Calculate TP/SL with enforced 2:1 minimum R:R
const SL_PCT = 2.5
const TP_PCT = Math.max(SL_PCT * 2, 5)

const SL_PRICE = ENTRY_PRICE * (is_buy ? (1 - SL_PCT/100) : (1 + SL_PCT/100))
const TP_PRICE = ENTRY_PRICE * (is_buy ? (1 + TP_PCT/100) : (1 - TP_PCT/100))

// Use slippage-protected order (tightest tolerance)
const entry_result = await place_protected_order(COIN, is_buy, POSITION_SIZE, 'conservative')

// Place TP/SL after entry
hyperliquid_place_order({ coin: "COIN", order_type: "take_profit", trigger_price: TP_PRICE, reduce_only: true })
hyperliquid_place_order({ coin: "COIN", order_type: "stop_loss", trigger_price: SL_PRICE, reduce_only: true })
```

### Step 5: Setup Monitoring

```javascript
// Subscribe for all positions
// Note: If multiple coins, duplicate this call for each coin
hyperliquid_subscribe_webhook({
  webhook_url: WEBHOOK_URL,
  coins: ["COIN"],
  events: ["fills", "orders"],
  position_alerts: [
    { coin: "COIN", condition: "pnl_pct_gt", value: 2.5 },  // Breakeven trigger
    { coin: "COIN", condition: "pnl_pct_gt", value: 4 },    // +2% lock trigger
    { coin: "COIN", condition: "pnl_pct_gt", value: 6 },    // Trailing start
    { coin: "COIN", condition: "pnl_pct_lt", value: -1.5 }  // Early warning
  ]
})

// Subscribe to Event Hub
event_subscribe({
  webhook_id: WEBHOOK_ID,
  timeout: 2592000,  // 30 days (max)
  triggers: [
    { name: "trade_events", filter: "payload.type == 'fill' || payload.type == 'order'", debounce: 10 },
    { name: "position_alerts", filter: "payload.type == 'position_alert'", debounce: 10 }
  ]
})

// Save subscription_id to session
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.subscription_id = SUBSCRIPTION_ID
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
```

### Step 6: Schedule 3-Day Scans

```javascript
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 259200,  // 3 days = 72 hours
  message: "3-day scan: Check macro conditions, review portfolio"
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
const daily = await check_daily_loss_limit(chat_id, 'conservative')

// 4. Check consecutive losses
const loss_check = await check_consecutive_losses(chat_id)
if (loss_check.stop_24h) {
  schedule({ subscription_id: SUBSCRIPTION_ID, delay: 86400, message: "24h cooldown ended" })
  STOP
}

// 5. Manage dynamic stops and partial takes
for (const position of positions) {
  await manage_dynamic_stop(position, 'conservative')
  await manage_partial_takes(chat_id, position, 5)  // tp_pct = 5% for conservative
}

// 6. Get stats and report to Telegram
const stats = JSON.parse(await splox_kv_get({ key: `${chat_id}_stats` }) || '{}')
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }) || '{}')

telegram_send_message({
  chat_id: TELEGRAM_CHAT_ID,
  text: `📊 3-Day Report:
Positions: ${positions.length}/6
Cash: ${cashPct}%
Total P&L: ${totalPnl}%
Annual target: $${session.target_balance} (+20%)
Progress: ${progress}%
Stats: ${stats.wins || 0}W/${stats.losses || 0}L`
})

// 7. Check quarterly milestone
await check_quarterly_progress(chat_id, accountValue)
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
      // Execute re-entry
    }
  }

  // Position closed → slot opens (don't rush to fill)
}
```

### On Position Alert

| Alert | Action |
|-------|--------|
| +2.5% | Move stop to breakeven |
| +4% | Lock +2% profit |
| +6%+ | Trail 2% behind price |
| -1.5% | Review, consider early exit |

### On Schedule (3-day scan)

1. Report portfolio status (see above)
2. Check quarterly milestone (Q1=90d, Q2=180d, Q3=270d, Q4=365d)
3. If < 6 positions AND cash > 40% AND confidence 8+ → Maybe trade
4. If unclear → Skip (that's fine for conservative)

### Quarterly Progress Check

```javascript
async function check_quarterly_progress(chat_id, current_balance) {
  const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
  const stats = JSON.parse(await splox_kv_get({ key: `${chat_id}_stats` }) || '{}')

  const days_elapsed = (Date.now() - session.start_date) / (1000 * 60 * 60 * 24)
  const quarter = Math.ceil(days_elapsed / 90)

  if (quarter <= 4 && days_elapsed % 90 < 3) {  // Within 3 days of quarter end
    const target = session.quarterly_targets[`Q${quarter}`]
    const on_track = current_balance >= target

    telegram_send_message({
      chat_id: TELEGRAM_CHAT_ID,
      text: `📅 *QUARTERLY REVIEW Q${quarter}*
Period: ${quarter * 90} days
Started: $${session.starting_balance}
Current: $${current_balance.toFixed(2)} (${((current_balance/session.starting_balance - 1) * 100).toFixed(1)}%)
Target pace: +${quarter * 5}% → ${on_track ? 'Ahead ✓' : 'Behind ✗'}

Trades this quarter:
  • ${stats.total_trades || 0} total trades
  • ${stats.wins || 0} wins, ${stats.losses || 0} losses
  • Win rate: ${stats.total_trades ? ((stats.wins/stats.total_trades)*100).toFixed(0) : 0}%

Assessment: ${on_track ? 'On track. Continue strategy.' : 'Behind pace. Review approach.'}
Next review: Q${Math.min(quarter + 1, 4)}`
    })
  }
}
```

### LAST STEP (every event, never skip)

```javascript
// Always re-schedule before ending
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 259200,  // 3 days
  message: "3-day scan"
})

// If schedule fails (subscription expired):
// → Run Step 5-6 again to recreate subscription + schedule
```

**If you don't re-schedule, the agent dies.**

## Cleanup (on manual stop)

```javascript
await cleanup_session(chat_id)

// Final report includes:
// - Duration: X months
// - Starting: $Y
// - Final: $Z
// - Return: +W%
// - Annualized: +V%
// - Trades: N total (W wins, L losses)
```

## Position Sizing

```
Account Balance: $X
Max Positions: 6
Per Position: 5-8% margin (6% default)
Leverage: 1-2x (2x default)
Cash Reserve: Always keep 40%+

Example ($1,000 account):
- Per position: $60 margin (6%)
- Leverage: 2x
- Notional per position: $120
- Max 6 positions = $360 margin (36%)
- Cash reserve: $640 (64%)
- Max loss per position (2.5% SL): $1.50
- Max loss all positions: $9 (0.9% of account)
```

## Risk Controls

| Control | Value |
|---------|-------|
| Daily Loss Limit | -5% → Stop for day |
| 3 Consecutive Losses | Cooldown 4-6 hours, size -50% |
| 5 Losses in Day | Stop for 24 hours |
| Drawdown -10% | Size reduced 30% |
| Drawdown -15% | Pause 2 hours, size -50% |
| Drawdown -20% | **HALT ALL TRADING** |
| Min R:R | 2:1 (enforced) |
| Weekend Trading | **DISABLED** |

## Notifications

- "🛡️ Conservative: ${STARTING} → Annual: ${TARGET} (+20%) | Scan: 3d"
- "🔍 Market unclear, staying cash | Next: 3d"
- "🎯 {DIRECTION} {COIN} @ ${ENTRY} | {LEV}x | Conf: {CONF}/10 | Next: 3d"
- "📊 Report: ${BALANCE} (+{PNL_PCT}%) | Pos: {POS}/{MAX} | {W}W/{L}L | Next: 3d"
- "💰 {COIN} Partial #1: 30% at +{PNL}%"
- "✅ WIN +${PNL} | {W}W/{L}L | Next: 3d"
- "❌ LOSS ${PNL} | Next: 3d"
- "📅 Q{Q} Review: +{PCT}% | {ON_TRACK}"
- "🎉 ANNUAL TARGET! ${STARTING} → ${FINAL} (+{RETURN}%) | {W}W/{L}L"

Patient. Protected. Compounding.
