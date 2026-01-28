# Conservative Mode

Macro Trends + Capital Preservation | Top 10 Only | 1-2x | 3-day scans

## Config

```
target: +20% annually
  Q1: +5%, Q2: +10%, Q3: +15%, Q4: +20%

leverage: 1-2x
position: 6% of account
sl: -2.5%
tp: +5% (min 2x SL)
max_positions: 6
max_margin: 60% (keep 40% cash)
scan: 3 days
daily_limit: -5%
confidence_min: 8
btc_alignment: REQUIRED
weekend_trading: DISABLED
```

## Allowed Coins

BTC, ETH, SOL, BNB, XRP, ADA, AVAX, DOGE, LINK, DOT

## Dynamic Stops

```
+2.5% → breakeven (-0.1%)
+4%   → +2% locked
+6%   → trail 2% below max
```

## Setup

### Step 0: Init Session

```javascript
hyperliquid_get_balance({})
const starting = accountValue
const target = starting * 1.20

await init_session(chat_id, 'conservative', starting, target)

// Store quarterly targets
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.quarterly_targets = {
  Q1: starting * 1.05,
  Q2: starting * 1.10,
  Q3: starting * 1.15,
  Q4: starting * 1.20
}
session.start_date = Date.now()
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })

telegram_send_message({ text: `🛡️ Conservative | $${starting} → $${target} (+20%/year) | 3d` })
```

### Step 1: Create Webhook

```javascript
const { webhook_id, webhook_url } = await event_create_webhook({ label: "hyperliquid_conservative" })

const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.webhook_id = webhook_id
session.webhook_url = webhook_url
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
```

### Step 2: Research (Macro Focus)

```javascript
market_deepresearch({
  context_memory_id: `${chat_id}_conservative_session`,
  message: `Macro scan for Hyperliquid:
1. BTC weekly trend - clear direction?
2. Top 10 only: BTC, ETH, SOL, BNB, XRP, ADA, AVAX, DOGE, LINK, DOT
3. Weekly/monthly charts
4. Market phase: accumulation/markup/distribution/markdown?
5. Major events coming?

Rules:
- LONG only in uptrends
- SHORT only in confirmed downtrends
- Follow BTC
- Skip if unclear (most scans = no trade)
- Confidence 8+ to trade

Need: market assessment, up to 2 ideas from top 10 OR recommend waiting.`
})
```

### Step 3: Validate

```javascript
const ALLOWED = ["BTC", "ETH", "SOL", "BNB", "XRP", "ADA", "AVAX", "DOGE", "LINK", "DOT"]
if (!ALLOWED.includes(coin)) return SKIP

hyperliquid_get_meta({ coin })
hyperliquid_get_all_prices({ coins: [coin] })
hyperliquid_get_funding_rates({ coin })
```

### Step 3b: Pre-Trade Checks (STRICT)

```javascript
const liq = await check_liquidity(coin, margin, 'conservative')
if (!liq.ok) return SKIP

// BTC alignment REQUIRED - no exceptions
const btc = await check_btc_alignment(coin, direction)
if (!btc.aligned) {
  telegram_send_message({ text: `⚠️ ${coin} vs BTC, conservative requires alignment, skip` })
  return SKIP
}

// Check funding - skip if extreme against us (conservative is strict)
const funding = await check_funding_edge(coin, direction)
if (funding.confidence_penalty < 0) {
  telegram_send_message({ text: `⚠️ ${coin} funding against ${direction}, conservative skips` })
  return SKIP
}

// No weekend trading
const time = check_trading_conditions()
if (time.is_weekend) {
  telegram_send_message({ text: `⚠️ Weekend, conservative pauses` })
  return SKIP
}
let size_mult = time.multiplier

const balance = await hyperliquid_get_balance({})
const dd = await check_drawdown_circuit_breaker(chat_id, balance.accountValue)
if (dd.halt) return STOP
size_mult *= dd.size_multiplier

const loss = await check_consecutive_losses(chat_id)
if (loss.stop_24h) return STOP
if (loss.cooldown) size_mult *= loss.size_multiplier

const daily = await check_daily_loss_limit(chat_id, 'conservative')
if (daily.exceeded) return SKIP

if (confidence < 8) return SKIP
```

### Step 4: Execute (Small & Safe)

```javascript
hyperliquid_get_positions({})

// Max 6 positions, max 60% margin
const margin_used_pct = totalMarginUsed / accountValue
if (margin_used_pct > 0.60) {
  telegram_send_message({ text: `Portfolio at 60%, keeping cash` })
  return SKIP
}
if (positions.length >= 6) return SKIP

const leverage = 2

hyperliquid_update_leverage({ coin, leverage, is_cross: true })

let margin = accountValue * 0.06 * size_mult
const sl_pct = 2.5, tp_pct = 5
const price = await hyperliquid_get_price(coin)
const size = calculate_size(margin, leverage, price)

const sl_price = price * (is_buy ? (1 - sl_pct/100) : (1 + sl_pct/100))
const tp_price = price * (is_buy ? (1 + tp_pct/100) : (1 - tp_pct/100))

const result = await place_protected_order(coin, is_buy, size, 'conservative')

hyperliquid_place_order({ coin, order_type: "take_profit", trigger_price: tp_price, reduce_only: true })
hyperliquid_place_order({ coin, order_type: "stop_loss", trigger_price: sl_price, reduce_only: true })
```

### Step 5: Subscribe

```javascript
hyperliquid_subscribe_webhook({
  webhook_url,
  coins: [coin],
  events: ["fills", "orders"],
  position_alerts: [
    { coin, condition: "pnl_pct_gt", value: 2.5 },
    { coin, condition: "pnl_pct_gt", value: 4 },
    { coin, condition: "pnl_pct_gt", value: 6 },
    { coin, condition: "pnl_pct_lt", value: -1.5 }
  ]
})

const { subscription_id } = await event_subscribe({
  webhook_id,
  timeout: 2592000, // 30 days
  triggers: [
    { name: "trade_events", filter: "payload.type == 'fill' || payload.type == 'order'", debounce: 10 },
    { name: "position_alerts", filter: "payload.type == 'position_alert'", debounce: 10 }
  ]
})

const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.subscription_id = subscription_id
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
```

### Step 6: Schedule

```javascript
schedule({ subscription_id, delay: 259200, message: "3-day scan" }) // 72 hours
```

## Event Handling

### On Wake-up (ALWAYS FIRST)

```javascript
hyperliquid_get_balance({})
hyperliquid_get_positions({})

const dd = await check_drawdown_circuit_breaker(chat_id, accountValue)
if (dd.halt) { await cleanup_session(chat_id); return STOP }

const daily = await check_daily_loss_limit(chat_id, 'conservative')

const loss = await check_consecutive_losses(chat_id)
if (loss.stop_24h) {
  schedule({ subscription_id, delay: 86400, message: "24h cooldown" })
  return STOP
}

for (const pos of positions) {
  await manage_dynamic_stop(pos, 'conservative')
  await manage_partial_takes(chat_id, pos, 5)
}

const stats = JSON.parse(await splox_kv_get({ key: `${chat_id}_stats` }) || '{}')
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }) || '{}')

telegram_send_message({
  text: `📊 3-Day | ${positions.length}/6 | Cash ${(100 - totalMarginUsed/accountValue*100).toFixed(0)}% | ${stats.wins}W/${stats.losses}L`
})

if (accountValue >= session.target_balance) { await cleanup_session(chat_id); return STOP }

// Quarterly check
await check_quarterly_progress(chat_id, accountValue)
```

### Quarterly Progress

```javascript
async function check_quarterly_progress(chat_id, balance) {
  const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
  const stats = JSON.parse(await splox_kv_get({ key: `${chat_id}_stats` }) || '{}')

  const days = (Date.now() - session.start_date) / 86400000
  const quarter = Math.ceil(days / 90)

  if (quarter <= 4 && days % 90 < 3) {
    const target = session.quarterly_targets[`Q${quarter}`]
    const on_track = balance >= target

    telegram_send_message({
      text: `📅 Q${quarter} Review | $${balance.toFixed(0)} vs $${target.toFixed(0)} | ${on_track ? '✓' : '✗'} | ${stats.wins}W/${stats.losses}L`
    })
  }
}
```

### On Trade Close

```javascript
await record_trade(chat_id, { coin, direction, entry_price, exit_price, pnl_pct, pnl_usd, exit_reason })
await clear_partials(chat_id, coin)

if (exit_reason === 'trailing_stop' && pnl_pct > 0) {
  const re = await check_reentry_opportunity(closed)
  if (re.reentry) // re-enter
}
```

### On Position Alert

```
+2.5% → breakeven
+4%   → +2% locked
+6%   → trail 2%
-1.5% → review, consider early exit
```

### LAST STEP (NEVER SKIP)

```javascript
schedule({ subscription_id, delay: 259200, message: "3-day scan" })
```

## Cleanup

```javascript
await cleanup_session(chat_id)
```
