# Balanced Mode

Trend Following + Multi-Confirm | Top 30 | 3-5x | 2hr scans

## Config

```
target: +25% to +50% based on account size
  <$100: +50%
  $100-1000: +40%
  >$1000: +25%

leverage: 3-5x
position: 10% of account
sl: -4%
tp: +8% (min 2x SL)
max_positions: 4
scan: 2hr
daily_limit: -8%
confidence_min: 7
btc_alignment: REQUIRED
```

## Dynamic Stops

```
+3%  → breakeven (-0.2%)
+5%  → +2% locked
+8%  → +5% locked
+10% → trail 3% below max
```

## Setup

### Step 0: Init Session

```javascript
hyperliquid_get_balance({})
const starting = accountValue

let target_pct = 50
if (starting >= 1000) target_pct = 25
else if (starting >= 100) target_pct = 40

const target = starting * (1 + target_pct / 100)

telegram_send_message({ text: `⚖️ Balanced | $${starting} → $${target} (+${target_pct}%) | 2hr` })
```

### Step 1: Create Webhook

```javascript
const { webhook_id, webhook_url } = await event_create_webhook({ label: "hyperliquid_balanced" })
```

### Step 2: Research

```javascript
market_deepresearch({
  context_memory_id: `${chat_id}_balanced_session`,
  message: `Find best trade on Hyperliquid. Requirements - ALL must be met:
1. TREND: Clear on 4h/daily
2. MOMENTUM: Price confirms trend
3. FUNDING: Not extreme against us
4. CATALYST: Bonus if news supports

Need up to 2 recommendations for diversification:
- Coin (top 30, good liquidity)
- Direction (LONG only uptrend, SHORT only downtrend)
- Confirmations list
- Confidence (7+ to trade)

OK to recommend nothing if unclear.`
})
```

### Step 3: Validate

```javascript
hyperliquid_get_meta({ coin })
hyperliquid_get_all_prices({ coins: [coin] })
hyperliquid_get_funding_rates({ coin })

// Check not already in portfolio
// Funding not extreme
```

### Step 3b: Pre-Trade Checks

```javascript
const liq = await check_liquidity(coin, margin, 'balanced')
if (!liq.ok) return SKIP

// BTC alignment REQUIRED for balanced
const btc = await check_btc_alignment(coin, direction)
if (!btc.aligned) {
  telegram_send_message({ text: `⚠️ ${coin} vs BTC, skip` })
  return SKIP
}

const funding = await check_funding_edge(coin, direction)
confidence += funding.confidence_penalty

const time = check_trading_conditions()
let size_mult = time.multiplier

if (confidence < 7) return SKIP
```

### Step 4: Execute

```javascript
hyperliquid_get_positions({})

// Max 4 positions, max 2 long/2 short
if (positions.length >= 4) return SKIP

const leverage = Math.min(Math.max(maxLeverage, 3), 5)

hyperliquid_update_leverage({ coin, leverage, is_cross: true })

let margin = accountValue * 0.10 * size_mult
const sl_pct = 4, tp_pct = 8
const price = await hyperliquid_get_price(coin)
const size = calculate_size(margin, leverage, price)

const sl_price = price * (is_buy ? (1 - sl_pct/100) : (1 + sl_pct/100))
const tp_price = price * (is_buy ? (1 + tp_pct/100) : (1 - tp_pct/100))

const result = await place_protected_order(coin, is_buy, size, 'balanced')

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
    { coin, condition: "pnl_pct_gt", value: 3 },
    { coin, condition: "pnl_pct_gt", value: 5 },
    { coin, condition: "pnl_pct_gt", value: 8 },
    { coin, condition: "pnl_pct_gt", value: 10 },
    { coin, condition: "pnl_pct_lt", value: -2 }
  ]
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
schedule({ subscription_id, delay: 7200, message: "2hr scan" })
```

## Event Handling

### On Wake-up (ALWAYS FIRST)

```javascript
hyperliquid_get_balance({})
hyperliquid_get_positions({})

for (const pos of positions) {
  await manage_dynamic_stop(pos, 'balanced')
}

telegram_send_message({
  text: `📊 ${positions.length}/4 | ${positions.map(p => `${p.coin}: ${p.pnl_pct}%`).join(' | ')}`
})

if (accountValue >= target) { cleanup(); return STOP }
```

### On Trade Close

```javascript
if (exit_reason === 'trailing_stop' && pnl_pct > 0) {
  const re = await check_reentry_opportunity(closed)
  if (re.reentry) // re-enter
}
```

### On Position Alert

```
+3%  → breakeven
+5%  → +2% locked
+8%  → +5% locked
+10% → trail 3%
-2%  → watch only
```

### LAST STEP (NEVER SKIP)

```javascript
schedule({ subscription_id, delay: 7200, message: "2hr scan" })
```

## Cleanup

```javascript
for (const pos of positions) hyperliquid_market_close({ coin: pos.coin })
cancel_schedule({ schedule_id })
event_unsubscribe({ subscription_id })
hyperliquid_unsubscribe_webhook({})
telegram_send_message({ text: `Session ended: $${starting} → $${final} (${pnl_pct}%)` })
```
