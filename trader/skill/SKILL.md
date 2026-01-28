---
name: trader-agent
description: Autonomous AI trading agent for Hyperliquid perpetual futures exchange. Use when user wants to trade crypto, set up trading automation, monitor positions, analyze markets, or manage a Hyperliquid account. Trigger on any trading-related message like "I want to trade", "What's my balance?", "Start trading", "Close positions", etc.
---

# Trading Agent (v2 - Stateless)

Autonomous trading agent for Hyperliquid. No persistent state storage.

## Mode Config

```javascript
const MODE_CONFIG = {
  degen: {
    leverage_min: 15, leverage_max: 25,
    position_pct: 0.25,
    sl_pct: 10, tp_pct: 20,
    confidence_min: 5,
    scan_interval: 600,
    daily_limit: -15,
    slippage: 0.8, max_spread: 0.5
  },
  aggressive: {
    leverage_min: 5, leverage_max: 15,
    position_pct: 0.15,
    sl_pct: 6, tp_pct: 12,
    confidence_min: 6,
    scan_interval: 1200,
    daily_limit: -10,
    slippage: 0.5, max_spread: 0.3
  },
  balanced: {
    leverage_min: 3, leverage_max: 5,
    position_pct: 0.10,
    sl_pct: 4, tp_pct: 8,
    confidence_min: 7,
    scan_interval: 7200,
    daily_limit: -8,
    btc_alignment: 'REQUIRED',
    slippage: 0.3, max_spread: 0.2
  },
  conservative: {
    leverage_min: 1, leverage_max: 2,
    position_pct: 0.06,
    sl_pct: 2.5, tp_pct: 5,
    confidence_min: 8,
    scan_interval: 259200,
    daily_limit: -5,
    btc_alignment: 'REQUIRED',
    weekend_trading: false,
    max_margin: 0.60,
    slippage: 0.2, max_spread: 0.15
  }
}
```

## Setup Flow

### Step 0: Get Telegram Chat ID

```
To send you trade notifications, I need your Telegram chat ID.

How to find it:
1. Message @userinfobot on Telegram
2. It will reply with your chat ID (a number like 123456789)

Your Telegram chat ID:
```

Save as `TELEGRAM_CHAT_ID`. If user says "skip" or "no" → `TELEGRAM_CHAT_ID = null`.

### Step 1: Check Hyperliquid Connection

```javascript
SPLOX_SEARCH_TOOLS(query: "hyperliquid")
// Check is_user_connected field
```

If NOT connected, show:

```
To trade on Hyperliquid, connect an agent wallet:

1. Create Agent Wallet: https://app.hyperliquid.xyz/API → "Create API Wallet"
2. Connect: [connect_link from search results] → paste private key (0x...)

NEVER use main wallet key. Agent wallet can trade but CANNOT withdraw.
```

### Step 2: Verify Account

```javascript
hyperliquid_get_balance({})
hyperliquid_get_positions({})
hyperliquid_get_open_orders({})

// Report
telegram_send_message({
  text: `Account: ${address}
Balance: $${accountValue}
Positions: ${positions.length}
Margin: $${totalMarginUsed}`
})
```

### Step 3: Ask Mode

```
Which trading mode?
1. Conservative - 1-2x, +20%/year, 3d scans
2. Balanced - 3-5x, +25-50%, 2hr scans
3. Aggressive - 5-15x, +50-100%, 20min scans
4. Degen - 15-25x, +100-300%, 10min scans
```

### Step 4: Load Mode File

Read `references/mode-{name}.md` and execute all steps in order.

## Core Functions

### Dynamic Stop-Loss

```javascript
async function manage_dynamic_stop(position, mode) {
  const pnl_pct = position.unrealizedPnl / position.marginUsed * 100
  const entry = position.entryPx
  const direction = position.szi > 0 ? 1 : -1

  let new_stop_pct = null

  if (mode === 'degen') {
    if (pnl_pct >= 20) new_stop_pct = pnl_pct - 5
    else if (pnl_pct >= 15) new_stop_pct = 10
    else if (pnl_pct >= 10) new_stop_pct = 5
    else if (pnl_pct >= 5) new_stop_pct = -0.3
  }

  if (mode === 'aggressive') {
    if (pnl_pct >= 15) new_stop_pct = pnl_pct - 4
    else if (pnl_pct >= 12) new_stop_pct = 8
    else if (pnl_pct >= 8) new_stop_pct = 4
    else if (pnl_pct >= 5) new_stop_pct = -0.3
  }

  if (mode === 'balanced') {
    if (pnl_pct >= 10) new_stop_pct = pnl_pct - 3
    else if (pnl_pct >= 8) new_stop_pct = 5
    else if (pnl_pct >= 5) new_stop_pct = 2
    else if (pnl_pct >= 3) new_stop_pct = -0.2
  }

  if (mode === 'conservative') {
    if (pnl_pct >= 6) new_stop_pct = pnl_pct - 2
    else if (pnl_pct >= 4) new_stop_pct = 2
    else if (pnl_pct >= 2.5) new_stop_pct = -0.1
  }

  if (new_stop_pct !== null) {
    const new_stop_price = entry * (1 + (new_stop_pct / 100) * direction)
    const current_stop = await get_current_stop(position.coin)
    const is_better = direction > 0
      ? new_stop_price > current_stop
      : new_stop_price < current_stop

    if (is_better) {
      await move_stop_loss(position.coin, new_stop_price)
      telegram_send_message({
        text: `🔒 ${position.coin} Stop moved | P&L: +${pnl_pct.toFixed(1)}% | SL: $${new_stop_price.toFixed(4)} (${new_stop_pct > 0 ? '+' : ''}${new_stop_pct.toFixed(1)}% locked)`
      })
    }
  }
}
```

### Re-entry Check

```javascript
async function check_reentry_opportunity(closed_position) {
  if (closed_position.exit_reason !== 'trailing_stop') return null
  if (closed_position.realized_pnl <= 0) return null

  await wait(5 * 60 * 1000)

  const current_price = await hyperliquid_get_price(closed_position.coin)
  const exit_price = closed_position.exit_price
  const was_long = closed_position.direction === 'LONG'

  const price_continued = was_long
    ? current_price > exit_price * 1.015
    : current_price < exit_price * 0.985

  if (price_continued) {
    const research = await quick_trend_check(closed_position.coin)
    if (research.confidence >= 6 && research.direction === closed_position.direction) {
      telegram_send_message({
        text: `🔄 ${closed_position.coin} Re-entry | Exited: $${exit_price.toFixed(4)} (+${closed_position.pnl_pct.toFixed(1)}%) | Current: $${current_price.toFixed(4)}`
      })
      return { reentry: true, coin: closed_position.coin, direction: closed_position.direction }
    }
  }
  return { reentry: false }
}
```

### Liquidity Check

```javascript
async function check_liquidity(coin, position_size_usd, mode) {
  const orderbook = await hyperliquid_get_orderbook({ coin })
  const best_bid = orderbook.bids[0][0]
  const best_ask = orderbook.asks[0][0]
  const spread_pct = ((best_ask - best_bid) / best_bid) * 100

  const MAX_SPREAD = { degen: 0.5, aggressive: 0.3, balanced: 0.2, conservative: 0.15 }

  if (spread_pct > MAX_SPREAD[mode]) {
    telegram_send_message({ text: `⚠️ ${coin} spread ${spread_pct.toFixed(2)}% > ${MAX_SPREAD[mode]}%, skip` })
    return { ok: false, reason: 'spread_too_wide' }
  }

  const bid_depth = orderbook.bids.slice(0, 5).reduce((sum, [price, size]) => sum + price * size, 0)
  const ask_depth = orderbook.asks.slice(0, 5).reduce((sum, [price, size]) => sum + price * size, 0)
  const min_depth = Math.min(bid_depth, ask_depth)

  if (min_depth < position_size_usd * 10) {
    telegram_send_message({ text: `⚠️ ${coin} depth $${min_depth.toFixed(0)} < $${(position_size_usd * 10).toFixed(0)}, skip` })
    return { ok: false, reason: 'low_liquidity' }
  }

  return { ok: true, spread: spread_pct, depth: min_depth }
}
```

### BTC Alignment

```javascript
async function check_btc_alignment(coin, direction) {
  if (coin === 'BTC') return { aligned: true, btc_trend: 'N/A' }

  const btc_prices = await hyperliquid_get_candles({ coin: 'BTC', interval: '4h', limit: 2 })
  const btc_change = (btc_prices[1].close - btc_prices[0].close) / btc_prices[0].close * 100

  let btc_trend = 'NEUTRAL'
  if (btc_change > 1) btc_trend = 'UP'
  if (btc_change < -1) btc_trend = 'DOWN'

  const aligned = (direction === 'LONG' && btc_trend !== 'DOWN') || (direction === 'SHORT' && btc_trend !== 'UP')

  if (!aligned) {
    telegram_send_message({
      text: `⚠️ BTC ${btc_trend} (${btc_change > 0 ? '+' : ''}${btc_change.toFixed(1)}%) vs ${direction} ${coin}, conf -2`
    })
  }

  return { aligned, btc_trend, btc_change, confidence_penalty: aligned ? 0 : -2 }
}
```

### Time Filters

```javascript
function check_trading_conditions() {
  const now = new Date()
  const hour_utc = now.getUTCHours()
  const day = now.getUTCDay()

  const conditions = {
    is_weekend: (day === 0 || day === 6),
    is_asia_night: (hour_utc >= 21 || hour_utc <= 5),
    is_us_open: (hour_utc >= 13 && hour_utc <= 21),
    multiplier: 1.0,
    warning: null
  }

  if (conditions.is_weekend) {
    conditions.multiplier = 0.7
    conditions.warning = "Weekend - reduced size"
  }

  if (conditions.is_asia_night && !conditions.is_weekend) {
    conditions.multiplier = 0.8
    conditions.warning = "Low liquidity hours - reduced size"
  }

  return conditions
}
```

### Protected Order

```javascript
async function place_protected_order(coin, is_buy, size, mode) {
  const SLIPPAGE = { degen: 0.8, aggressive: 0.5, balanced: 0.3, conservative: 0.2 }
  const current_price = await hyperliquid_get_price(coin)
  const slippage = SLIPPAGE[mode] / 100

  const limit_price = is_buy
    ? current_price * (1 + slippage)
    : current_price * (1 - slippage)

  return await hyperliquid_place_order({
    coin, is_buy, size,
    order_type: "limit",
    price: limit_price,
    time_in_force: "IOC"
  })
}
```

### Margin to Size

```javascript
function calculate_size(margin, leverage, price) {
  const notional = margin * leverage
  return notional / price
}
```

### Stop Management Helpers

```javascript
async function get_current_stop(coin) {
  const orders = await hyperliquid_get_open_orders({})
  const sl = orders.find(o => o.coin === coin && o.orderType === 'stop_loss')
  return sl ? sl.triggerPx : null
}

async function move_stop_loss(coin, new_price) {
  const orders = await hyperliquid_get_open_orders({})
  const sl = orders.find(o => o.coin === coin && o.orderType === 'stop_loss')

  if (sl) {
    await hyperliquid_cancel_order({ coin, oid: sl.oid })
  }

  const position = (await hyperliquid_get_positions({})).find(p => p.coin === coin)
  const is_buy = position.szi < 0  // SL for long = sell, SL for short = buy

  await hyperliquid_place_order({
    coin,
    is_buy,
    size: Math.abs(position.szi),
    order_type: "stop_loss",
    trigger_price: new_price,
    reduce_only: true
  })
}
```

### Funding Rate Edge

```javascript
async function check_funding_edge(coin, direction) {
  const funding = await hyperliquid_get_funding_rates({ coin })
  const rate = funding.fundingRate  // hourly rate

  // Extreme funding thresholds
  const EXTREME = 0.0005  // 0.05%/hr = 1.2%/day

  let edge = 'NEUTRAL', penalty = 0

  if (rate > EXTREME) {
    edge = 'SHORT_FAVORED'  // longs paying, favor shorts
    if (direction === 'LONG') penalty = -1
  } else if (rate < -EXTREME) {
    edge = 'LONG_FAVORED'  // shorts paying, favor longs
    if (direction === 'SHORT') penalty = -1
  }

  if (penalty !== 0) {
    telegram_send_message({ text: `⚠️ ${coin} funding ${(rate*100).toFixed(3)}%/hr against ${direction}, conf -1` })
  }

  return { edge, funding_rate: rate, confidence_penalty: penalty }
}
```

## Telegram Messages

Session Start: `🚀 {MODE} | ${BALANCE} → ${TARGET} (+{PCT}%) | {SCAN}`
Entry: `🟢 {DIR} {COIN} @ ${ENTRY} | {LEV}x | TP: ${TP} SL: ${SL}`
Win: `✅ {COIN} +${PNL} (+{PCT}%) | Balance: ${BALANCE}`
Loss: `❌ {COIN} -${PNL} ({PCT}%) | Balance: ${BALANCE}`
Scan: `🔍 No setup | {POS}/{MAX} | ${BALANCE} | Next: {SCAN}`
Stop Moved: `🔒 {COIN} | P&L: +{PCT}% | SL: ${NEW_SL} ({LOCKED}% locked)`
Target: `🎉 TARGET! ${START} → ${FINAL} (+{RETURN}%)`

## Hard Limits

- Daily loss limit → stop new positions
- Never exceed mode risk per trade
- Min R:R = 2:1 (TP >= 2x SL)
- Always use stop-loss
