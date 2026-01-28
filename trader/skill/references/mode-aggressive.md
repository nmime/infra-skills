# Aggressive Mode

Momentum + Trend | Top 50 Coins | 5-15x | 20min scans

## Config

```
target: +50% to +100% based on account size
  <$100: +100%
  $100-1000: +75%
  >$1000: +50%

leverage: 5-15x (max 15x)
position: 15% of account
sl: -6%
tp: +12% (min 2x SL)
max_positions: 3
scan: 20min
daily_limit: -10%
confidence_min: 6
```

## Dynamic Stops

```
+5%  → breakeven (-0.3%)
+8%  → +4% locked
+12% → +8% locked
+15% → trail 4% below max
```

## Setup

### Step 0: Init Session

```javascript
hyperliquid_get_balance({})
const starting = accountValue

let target_pct = 100
if (starting >= 1000) target_pct = 50
else if (starting >= 100) target_pct = 75

const target = starting * (1 + target_pct / 100)

telegram_send_message({ text: `🚀 Aggressive | $${starting} → $${target} (+${target_pct}%) | 20min` })
```

### Step 1: Create Webhook

```javascript
const { webhook_id, webhook_url } = await event_create_webhook({ label: "hyperliquid_aggressive" })
```

### Step 2: Research

```javascript
market_deepresearch({
  context_memory_id: `${chat_id}_aggressive_session`,
  message: `Find best momentum trade on Hyperliquid perps.
1. Top 50 coins only
2. Clear trend direction
3. Recent catalyst preferred
4. Check funding rates

Need: coin, direction, why, confidence (6+ to trade).`
})
```

### Step 3: Validate

```javascript
hyperliquid_get_meta({ coin })
hyperliquid_get_all_prices({ coins: [coin] })
hyperliquid_get_funding_rates({ coin })

if (maxLeverage < 10) return SKIP // need 10x min
```

### Step 3b: Pre-Trade Checks

```javascript
const liq = await check_liquidity(coin, margin, 'aggressive')
if (!liq.ok) return SKIP

const btc = await check_btc_alignment(coin, direction)
confidence += btc.confidence_penalty

const funding = await check_funding_edge(coin, direction)
confidence += funding.confidence_penalty

const time = check_trading_conditions()
let size_mult = time.multiplier

if (confidence < 6) {
  telegram_send_message({ text: `⚠️ ${coin} conf ${confidence}/10, skip` })
  return SKIP
}
```

### Step 4: Execute

```javascript
const leverage = Math.min(Math.max(maxLeverage, 5), 15)

hyperliquid_update_leverage({ coin, leverage, is_cross: true })

let margin = accountValue * 0.15 * size_mult
const sl_pct = 6, tp_pct = 12
const price = await hyperliquid_get_price(coin)
const size = calculate_size(margin, leverage, price)

const sl_price = price * (is_buy ? (1 - sl_pct/100) : (1 + sl_pct/100))
const tp_price = price * (is_buy ? (1 + tp_pct/100) : (1 - tp_pct/100))

const result = await place_protected_order(coin, is_buy, size, 'aggressive')

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
    { coin: c, condition: "pnl_pct_gt", value: 8 },
    { coin: c, condition: "pnl_pct_gt", value: 12 },
    { coin: c, condition: "pnl_pct_gt", value: 15 },
    { coin: c, condition: "pnl_pct_lt", value: -4 }
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
schedule({ subscription_id, delay: 1200, message: "20min scan" })
```

## Event Handling

### On Wake-up (ALWAYS FIRST)

```javascript
hyperliquid_get_balance({})
hyperliquid_get_positions({})

for (const pos of positions) {
  await manage_dynamic_stop(pos, 'aggressive')
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
+5%  → breakeven
+8%  → +4% locked
+12% → +8% locked
+15% → trail 4%
-4%  → watch only
```

### LAST STEP (NEVER SKIP)

```javascript
schedule({ subscription_id, delay: 1200, message: "20min scan" })
```

## Cleanup

```javascript
for (const pos of positions) hyperliquid_market_close({ coin: pos.coin })
cancel_schedule({ schedule_id })
event_unsubscribe({ subscription_id })
hyperliquid_unsubscribe_webhook({})
telegram_send_message({ text: `Session ended: $${starting} → $${final} (${pnl_pct}%)` })
```
