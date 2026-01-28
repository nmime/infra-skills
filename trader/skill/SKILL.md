---
name: trader-agent-v2
description: Autonomous AI trading agent for Hyperliquid perpetual futures exchange. Use when user wants to trade crypto, set up trading automation, monitor positions, analyze markets, or manage a Hyperliquid account. Trigger on any trading-related message like "I want to trade", "What's my balance?", "Start trading", "Close positions", etc.
---

# Trading Agent (v2 - Stateless)

Autonomous trading agent for Hyperliquid. No persistent state storage.

## Mode Config

```javascript
const MODE_CONFIG = {
  degen: {
    leverage: [15, 25],           // 15-25x (max 25x)
    position_pct: 0.25,           // 25% of account
    risk_per_trade: 0.05,         // 5% max risk per trade
    sl_pct: 10, tp_pct: 20,
    // Progressive trailing (research-based: tighten as profit grows)
    trailing: [
      { profit: 10, distance: 8 },   // +10% → 8% trail
      { profit: 15, distance: 5 },   // +15% → 5% trail
      { profit: 20, distance: 3 }    // +20% → 3% trail
    ],
    partials: [
      { at_pct: 50, take: 0.30 },    // 30% at +10%
      { at_pct: 75, take: 0.30 }     // 30% at +15%
    ],
    confidence_min: 5,
    scan_interval: { base: 600, volatile: 300, quiet: 1200 },
    daily_limit: -15,
    max_positions: 3,
    // Risk profile: High Risk (ALL checks apply, smaller penalties)
    btc_penalty: -1,
    funding_penalty: -1,
    time_size_mult: 0.85,
    slippage: 0.8, max_spread: 0.5,
    min_liquidity_mult: 5
  },
  aggressive: {
    leverage: [5, 15],            // 5-15x (max 15x)
    position_pct: 0.15,           // 15% of account
    risk_per_trade: 0.03,         // 3% max risk per trade
    sl_pct: 6, tp_pct: 12,
    trailing: [
      { profit: 6, distance: 6 },
      { profit: 10, distance: 4 },
      { profit: 15, distance: 3 }
    ],
    partials: [
      { at_pct: 50, take: 0.30 },
      { at_pct: 75, take: 0.30 }
    ],
    confidence_min: 6,
    scan_interval: { base: 1200, volatile: 600, quiet: 2400 },
    daily_limit: -10,
    max_positions: 3,
    // Risk profile: Moderate-High
    btc_penalty: -2,
    funding_penalty: -1,
    time_size_mult: 0.75,
    slippage: 0.5, max_spread: 0.3,
    min_liquidity_mult: 8
  },
  balanced: {
    leverage: [3, 5],             // 3-5x
    position_pct: 0.10,           // 10% of account
    risk_per_trade: 0.02,         // 2% max risk per trade (optimal)
    sl_pct: 4, tp_pct: 8,
    trailing: [
      { profit: 4, distance: 4 },
      { profit: 6, distance: 3 },
      { profit: 10, distance: 2 }
    ],
    partials: [
      { at_pct: 50, take: 0.30 },
      { at_pct: 75, take: 0.30 }
    ],
    confidence_min: 7,
    scan_interval: { base: 7200, volatile: 3600, quiet: 14400 },
    daily_limit: -8,
    max_positions: 4,
    // Risk profile: Moderate (strict checks)
    btc_check: 'REQUIRED',
    funding_penalty: -2,
    time_size_mult: 0.60,
    slippage: 0.3, max_spread: 0.2,
    min_liquidity_mult: 10
  },
  conservative: {
    leverage: [1, 2],             // 1-2x
    position_pct: 0.06,           // 6% of account
    risk_per_trade: 0.01,         // 1% max risk per trade
    sl_pct: 2.5, tp_pct: 5,
    trailing: [
      { profit: 2.5, distance: 3 },
      { profit: 4, distance: 2 },
      { profit: 6, distance: 1.5 }
    ],
    partials: [
      { at_pct: 50, take: 0.30 },
      { at_pct: 75, take: 0.30 }
    ],
    confidence_min: 8,
    scan_interval: { base: 259200, volatile: 86400, quiet: 432000 },
    daily_limit: -5,
    max_positions: 6,
    max_margin: 0.60,
    // Risk profile: Capital Preservation (strictest)
    btc_check: 'REQUIRED',
    funding_check: 'REQUIRED',
    weekend_trading: false,
    time_size_mult: 0.50,
    slippage: 0.2, max_spread: 0.15,
    min_liquidity_mult: 15
  }
}
```

## Risk Check Functions

```javascript
function apply_risk_checks(mode, coin, direction, confidence) {
  const config = MODE_CONFIG[mode]
  let final_confidence = confidence
  let size_mult = 1.0
  let skip = false

  // BTC alignment check (ALL modes check, different consequences)
  const btc = await check_btc_alignment(coin, direction)
  if (config.btc_check === 'REQUIRED' && !btc.aligned) {
    skip = true  // Hard skip for balanced/conservative
  } else if (config.btc_penalty) {
    final_confidence += btc.aligned ? 0 : config.btc_penalty
  }

  // Funding rate check (ALL modes check)
  const funding = await check_funding_edge(coin, direction)
  if (config.funding_check === 'REQUIRED' && funding.confidence_penalty < 0) {
    skip = true  // Hard skip for conservative
  } else if (config.funding_penalty) {
    final_confidence += funding.confidence_penalty > 0 ? 0 : config.funding_penalty
  }

  // Time filter (ALL modes apply size reduction)
  const time = check_trading_conditions()
  if (config.weekend_trading === false && time.is_weekend) {
    skip = true
  }
  if (time.is_weekend || time.is_asia_night) {
    size_mult *= config.time_size_mult
  }

  return { skip, confidence: final_confidence, size_mult }
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

### Bracket Order (Atomic Entry + TP + SL)

```javascript
async function place_bracket_order(coin, is_buy, size, entry_price, tp_price, sl_price, mode) {
  const slippage = MODE_CONFIG[mode].slippage / 100
  const limit_price = is_buy
    ? entry_price * (1 + slippage)
    : entry_price * (1 - slippage)

  // Single atomic order with TP and SL attached
  return await hyperliquid_place_order({
    coin, is_buy, size,
    order_type: "limit",
    price: limit_price,
    time_in_force: "IOC",
    take_profit: { trigger_price: tp_price },
    stop_loss: { trigger_price: sl_price }
  })
}
```

### Fallback: Separate Orders

```javascript
async function place_orders_separate(coin, is_buy, size, entry_price, tp_price, sl_price, mode) {
  const slippage = MODE_CONFIG[mode].slippage / 100
  const limit_price = is_buy
    ? entry_price * (1 + slippage)
    : entry_price * (1 - slippage)

  const entry = await hyperliquid_place_order({
    coin, is_buy, size,
    order_type: "limit",
    price: limit_price,
    time_in_force: "IOC"
  })

  if (entry.filled) {
    await hyperliquid_place_order({ coin, is_buy: !is_buy, size, order_type: "take_profit", trigger_price: tp_price, reduce_only: true })
    await hyperliquid_place_order({ coin, is_buy: !is_buy, size, order_type: "stop_loss", trigger_price: sl_price, reduce_only: true })
  }

  return entry
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
  const rate = funding.fundingRate

  const EXTREME = 0.0005  // 0.05%/hr

  let edge = 'NEUTRAL', penalty = 0

  if (rate > EXTREME) {
    edge = 'SHORT_FAVORED'
    if (direction === 'LONG') penalty = -1
  } else if (rate < -EXTREME) {
    edge = 'LONG_FAVORED'
    if (direction === 'SHORT') penalty = -1
  }

  return { edge, funding_rate: rate, confidence_penalty: penalty }
}
```

### Adaptive Scan Interval

```javascript
async function get_scan_interval(mode) {
  const config = MODE_CONFIG[mode]
  const intervals = config.scan_interval

  // Check market volatility (BTC 1h change)
  const btc = await hyperliquid_get_candles({ coin: 'BTC', interval: '1h', limit: 2 })
  const volatility = Math.abs((btc[1].close - btc[0].close) / btc[0].close * 100)

  if (volatility > 3) return intervals.volatile    // High vol: scan faster
  if (volatility < 0.5) return intervals.quiet     // Low vol: scan slower
  return intervals.base                             // Normal: base interval
}
```

### Trailing Stop Management

```javascript
async function check_trailing_stop(position, mode) {
  const config = MODE_CONFIG[mode]
  const trailing_levels = config.trailing  // Array of {profit, distance}
  const pnl_pct = position.unrealizedPnl / position.marginUsed * 100

  // Progressive trailing: find appropriate level based on profit
  let active_trail = null
  for (const level of trailing_levels) {
    if (pnl_pct >= level.profit) {
      active_trail = level  // Higher profit = tighter trail
    }
  }

  if (active_trail) {
    const trail_price = position.markPx * (position.szi > 0
      ? (1 - active_trail.distance / 100)
      : (1 + active_trail.distance / 100))

    const current_stop = await get_current_stop(position.coin)
    const is_better = position.szi > 0
      ? trail_price > current_stop
      : trail_price < current_stop

    if (is_better) {
      await move_stop_loss(position.coin, trail_price)
      const locked = pnl_pct - active_trail.distance
      telegram_send_message({
        text: `🔒 ${position.coin} +${pnl_pct.toFixed(1)}% | Trail ${active_trail.distance}% | Locked +${locked.toFixed(1)}%`
      })
      return { moved: true, new_stop: trail_price, locked, trail_distance: active_trail.distance }
    }
  }
  return { moved: false }
}
```

### Progress Calculation

```javascript
function calculate_progress(current, starting, target) {
  const gained = current - starting
  const needed = target - starting
  const progress_pct = (gained / needed * 100).toFixed(0)
  return {
    progress_pct,
    gained,
    remaining: target - current,
    on_track: current >= starting
  }
}
```

### Parameter Selection (Confidence-Based)

```javascript
function select_params(mode, confidence) {
  const config = MODE_CONFIG[mode]
  const conf_factor = (confidence - config.confidence_min) / (10 - config.confidence_min)

  // Higher confidence = higher end of range (for leverage only)
  const pick = (range) => Array.isArray(range)
    ? range[0] + (range[1] - range[0]) * conf_factor
    : range

  return {
    leverage: pick(config.leverage),
    position_pct: config.position_pct,
    sl_pct: config.sl_pct,
    tp_pct: config.tp_pct
  }
}
```

## Telegram Messages (Mode-Specific)

```javascript
const NOTIFICATIONS = {
  degen: {
    start: "🎰 DEGEN MODE | $${bal} → $${target} (+${pct}%) | LFG",
    entry: "🎰 YOLO ${dir} ${coin} @ $${price} | ${lev}x | TP $${tp} SL $${sl}",
    win: "💰 ${coin} PRINTED +$${pnl} (+${pct}%) | $${bal} | ${progress}% to target",
    loss: "💀 ${coin} REKT -$${pnl} (${pct}%) | $${bal} | ${progress}% to target",
    scan: "👀 Scanning for plays... | ${pos}/${max} positions | $${bal}",
    trail: "🔥 ${coin} MOONING +${pnl}% | Trailing at +${locked}%",
    target: "🎉🎰 TARGET HIT! $${start} → $${final} (+${ret}%) | WAGMI"
  },
  aggressive: {
    start: "🚀 Aggressive | $${bal} → $${target} (+${pct}%)",
    entry: "🟢 ${dir} ${coin} @ $${price} | ${lev}x | TP $${tp} SL $${sl}",
    win: "✅ ${coin} +$${pnl} (+${pct}%) | $${bal} | ${progress}% to target",
    loss: "❌ ${coin} -$${pnl} (${pct}%) | $${bal} | ${progress}% to target",
    scan: "🔍 Scanning | ${pos}/${max} | $${bal}",
    trail: "🔒 ${coin} +${pnl}% | Trailing at +${locked}%",
    target: "🎉 TARGET! $${start} → $${final} (+${ret}%)"
  },
  balanced: {
    start: "⚖️ Balanced | $${bal} → $${target} (+${pct}%)",
    entry: "📊 ${dir} ${coin} @ $${price} | ${lev}x | R:R ${rr}",
    win: "✅ ${coin} +$${pnl} (+${pct}%) | $${bal} | ${progress}%",
    loss: "❌ ${coin} -$${pnl} (${pct}%) | $${bal}",
    scan: "📊 Portfolio: ${pos}/${max} | $${bal} | ${progress}%",
    trail: "🔒 ${coin} secured at +${locked}%",
    target: "🎯 Target reached: $${start} → $${final} (+${ret}%)"
  },
  conservative: {
    start: "🛡️ Conservative | $${bal} → $${target} (+${pct}%/year)",
    entry: "🛡️ ${dir} ${coin} @ $${price} | ${lev}x | Safe entry",
    win: "✓ ${coin} +$${pnl} (+${pct}%)",
    loss: "✗ ${coin} -$${pnl} (${pct}%)",
    scan: "📅 3-Day Check | ${pos}/${max} | Cash: ${cash}%",
    trail: "🔒 ${coin} protected at +${locked}%",
    target: "🏆 Annual target: $${start} → $${final} (+${ret}%)"
  }
}

function notify(mode, event, data) {
  const template = NOTIFICATIONS[mode][event]
  const msg = template.replace(/\$\{(\w+)\}/g, (_, k) => data[k] ?? '')
  telegram_send_message({ text: msg })
}
```

## Hard Limits

- Daily loss limit → stop new positions
- Never exceed mode risk per trade
- Min R:R = 2:1 (TP >= 2x SL)
- Always use stop-loss
