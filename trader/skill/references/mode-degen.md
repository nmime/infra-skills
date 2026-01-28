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

telegram_send_message({ text: `🎰 Degen | $${starting} → $${target} (+${target_pct}%) | 10min` })
```

### Step 1: Create Webhook

```javascript
const { webhook_id, webhook_url } = await event_create_webhook({ label: "hyperliquid_degen" })
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

for (const pos of positions) {
  await manage_dynamic_stop(pos, 'degen')
}

telegram_send_message({
  text: `📊 ${positions.map(p => `${p.coin}: ${p.roe}%`).join(' | ')} | $${balance}`
})

if (accountValue >= target) { cleanup(); return STOP }
```

### On Trade Close

```javascript
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
for (const pos of positions) hyperliquid_market_close({ coin: pos.coin })
cancel_schedule({ schedule_id })
event_unsubscribe({ subscription_id })
hyperliquid_unsubscribe_webhook({})
telegram_send_message({ text: `Session ended: $${starting} → $${final} (${pnl_pct}%)` })
```
