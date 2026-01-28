# Aggressive Mode

Momentum + Trend | Top 50 Coins | 10-20x | 1hr scans

## Config

```
target: +50% to +100% based on account size
  <$100: +100%
  $100-1000: +75%
  >$1000: +50%

leverage: 5-15x (max 15x)
position: 15% of account
risk_per_trade: 3% max
sl: -6%
tp: +12% (min 2x SL)
max_positions: 3
scan: 20min base (10min volatile, 40min quiet)
daily_limit: -10%
confidence_min: 6

risk_profile: Moderate-High
  btc_check: penalty -2 (not skip)
  funding_check: penalty -1 (not skip)
  time_filter: size x0.75 off-hours

progressive_trailing:
  +6% profit → 6% trail
  +10% profit → 4% trail
  +15% profit → 3% trail (lock gains)

partial_takes:
  +6% (50% TP) → take 30%
  +9% (75% TP) → take 30%
  remainder → trail
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

notify('aggressive', 'start', { bal: starting, target, pct: target_pct })
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

if (maxLeverage < 10) return SKIP
```

### Step 3b: Pre-Trade Checks

```javascript
const liq = await check_liquidity(coin, margin, 'aggressive')
if (!liq.ok) return SKIP

// BTC check (penalty, not skip)
const btc = await check_btc_alignment(coin, direction)
confidence += btc.confidence_penalty

// Funding check (penalty)
const funding = await check_funding_edge(coin, direction)
confidence += funding.confidence_penalty

// Time filter (size adjustment)
const time = check_trading_conditions()
let size_mult = time.multiplier

if (confidence < 6) return SKIP
```

### Step 4: Execute

```javascript
hyperliquid_get_positions({})
if (positions.length >= 3) return SKIP

const leverage = Math.min(Math.max(maxLeverage, 5), 15)

hyperliquid_update_leverage({ coin, leverage, is_cross: true })

const price = await hyperliquid_get_price(coin)
const margin = accountValue * 0.15 * size_mult
const size = calculate_size(margin, leverage, price)

const sl_pct = 6, tp_pct = 12
const sl_price = price * (is_buy ? (1 - sl_pct/100) : (1 + sl_pct/100))
const tp_price = price * (is_buy ? (1 + tp_pct/100) : (1 - tp_pct/100))

const result = await place_bracket_order(coin, is_buy, size, price, tp_price, sl_price, 'aggressive')

notify('aggressive', 'entry', { dir: is_buy ? 'LONG' : 'SHORT', coin, price, lev: leverage, tp: tp_price, sl: sl_price })
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
    { coin: c, condition: "pnl_pct_gt", value: 8 },
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

### Step 6: Schedule (Adaptive)

```javascript
const { interval, reason } = await get_scan_interval('aggressive')
const next_scan = format_interval(interval)
schedule({ subscription_id, delay: interval, message: `aggressive scan (${reason})` })

notify('aggressive', 'scan', { pos: positions.length, max: 3, bal: accountValue.toFixed(2), next_scan })
```

## Event Handling

### On Wake-up (ALWAYS FIRST)

```javascript
hyperliquid_get_balance({})
hyperliquid_get_positions({})

const progress = calculate_progress(accountValue, starting, target)
if (accountValue >= target) {
  notify('aggressive', 'target', { start: starting, final: accountValue, ret: progress.progress_pct })
  cleanup()
  return STOP
}

for (const pos of positions) {
  const trail = await check_trailing_stop(pos, 'aggressive')
  if (trail.moved) {
    notify('aggressive', 'trail', { coin: pos.coin, pnl: (pos.unrealizedPnl/pos.marginUsed*100).toFixed(1), locked: trail.locked.toFixed(1) })
  }
}

// notify moved to LAST STEP with next_scan
```

### On Trade Close

```javascript
const progress = calculate_progress(accountValue, starting, target)

if (pnl_pct > 0) {
  notify('aggressive', 'win', { coin, pnl: pnl_usd.toFixed(2), pct: pnl_pct.toFixed(1), bal: accountValue.toFixed(2), progress: progress.progress_pct })
} else {
  notify('aggressive', 'loss', { coin, pnl: Math.abs(pnl_usd).toFixed(2), pct: pnl_pct.toFixed(1), bal: accountValue.toFixed(2), progress: progress.progress_pct })
}

if (accountValue >= target) {
  notify('aggressive', 'target', { start: starting, final: accountValue, ret: progress.progress_pct })
  cleanup()
  return STOP
}

if (positions.length < 3) // research new trade
```

### On Position Alert

```
Progressive Trailing (tighten as profit grows):
+6%  → trail 6% (move SL to breakeven)
+10% → trail 4% (tighten)
+15% → trail 3% (lock gains)

Partials:
+6% (50% TP) → take 30% profit
+9% (75% TP) → take 30% profit
Remainder runs with trail
```

### LAST STEP (NEVER SKIP)

```javascript
const { interval, reason } = await get_scan_interval('aggressive')
const next_scan = format_interval(interval)
schedule({ subscription_id, delay: interval, message: `aggressive scan (${reason})` })

notify('aggressive', 'scan', { pos: positions.length, max: 3, bal: accountValue.toFixed(2), next_scan })
```

## Cleanup

```javascript
for (const pos of positions) hyperliquid_market_close({ coin: pos.coin })
cancel_schedule({ schedule_id })
event_unsubscribe({ subscription_id })
hyperliquid_unsubscribe_webhook({})

notify('aggressive', 'target', { start: starting, final: accountValue, ret: ((accountValue-starting)/starting*100).toFixed(1) })
```
