# Degen Mode

Momentum + News | Any Coin | Max Leverage | 2hr scans

## Config

```
target: +100% to +300% based on account size
  <$100: +250%
  $100-1000: +175%
  >$1000: +125%

leverage: 15-25x (max 25x)
position: 25% of account
risk_per_trade: 5% max
sl: -10%
tp: +20% (min 2x SL)
max_positions: 3
scan: 10min base (5min volatile, 20min quiet)
daily_limit: -15%
confidence_min: 5

risk_profile: High Risk (NOT YOLO - all checks apply)
  btc_check: penalty -1 (not skip)
  funding_check: penalty -1 (not skip)
  time_filter: size x0.85 off-hours

progressive_trailing:
  +10% profit → 8% trail (breathing room)
  +15% profit → 5% trail (tightening)
  +20% profit → 3% trail (lock gains)

partial_takes:
  50% of TP (+10%) → take 30%
  75% of TP (+15%) → take 30%
  remainder → runs with trail
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

notify('degen', 'start', { bal: starting, target, pct: target_pct })
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

Need: coin, direction, why, confidence (1-10). Shitcoins welcome. YOLO plays ok.`
})
```

### Step 3: Validate

```javascript
hyperliquid_get_meta({ coin })
hyperliquid_get_all_prices({ coins: [coin] })
hyperliquid_get_funding_rates({ coin })
```

### Step 3b: Pre-Trade Checks (ALL checks apply - smaller penalties)

```javascript
// Liquidity check (required for all modes)
const liq = await check_liquidity(coin, margin, 'degen')
if (!liq.ok) return SKIP

// BTC alignment (penalty, not skip - degen takes more risk)
const btc = await check_btc_alignment(coin, direction)
confidence += btc.aligned ? 0 : -1  // Small penalty

// Funding rate (penalty, not skip)
const funding = await check_funding_edge(coin, direction)
confidence += funding.confidence_penalty > 0 ? 0 : -1

// Time filter (size reduction, not skip)
const time = check_trading_conditions()
let size_mult = time.is_weekend || time.is_asia_night ? 0.85 : 1.0

// V3 safety checks (always apply)
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
  notify('degen', 'scan', { pos: positions.length, max: 3, bal: accountValue })
  return SKIP
}
```

### Step 4: Execute

```javascript
hyperliquid_get_positions({})
if (positions.length >= 3) return SKIP

// Get params based on confidence
const params = select_params('degen', confidence)
const leverage = Math.min(maxLeverage, 25)

hyperliquid_update_leverage({ coin, leverage, is_cross: true })

const price = await hyperliquid_get_price(coin)
const margin = accountValue * 0.25 * size_mult
const size = calculate_size(margin, leverage, price)

const sl_pct = 10, tp_pct = 20
const sl_price = price * (is_buy ? (1 - sl_pct/100) : (1 + sl_pct/100))
const tp_price = price * (is_buy ? (1 + tp_pct/100) : (1 - tp_pct/100))

// Atomic bracket order
const result = await place_bracket_order(coin, is_buy, size, price, tp_price, sl_price, 'degen')

notify('degen', 'entry', { dir: is_buy ? 'LONG' : 'SHORT', coin, price, lev: leverage, tp: tp_price, sl: sl_price })
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
    { coin: c, condition: "pnl_pct_gt", value: 10 },
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

### Step 6: Schedule (Adaptive)

```javascript
const interval = await get_scan_interval('degen')
schedule({ subscription_id, delay: interval, message: "degen scan" })
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
  notify('degen', 'target', { start: session.starting_balance, final: accountValue, ret: progress.progress_pct })
  await cleanup_session(chat_id)
  return STOP
}

const daily = await check_daily_loss_limit(chat_id, 'degen')

const loss = await check_consecutive_losses(chat_id)
if (loss.stop_24h) {
  schedule({ subscription_id, delay: 86400, message: "24h cooldown" })
  return STOP
}

for (const pos of positions) {
  const trail = await check_trailing_stop(pos, 'degen')
  if (trail.moved) {
    notify('degen', 'trail', { coin: pos.coin, pnl: (pos.unrealizedPnl/pos.marginUsed*100).toFixed(1), locked: trail.locked.toFixed(1) })
  }
  await manage_partial_takes(chat_id, pos, params.tp_pct)
}

notify('degen', 'scan', { pos: positions.length, max: 3, bal: accountValue.toFixed(2) })
```

### On Trade Close

```javascript
const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }))
const progress = calculate_progress(accountValue, session.starting_balance, session.target_balance)

await record_trade(chat_id, { coin, direction, entry_price, exit_price, pnl_pct, pnl_usd, exit_reason })
await clear_partials(chat_id, coin)

if (pnl_pct > 0) {
  notify('degen', 'win', { coin, pnl: pnl_usd.toFixed(2), pct: pnl_pct.toFixed(1), bal: accountValue.toFixed(2), progress: progress.progress_pct })
} else {
  notify('degen', 'loss', { coin, pnl: Math.abs(pnl_usd).toFixed(2), pct: pnl_pct.toFixed(1), bal: accountValue.toFixed(2), progress: progress.progress_pct })
}

if (accountValue >= session.target_balance) {
  notify('degen', 'target', { start: session.starting_balance, final: accountValue, ret: progress.progress_pct })
  await cleanup_session(chat_id)
  return STOP
}

if (positions.length < 3) // research new trade
```

### On Position Alert

```
+5%  → move SL to breakeven (-0.3%)
+10% → progressive trail activates (8% distance)
+15% → trail tightens to 5%
+20% → trail tightens to 3% (lock gains)

Partials:
+10% (50% of TP) → take 30% profit
+15% (75% of TP) → take 30% profit
Remainder runs with trail
```

### LAST STEP (NEVER SKIP)

```javascript
const interval = await get_scan_interval('degen')
schedule({ subscription_id, delay: interval, message: "degen scan" })
```

## Cleanup

```javascript
await cleanup_session(chat_id)
```
