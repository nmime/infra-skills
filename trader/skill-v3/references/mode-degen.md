# Degen Mode

Momentum + News | Any Coin | 15-25x | 10min scans

## Config

```
target: +100% to +300% based on account size
  <$100: +250%
  $100-1000: +175%
  >$1000: +125%

leverage: 15-25x (max 25x)
position: 25% of account
sl: -10%
tp: +20% (min 2x SL)
max_positions: 3
scan: 10min
daily_limit: -15%
confidence_min: 5
```

## Dynamic Stops

```
+5%  → breakeven (-0.3%)
+10% → +5% locked
+15% → +10% locked
+20% → trail 5% below max
```

## Setup

### Step 0: Init Session

```javascript
hyperliquid_get_balance({})
const starting = accountValue

let target_pct = 250
if (starting >= 1000) target_pct = 125
else if (starting >= 100) target_pct = 175

const target = starting * (1 + target_pct / 100)

await init_session(chat_id, 'degen', starting, target)

telegram_send_message({ text: `🎰 Degen | $${starting} → $${target} (+${target_pct}%) | 10min` })
```

### Step 1: Create Webhook

```javascript
const { webhook_id, webhook_url } = await event_create_webhook({ label: "hyperliquid_degen" })

const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.webhook_id = webhook_id
session.webhook_url = webhook_url
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
```

### Step 2: Research

```javascript
market_deepresearch({
  context_memory_id: `${chat_id}_degen_session`,
  message: `Find best momentum trade on Hyperliquid perps. Check:
1. Top movers last 1-4h
2. Breaking news/catalysts
3. Extreme funding rates

Need: coin, direction, why, confidence (1-10). Shitcoins ok.`
})
```

### Step 3: Validate

```javascript
hyperliquid_get_meta({ coin })
hyperliquid_get_all_prices({ coins: [coin] })
hyperliquid_get_funding_rates({ coin })
```

### Step 3b: Pre-Trade Checks

```javascript
const liq = await check_liquidity(coin, margin, 'degen')
if (!liq.ok) return SKIP

const btc = await check_btc_alignment(coin, direction)
confidence += btc.confidence_penalty

const time = check_trading_conditions()
let size_mult = time.multiplier

const balance = await hyperliquid_get_balance({})
const dd = await check_drawdown_circuit_breaker(chat_id, balance.accountValue)
if (dd.halt) return STOP
size_mult *= dd.size_multiplier

const loss = await check_consecutive_losses(chat_id)
if (loss.stop_24h) return STOP
if (loss.cooldown) size_mult *= loss.size_multiplier

const daily = await check_daily_loss_limit(chat_id, 'degen')
if (daily.exceeded) return SKIP

if (confidence < 5) {
  telegram_send_message({ text: `⚠️ ${coin} conf ${confidence}/10, skip` })
  return SKIP
}
```

### Step 4: Execute

```javascript
const leverage = Math.min(maxLeverage, 25)

hyperliquid_update_leverage({ coin, leverage, is_cross: true })

let margin = accountValue * 0.25 * size_mult
const sl_pct = 10, tp_pct = 20

const sl_price = entry * (is_buy ? (1 - sl_pct/100) : (1 + sl_pct/100))
const tp_price = entry * (is_buy ? (1 + tp_pct/100) : (1 - tp_pct/100))

const result = await place_protected_order(coin, is_buy, size, 'degen')
if (result.fill_pct < 0.7) // retry or skip

hyperliquid_place_order({ coin, order_type: "take_profit", trigger_price: tp_price, reduce_only: true })
hyperliquid_place_order({ coin, order_type: "stop_loss", trigger_price: sl_price, reduce_only: true })
```

### Step 5: Subscribe

```javascript
hyperliquid_get_positions({})
const coins = positions.map(p => p.coin)

hyperliquid_subscribe_webhook({
  webhook_url,
  coins,
  events: ["fills", "orders"],
  position_alerts: coins.flatMap(c => [
    { coin: c, condition: "pnl_pct_gt", value: 5 },
    { coin: c, condition: "pnl_pct_gt", value: 10 },
    { coin: c, condition: "pnl_pct_gt", value: 15 },
    { coin: c, condition: "pnl_pct_gt", value: 20 },
    { coin: c, condition: "pnl_pct_lt", value: -5 }
  ])
})

const { subscription_id } = await event_subscribe({
  webhook_id,
  timeout: 86400,
  triggers: [
    { name: "trade_events", filter: "payload.type == 'fill' || payload.type == 'order'", debounce: 5 },
    { name: "position_alerts", filter: "payload.type == 'position_alert'", debounce: 5 }
  ]
})

const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.subscription_id = subscription_id
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
```

### Step 6: Schedule

```javascript
schedule({ subscription_id, delay: 600, message: "10min scan" })
```

## Event Handling

### On Wake-up (ALWAYS FIRST)

```javascript
hyperliquid_get_balance({})
hyperliquid_get_positions({})

const dd = await check_drawdown_circuit_breaker(chat_id, accountValue)
if (dd.halt) { await cleanup_session(chat_id); return STOP }

const daily = await check_daily_loss_limit(chat_id, 'degen')

const loss = await check_consecutive_losses(chat_id)
if (loss.stop_24h) {
  schedule({ subscription_id, delay: 86400, message: "24h cooldown" })
  return STOP
}

for (const pos of positions) {
  await manage_dynamic_stop(pos, 'degen')
  await manage_partial_takes(chat_id, pos, 20)
}

const stats = JSON.parse(await splox_kv_get({ key: `${chat_id}_stats` }) || '{}')
telegram_send_message({
  text: `📊 ${positions.map(p => `${p.coin}: ${p.roe}%`).join(' | ')} | $${balance} | ${stats.wins}W/${stats.losses}L`
})

if (accountValue >= target) { await cleanup_session(chat_id); return STOP }
```

### On Trade Close

```javascript
await record_trade(chat_id, {
  coin, direction, entry_price, exit_price, pnl_pct, pnl_usd, exit_reason
})

await clear_partials(chat_id, coin)

if (exit_reason === 'trailing_stop' && pnl_pct > 0) {
  const re = await check_reentry_opportunity(closed)
  if (re.reentry) // re-enter
}

if (positions.length < 3) // research new trade
```

### On Position Alert

```
+5%  → move stop to breakeven
+10% → lock +5%
+15% → lock +10%
+20% → trail 5%
-5%  → watch, never move stop down
```

### LAST STEP (NEVER SKIP)

```javascript
schedule({ subscription_id, delay: 600, message: "10min scan" })
// If fails, recreate subscription
```

## Cleanup

```javascript
await cleanup_session(chat_id)
```
