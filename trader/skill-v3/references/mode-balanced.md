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
risk_per_trade: 2% max (optimal per research)
sl: -4%
tp: +8% (min 2x SL)
max_positions: 4
scan: 2hr base (1hr volatile, 4hr quiet)
daily_limit: -8%
confidence_min: 7

risk_profile: Moderate (strict checks)
  btc_check: REQUIRED (hard skip if misaligned)
  funding_check: penalty -2
  time_filter: size x0.60 off-hours

progressive_trailing:
  +4% profit → 4% trail
  +6% profit → 3% trail
  +10% profit → 2% trail (lock gains)

partial_takes:
  +4% (50% TP) → take 30%
  +6% (75% TP) → take 30%
  remainder → trail
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

await init_session(chat_id, 'balanced', starting, target)

notify('balanced', 'start', { bal: starting, target, pct: target_pct })
```

### Step 1: Create Webhook

```javascript
const { webhook_id, webhook_url } = await event_create_webhook({ label: "hyperliquid_balanced" })

const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.webhook_id = webhook_id
session.webhook_url = webhook_url
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
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

// V3 safety checks
const balance = await hyperliquid_get_balance({})
const dd = await check_drawdown_circuit_breaker(chat_id, balance.accountValue)
if (dd.halt) return STOP
size_mult *= dd.size_multiplier

const loss = await check_consecutive_losses(chat_id)
if (loss.stop_24h) return STOP
if (loss.cooldown) size_mult *= loss.size_multiplier

const daily = await check_daily_loss_limit(chat_id, 'balanced')
if (daily.exceeded) return SKIP

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

const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
session.subscription_id = subscription_id
await splox_kv_set({ key: `${chat_id}_session`, value: JSON.stringify(session) })
```

### Step 6: Schedule (Adaptive)

```javascript
const { interval, reason } = await get_scan_interval('balanced')
const next_scan = format_interval(interval)
schedule({ subscription_id, delay: interval, message: `balanced scan (${reason})` })

notify('balanced', 'scan', { pos: positions.length, max: 4, bal: accountValue.toFixed(2), progress: progress.progress_pct, next_scan })
```

## Event Handling

### On Wake-up (ALWAYS FIRST)

```javascript
hyperliquid_get_balance({})
hyperliquid_get_positions({})

const dd = await check_drawdown_circuit_breaker(chat_id, accountValue)
if (dd.halt) { await cleanup_session(chat_id); return STOP }

const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
const progress = calculate_progress(accountValue, session.starting_balance, session.target_balance)
if (accountValue >= session.target_balance) {
  notify('balanced', 'target', { start: session.starting_balance, final: accountValue, ret: progress.progress_pct })
  await cleanup_session(chat_id)
  return STOP
}

const daily = await check_daily_loss_limit(chat_id, 'balanced')

const loss = await check_consecutive_losses(chat_id)
if (loss.stop_24h) {
  schedule({ subscription_id, delay: 86400, message: "24h cooldown" })
  return STOP
}

for (const pos of positions) {
  const trail = await check_trailing_stop(pos, 'balanced')
  if (trail.moved) {
    notify('balanced', 'trail', { coin: pos.coin, locked: trail.locked.toFixed(1) })
  }
  await manage_partial_takes(chat_id, pos, 8)
}

// notify moved to LAST STEP with next_scan
```

### On Trade Close

```javascript
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
const progress = calculate_progress(accountValue, session.starting_balance, session.target_balance)

await record_trade(chat_id, { coin, direction, entry_price, exit_price, pnl_pct, pnl_usd, exit_reason })
await clear_partials(chat_id, coin)

if (pnl_pct > 0) {
  notify('balanced', 'win', { coin, pnl: pnl_usd.toFixed(2), pct: pnl_pct.toFixed(1), bal: accountValue.toFixed(2), progress: progress.progress_pct })
} else {
  notify('balanced', 'loss', { coin, pnl: Math.abs(pnl_usd).toFixed(2), pct: pnl_pct.toFixed(1), bal: accountValue.toFixed(2) })
}

if (accountValue >= session.target_balance) {
  notify('balanced', 'target', { start: session.starting_balance, final: accountValue, ret: progress.progress_pct })
  await cleanup_session(chat_id)
  return STOP
}
```

### On Position Alert

```
Progressive Trailing:
+4%  → trail 4% (move SL to breakeven)
+6%  → trail 3% (tighten)
+10% → trail 2% (lock gains)

Partials:
+4% (50% TP) → take 30% profit
+6% (75% TP) → take 30% profit
Remainder runs with trail
```

### LAST STEP (NEVER SKIP)

```javascript
const { interval, reason } = await get_scan_interval('balanced')
const next_scan = format_interval(interval)
schedule({ subscription_id, delay: interval, message: `balanced scan (${reason})` })

notify('balanced', 'scan', { pos: positions.length, max: 4, bal: accountValue.toFixed(2), progress: progress.progress_pct, next_scan })
```

## Cleanup

```javascript
await cleanup_session(chat_id)
```
