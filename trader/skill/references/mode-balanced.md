# Balanced Mode

Trend Following + Multi-Confirm | Top 30 | 3-7x | 2hr scans

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
trailing: 3% distance after +10% profit
max_positions: 4
scan: 2hr base (1hr volatile, 4hr quiet)
daily_limit: -8%
confidence_min: 7
btc_alignment: REQUIRED

risk_profile: Moderate
  btc_alignment: REQUIRED (hard skip)
  check_funding: true (penalty)
  check_time: true (size reduction)
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

notify('balanced', 'start', { bal: starting, target, pct: target_pct })
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
```

### Step 3b: Pre-Trade Checks

```javascript
const liq = await check_liquidity(coin, margin, 'balanced')
if (!liq.ok) return SKIP

// BTC alignment REQUIRED (hard skip)
const btc = await check_btc_alignment(coin, direction)
if (!btc.aligned) return SKIP

// Funding check (penalty)
const funding = await check_funding_edge(coin, direction)
confidence += funding.confidence_penalty

// Time filter
const time = check_trading_conditions()
let size_mult = time.multiplier

if (confidence < 7) return SKIP
```

### Step 4: Execute

```javascript
hyperliquid_get_positions({})
if (positions.length >= 4) return SKIP

const leverage = Math.min(Math.max(maxLeverage, 3), 5)

hyperliquid_update_leverage({ coin, leverage, is_cross: true })

const price = await hyperliquid_get_price(coin)
const margin = accountValue * 0.10 * size_mult
const size = calculate_size(margin, leverage, price)

const sl_pct = 4, tp_pct = 8
const sl_price = price * (is_buy ? (1 - sl_pct/100) : (1 + sl_pct/100))
const tp_price = price * (is_buy ? (1 + tp_pct/100) : (1 - tp_pct/100))

const result = await place_bracket_order(coin, is_buy, size, price, tp_price, sl_price, 'balanced')

const rr = (tp_pct / sl_pct).toFixed(1)
notify('balanced', 'entry', { dir: is_buy ? 'LONG' : 'SHORT', coin, price, lev: leverage, rr })
```

### Step 5: Subscribe

```javascript
hyperliquid_subscribe_webhook({
  webhook_url,
  coins: [coin],
  events: ["fills", "orders"],
  position_alerts: [
    { coin, condition: "pnl_pct_gt", value: 5 },
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

### Step 6: Schedule (Adaptive)

```javascript
const interval = await get_scan_interval('balanced')
schedule({ subscription_id, delay: interval, message: "balanced scan" })
```

## Event Handling

### On Wake-up (ALWAYS FIRST)

```javascript
hyperliquid_get_balance({})
hyperliquid_get_positions({})

const progress = calculate_progress(accountValue, starting, target)
if (accountValue >= target) {
  notify('balanced', 'target', { start: starting, final: accountValue, ret: progress.progress_pct })
  cleanup()
  return STOP
}

for (const pos of positions) {
  const trail = await check_trailing_stop(pos, 'balanced')
  if (trail.moved) {
    notify('balanced', 'trail', { coin: pos.coin, locked: trail.locked.toFixed(1) })
  }
}

notify('balanced', 'scan', { pos: positions.length, max: 4, bal: accountValue.toFixed(2), progress: progress.progress_pct })
```

### On Trade Close

```javascript
const progress = calculate_progress(accountValue, starting, target)

if (pnl_pct > 0) {
  notify('balanced', 'win', { coin, pnl: pnl_usd.toFixed(2), pct: pnl_pct.toFixed(1), bal: accountValue.toFixed(2), progress: progress.progress_pct })
} else {
  notify('balanced', 'loss', { coin, pnl: Math.abs(pnl_usd).toFixed(2), pct: pnl_pct.toFixed(1), bal: accountValue.toFixed(2) })
}

if (accountValue >= target) {
  notify('balanced', 'target', { start: starting, final: accountValue, ret: progress.progress_pct })
  cleanup()
  return STOP
}
```

### On Position Alert

```
+3%  → breakeven (-0.2%)
+5%  → +2% locked
+8%  → +5% locked
+10% → trail 3% below max
```

### LAST STEP (NEVER SKIP)

```javascript
const interval = await get_scan_interval('balanced')
schedule({ subscription_id, delay: interval, message: "balanced scan" })
```

## Cleanup

```javascript
for (const pos of positions) hyperliquid_market_close({ coin: pos.coin })
cancel_schedule({ schedule_id })
event_unsubscribe({ subscription_id })
hyperliquid_unsubscribe_webhook({})

notify('balanced', 'target', { start: starting, final: accountValue, ret: ((accountValue-starting)/starting*100).toFixed(1) })
```
