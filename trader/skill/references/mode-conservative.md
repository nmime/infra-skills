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
trailing: 2% distance after +6% profit
max_positions: 6
max_margin: 60% (keep 40% cash)
scan: 3d base (1d volatile, 5d quiet)
daily_limit: -5%
confidence_min: 8
btc_alignment: REQUIRED
weekend_trading: DISABLED

risk_profile: Capital Preservation
  btc_alignment: REQUIRED (hard skip)
  funding_check: STRICT (hard skip if against)
  weekend_trading: false
  allowed_coins: BTC, ETH, SOL, BNB, XRP, ADA, AVAX, DOGE, LINK, DOT
```

## Allowed Coins

BTC, ETH, SOL, BNB, XRP, ADA, AVAX, DOGE, LINK, DOT

## Setup

### Step 0: Init Session

```javascript
hyperliquid_get_balance({})
const starting = accountValue
const target = starting * 1.20

notify('conservative', 'start', { bal: starting, target, pct: 20 })
```

### Step 1: Create Webhook

```javascript
const { webhook_id, webhook_url } = await event_create_webhook({ label: "hyperliquid_conservative" })
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

// BTC alignment REQUIRED
const btc = await check_btc_alignment(coin, direction)
if (!btc.aligned) return SKIP

// Funding STRICT (hard skip if against)
const funding = await check_funding_edge(coin, direction)
if (funding.confidence_penalty < 0) return SKIP

// No weekend trading
const time = check_trading_conditions()
if (time.is_weekend) return SKIP
let size_mult = time.multiplier

if (confidence < 8) return SKIP
```

### Step 4: Execute (Small & Safe)

```javascript
hyperliquid_get_positions({})

const margin_used_pct = totalMarginUsed / accountValue
if (margin_used_pct > 0.60) return SKIP
if (positions.length >= 6) return SKIP

const leverage = 2

hyperliquid_update_leverage({ coin, leverage, is_cross: true })

const price = await hyperliquid_get_price(coin)
const margin = accountValue * 0.06 * size_mult
const size = calculate_size(margin, leverage, price)

const sl_pct = 2.5, tp_pct = 5
const sl_price = price * (is_buy ? (1 - sl_pct/100) : (1 + sl_pct/100))
const tp_price = price * (is_buy ? (1 + tp_pct/100) : (1 - tp_pct/100))

const result = await place_bracket_order(coin, is_buy, size, price, tp_price, sl_price, 'conservative')

notify('conservative', 'entry', { dir: is_buy ? 'LONG' : 'SHORT', coin, price, lev: leverage })
```

### Step 5: Subscribe

```javascript
hyperliquid_subscribe_webhook({
  webhook_url,
  coins: [coin],
  events: ["fills", "orders"],
  position_alerts: [
    { coin, condition: "pnl_pct_gt", value: 3 },
    { coin, condition: "pnl_pct_gt", value: 6 },
    { coin, condition: "pnl_pct_lt", value: -1.5 }
  ]
})

const { subscription_id } = await event_subscribe({
  webhook_id,
  timeout: 2592000,
  triggers: [
    { name: "trade_events", filter: "payload.type == 'fill' || payload.type == 'order'", debounce: 10 },
    { name: "position_alerts", filter: "payload.type == 'position_alert'", debounce: 10 }
  ]
})
```

### Step 6: Schedule (Adaptive)

```javascript
const interval = await get_scan_interval('conservative')
schedule({ subscription_id, delay: interval, message: "3-day scan" })
```

## Event Handling

### On Wake-up (ALWAYS FIRST)

```javascript
hyperliquid_get_balance({})
hyperliquid_get_positions({})

for (const pos of positions) {
  const trail = await check_trailing_stop(pos, 'conservative')
  if (trail.moved) {
    notify('conservative', 'trail', { coin: pos.coin, locked: trail.locked.toFixed(1) })
  }
}

const cash_pct = (100 - totalMarginUsed/accountValue*100).toFixed(0)
notify('conservative', 'scan', { pos: positions.length, max: 6, cash: cash_pct })
```

### On Trade Close

```javascript
if (pnl_pct > 0) {
  notify('conservative', 'win', { coin, pnl: pnl_usd.toFixed(2), pct: pnl_pct.toFixed(1) })
} else {
  notify('conservative', 'loss', { coin, pnl: Math.abs(pnl_usd).toFixed(2), pct: pnl_pct.toFixed(1) })
}
```

### On Position Alert

```
+2.5% → breakeven (-0.1%)
+4%   → +2% locked
+6%   → trail 2% below max
```

### LAST STEP (NEVER SKIP)

```javascript
const interval = await get_scan_interval('conservative')
schedule({ subscription_id, delay: interval, message: "3-day scan" })
```

## Cleanup

```javascript
for (const pos of positions) hyperliquid_market_close({ coin: pos.coin })
cancel_schedule({ schedule_id })
event_unsubscribe({ subscription_id })
hyperliquid_unsubscribe_webhook({})

notify('conservative', 'target', { start: starting, final: accountValue, ret: ((accountValue-starting)/starting*100).toFixed(1) })
```
