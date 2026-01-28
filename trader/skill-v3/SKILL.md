---
name: trader-agent-v3
description: Autonomous trading agent for Hyperliquid with KV state tracking. Trigger on trading messages.
---

# Trading Agent V3

Autonomous Hyperliquid trading agent with persistent state via KV storage.

## KV Storage Keys

```
{chat_id}_session        - object: mode, starting_balance, target_balance, webhook_id, subscription_id
{chat_id}_peak_balance   - number: highest balance for drawdown calc
{chat_id}_trade_history  - array: all completed trades
{chat_id}_stats          - object: total_trades, wins, losses, total_pnl, largest_win, largest_loss, current_streak
{chat_id}_partials_{coin} - object: {p50: bool, p75: bool}
{chat_id}_daily_losses   - number: today's cumulative loss %
{chat_id}_daily_date     - string: YYYY-MM-DD for daily reset
```

## Mode Config

```javascript
const MODE_CONFIG = {
  degen: {
    leverage: [15, 25],           // 15-25x (max 25x)
    position_pct: 0.25,           // 25% of account
    risk_per_trade: 0.05,         // 5% max risk per trade (aggressive but not reckless)
    sl_pct: 10, tp_pct: 20,
    // Progressive trailing: tighten as profit grows (research-based)
    trailing: [
      { profit: 10, distance: 8 },   // +10% profit → 8% trail (breathing room)
      { profit: 15, distance: 5 },   // +15% profit → 5% trail
      { profit: 20, distance: 3 }    // +20% profit → 3% trail (lock gains)
    ],
    // Partial takes: secure profits incrementally
    partials: [
      { at_pct: 50, take: 0.30 },    // Take 30% at 50% of TP (+10%)
      { at_pct: 75, take: 0.30 }     // Take 30% at 75% of TP (+15%)
    ],                               // Remaining 40% runs with trail
    confidence_min: 5,
    scan_interval: { base: 600, volatile: 300, quiet: 1200 },
    daily_loss_limit: 15,
    max_positions: 3,
    // Risk profile: High Risk (ALL checks apply, smaller penalties)
    btc_penalty: -1,              // Small penalty if misaligned (not skip)
    funding_penalty: -1,          // Small penalty if against (not skip)
    time_size_mult: 0.85,         // 15% size reduction off-hours
    slippage: 0.8, max_spread: 0.5,
    min_liquidity_mult: 5
  },
  aggressive: {
    leverage: [5, 15],            // 5-15x (max 15x)
    position_pct: 0.15,           // 15% of account
    risk_per_trade: 0.03,         // 3% max risk per trade
    sl_pct: 6, tp_pct: 12,
    trailing: [
      { profit: 6, distance: 6 },    // +6% profit → 6% trail
      { profit: 10, distance: 4 },   // +10% profit → 4% trail
      { profit: 15, distance: 3 }    // +15% profit → 3% trail
    ],
    partials: [
      { at_pct: 50, take: 0.30 },
      { at_pct: 75, take: 0.30 }
    ],
    confidence_min: 6,
    scan_interval: { base: 1200, volatile: 600, quiet: 2400 },
    daily_loss_limit: 10,
    max_positions: 3,
    // Risk profile: Moderate-High (standard penalties)
    btc_penalty: -2,              // Moderate penalty
    funding_penalty: -1,
    time_size_mult: 0.75,         // 25% size reduction off-hours
    slippage: 0.5, max_spread: 0.3,
    min_liquidity_mult: 8
  },
  balanced: {
    leverage: [3, 5],             // 3-5x
    position_pct: 0.10,           // 10% of account
    risk_per_trade: 0.02,         // 2% max risk per trade (optimal)
    sl_pct: 4, tp_pct: 8,
    trailing: [
      { profit: 4, distance: 4 },    // +4% profit → 4% trail
      { profit: 6, distance: 3 },    // +6% profit → 3% trail
      { profit: 10, distance: 2 }    // +10% profit → 2% trail
    ],
    partials: [
      { at_pct: 50, take: 0.30 },
      { at_pct: 75, take: 0.30 }
    ],
    confidence_min: 7,
    scan_interval: { base: 7200, volatile: 3600, quiet: 14400 },
    daily_loss_limit: 8,
    max_positions: 4,
    // Risk profile: Moderate (strict checks)
    btc_check: 'REQUIRED',        // Hard skip if misaligned
    funding_penalty: -2,          // Larger penalty
    time_size_mult: 0.60,         // 40% size reduction off-hours
    slippage: 0.3, max_spread: 0.2,
    min_liquidity_mult: 10
  },
  conservative: {
    leverage: [1, 2],             // 1-2x
    position_pct: 0.06,           // 6% of account
    risk_per_trade: 0.01,         // 1% max risk per trade (safest)
    sl_pct: 2.5, tp_pct: 5,
    trailing: [
      { profit: 2.5, distance: 3 },  // +2.5% profit → 3% trail
      { profit: 4, distance: 2 },    // +4% profit → 2% trail
      { profit: 6, distance: 1.5 }   // +6% profit → 1.5% trail
    ],
    partials: [
      { at_pct: 50, take: 0.30 },
      { at_pct: 75, take: 0.30 }
    ],
    confidence_min: 8,
    scan_interval: { base: 259200, volatile: 86400, quiet: 432000 },
    daily_loss_limit: 5,
    max_positions: 6,
    max_margin: 0.60,
    // Risk profile: Capital Preservation (strictest)
    btc_check: 'REQUIRED',        // Hard skip if misaligned
    funding_check: 'REQUIRED',    // Hard skip if against
    weekend_trading: false,       // No weekend trades
    time_size_mult: 0.50,         // 50% size reduction off-hours
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

  // BTC alignment check
  const btc = await check_btc_alignment(coin, direction)
  if (config.btc_check === 'REQUIRED' && !btc.aligned) {
    skip = true  // Hard skip for balanced/conservative
  } else if (config.btc_penalty) {
    final_confidence += btc.aligned ? 0 : config.btc_penalty
  }

  // Funding rate check
  const funding = await check_funding_edge(coin, direction)
  if (config.funding_check === 'REQUIRED' && funding.confidence_penalty < 0) {
    skip = true  // Hard skip for conservative
  } else if (config.funding_penalty) {
    final_confidence += funding.confidence_penalty > 0 ? 0 : config.funding_penalty
  }

  // Time filter
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

### 1. Get Telegram Chat ID

Ask user for Telegram chat ID (from @userinfobot). Save as TELEGRAM_CHAT_ID. Skip if user says no.

### 2. Check Hyperliquid Connection

```javascript
SPLOX_SEARCH_TOOLS({ query: "hyperliquid" })
// If not connected, show connect_link and wait
// If connected, get mcp_server_id
```

### 3. Verify Account

```javascript
hyperliquid_get_balance({})
hyperliquid_get_positions({})
hyperliquid_get_open_orders({})
```

### 4. Ask Trading Mode

```
1. Conservative - 1-2x, +20%/year
2. Balanced - 3-5x, +25-50%
3. Aggressive - 5-15x, +50-100%
4. Degen - 15-25x, +100-300%
```

### 5. Load Mode File

Read `references/mode-{name}.md` and execute setup flow.

## Session Init

```javascript
async function init_session(chat_id, mode, starting_balance, target_balance) {
  await splox_kv_set({
    key: `${chat_id}_session`,
    value: JSON.stringify({
      chat_id, mode, starting_balance, target_balance,
      start_time: Date.now(), webhook_id: null, subscription_id: null
    })
  })
  await splox_kv_set({ key: `${chat_id}_peak_balance`, value: starting_balance.toString() })
  await splox_kv_set({
    key: `${chat_id}_stats`,
    value: JSON.stringify({ total_trades: 0, wins: 0, losses: 0, total_pnl: 0, largest_win: 0, largest_loss: 0, current_streak: 0 })
  })
  await splox_kv_set({ key: `${chat_id}_daily_losses`, value: "0" })
  await splox_kv_set({ key: `${chat_id}_daily_date`, value: new Date().toISOString().split('T')[0] })
  await splox_kv_set({ key: `${chat_id}_trade_history`, value: "[]" })
}
```

## Drawdown Circuit Breaker

Call on EVERY wake-up before any action.

```javascript
async function check_drawdown_circuit_breaker(chat_id, current_balance) {
  const peak_str = await splox_kv_get({ key: `${chat_id}_peak_balance` })
  let peak = parseFloat(peak_str) || current_balance

  if (current_balance > peak) {
    peak = current_balance
    await splox_kv_set({ key: `${chat_id}_peak_balance`, value: peak.toString() })
  }

  const drawdown_pct = ((peak - current_balance) / peak) * 100
  let action = 'continue', size_mult = 1.0

  if (drawdown_pct >= 20) {
    action = 'halt'
    telegram_send_message({ text: `🛑 HALTED: -${drawdown_pct.toFixed(1)}% drawdown. Closing all.` })
    await cleanup_all_positions()
  } else if (drawdown_pct >= 15) {
    action = 'pause'
    size_mult = 0.5
    telegram_send_message({ text: `🔶 Drawdown -${drawdown_pct.toFixed(1)}%, pause 2h, size -50%` })
  } else if (drawdown_pct >= 10) {
    action = 'reduce'
    size_mult = 0.7
    telegram_send_message({ text: `⚠️ Drawdown -${drawdown_pct.toFixed(1)}%, size -30%` })
  }

  return { drawdown_pct, peak, action, size_multiplier: size_mult, halt: action === 'halt' }
}
```

## Daily Loss Tracking

```javascript
async function check_daily_loss_limit(chat_id, mode) {
  const today = new Date().toISOString().split('T')[0]
  const stored_date = await splox_kv_get({ key: `${chat_id}_daily_date` })

  if (stored_date !== today) {
    await splox_kv_set({ key: `${chat_id}_daily_date`, value: today })
    await splox_kv_set({ key: `${chat_id}_daily_losses`, value: "0" })
    return { exceeded: false, daily_loss: 0 }
  }

  const daily_loss = parseFloat(await splox_kv_get({ key: `${chat_id}_daily_losses` })) || 0
  const limit = MODE_CONFIG[mode].daily_loss_limit

  if (daily_loss >= limit) {
    telegram_send_message({ text: `🛑 Daily limit: -${daily_loss.toFixed(1)}% (max -${limit}%)` })
    return { exceeded: true, daily_loss }
  }
  return { exceeded: false, daily_loss }
}

async function record_daily_loss(chat_id, loss_pct) {
  await splox_kv_increment({ key: `${chat_id}_daily_losses`, increment: loss_pct })
}
```

## Consecutive Losses

```javascript
async function check_consecutive_losses(chat_id) {
  const stats = JSON.parse(await splox_kv_get({ key: `${chat_id}_stats` }) || '{}')
  const streak = stats.current_streak || 0

  if (streak <= -5) {
    telegram_send_message({ text: `🛑 ${Math.abs(streak)} losses, stopping 24h` })
    return { cooldown: true, streak, stop_24h: true, size_multiplier: 0.5 }
  }
  if (streak <= -3) {
    telegram_send_message({ text: `⏸️ ${Math.abs(streak)} losses, cooldown 4-6h, size -50%` })
    return { cooldown: true, streak, stop_24h: false, size_multiplier: 0.5 }
  }
  return { cooldown: false, streak, size_multiplier: 1.0 }
}
```

## Dynamic Stop Management

```javascript
async function manage_dynamic_stop(position, mode) {
  const pnl_pct = position.unrealizedPnl / position.marginUsed * 100
  const entry = position.entryPx
  const dir = position.szi > 0 ? 1 : -1
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
    const new_stop_price = entry * (1 + (new_stop_pct / 100) * dir)
    const current_stop = await get_current_stop(position.coin)
    const is_better = dir > 0 ? new_stop_price > current_stop : new_stop_price < current_stop

    if (is_better) {
      await move_stop_loss(position.coin, new_stop_price)
      telegram_send_message({ text: `🔒 ${position.coin} +${pnl_pct.toFixed(1)}% → SL at ${new_stop_pct > 0 ? '+' : ''}${new_stop_pct.toFixed(1)}%` })
    }
  }
}
```

## Partial Takes

```javascript
async function manage_partial_takes(chat_id, position, tp_pct) {
  const pnl_pct = position.unrealizedPnl / position.marginUsed * 100
  const progress = pnl_pct / tp_pct

  const key = `${chat_id}_partials_${position.coin}`
  const partials = JSON.parse(await splox_kv_get({ key }) || '{"p50":false,"p75":false}')

  if (progress >= 0.5 && !partials.p50) {
    await hyperliquid_market_close({ coin: position.coin, size: Math.abs(position.szi) * 0.3 })
    partials.p50 = true
    await splox_kv_set({ key, value: JSON.stringify(partials) })
    telegram_send_message({ text: `💰 ${position.coin} Partial #1: 30% at +${pnl_pct.toFixed(1)}%` })
  }

  if (progress >= 0.75 && !partials.p75) {
    await hyperliquid_market_close({ coin: position.coin, size: Math.abs(position.szi) * 0.3 })
    partials.p75 = true
    await splox_kv_set({ key, value: JSON.stringify(partials) })
    telegram_send_message({ text: `💰 ${position.coin} Partial #2: 30% at +${pnl_pct.toFixed(1)}%` })
  }
}

async function clear_partials(chat_id, coin) {
  await splox_kv_delete({ key: `${chat_id}_partials_${coin}` })
}
```

## Record Trade

```javascript
async function record_trade(chat_id, trade) {
  const stats = JSON.parse(await splox_kv_get({ key: `${chat_id}_stats` }) || '{}')

  stats.total_trades = (stats.total_trades || 0) + 1
  stats.total_pnl = (stats.total_pnl || 0) + trade.pnl_pct

  if (trade.pnl_pct > 0) {
    stats.wins = (stats.wins || 0) + 1
    stats.largest_win = Math.max(stats.largest_win || 0, trade.pnl_pct)
    stats.current_streak = stats.current_streak > 0 ? stats.current_streak + 1 : 1
  } else {
    stats.losses = (stats.losses || 0) + 1
    stats.largest_loss = Math.min(stats.largest_loss || 0, trade.pnl_pct)
    stats.current_streak = stats.current_streak < 0 ? stats.current_streak - 1 : -1
    await record_daily_loss(chat_id, Math.abs(trade.pnl_pct))
  }

  await splox_kv_set({ key: `${chat_id}_stats`, value: JSON.stringify(stats) })

  await splox_kv_append_array({
    key: `${chat_id}_trade_history`,
    value: JSON.stringify({
      coin: trade.coin, direction: trade.direction,
      entry: trade.entry_price, exit: trade.exit_price,
      pnl_pct: trade.pnl_pct, pnl_usd: trade.pnl_usd,
      exit_reason: trade.exit_reason, timestamp: Date.now()
    })
  })

  if (stats.total_trades % 5 === 0) {
    const wr = (stats.wins / stats.total_trades * 100).toFixed(0)
    telegram_send_message({ text: `📊 ${stats.total_trades} trades | ${wr}% WR | ${stats.total_pnl > 0 ? '+' : ''}${stats.total_pnl.toFixed(1)}% total` })
  }
}
```

## Bracket Order (Atomic Entry + TP + SL)

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

## Fallback: Separate Orders

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

## Liquidity Check

```javascript
async function check_liquidity(coin, size_usd, mode) {
  const book = await hyperliquid_get_orderbook({ coin })
  const spread = ((book.asks[0][0] - book.bids[0][0]) / book.bids[0][0]) * 100
  const max_spread = { degen: 0.5, aggressive: 0.3, balanced: 0.2, conservative: 0.15 }[mode]

  if (spread > max_spread) {
    telegram_send_message({ text: `⚠️ ${coin} spread ${spread.toFixed(2)}% > ${max_spread}%, skip` })
    return { ok: false }
  }

  const depth = Math.min(
    book.bids.slice(0,5).reduce((s,[p,sz]) => s + p*sz, 0),
    book.asks.slice(0,5).reduce((s,[p,sz]) => s + p*sz, 0)
  )

  if (depth < size_usd * 10) {
    telegram_send_message({ text: `⚠️ ${coin} low depth $${depth.toFixed(0)}, skip` })
    return { ok: false }
  }
  return { ok: true, spread, depth }
}
```

## BTC Alignment

```javascript
async function check_btc_alignment(coin, direction) {
  if (coin === 'BTC') return { aligned: true, confidence_penalty: 0 }

  const candles = await hyperliquid_get_candles({ coin: 'BTC', interval: '4h', limit: 2 })
  const change = (candles[1].close - candles[0].close) / candles[0].close * 100

  let trend = 'NEUTRAL'
  if (change > 1) trend = 'UP'
  if (change < -1) trend = 'DOWN'

  const aligned = (direction === 'LONG' && trend !== 'DOWN') || (direction === 'SHORT' && trend !== 'UP')

  if (!aligned) {
    telegram_send_message({ text: `⚠️ BTC ${trend}, ${direction} ${coin} misaligned, conf -2` })
  }
  return { aligned, btc_trend: trend, confidence_penalty: aligned ? 0 : -2 }
}
```

## Time Filter

```javascript
function check_trading_conditions() {
  const now = new Date()
  const hour = now.getUTCHours()
  const day = now.getUTCDay()

  let mult = 1.0, warning = null
  const is_weekend = day === 0 || day === 6
  const is_night = hour >= 21 || hour <= 5

  if (is_weekend) { mult = 0.7; warning = "Weekend, size x0.7" }
  else if (is_night) { mult = 0.8; warning = "Night, size x0.8" }

  return { is_weekend, is_night, multiplier: mult, warning }
}
```

## Margin to Size

```javascript
function calculate_size(margin, leverage, price) {
  const notional = margin * leverage
  return notional / price
}
```

## Stop Management Helpers

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
  const is_buy = position.szi < 0

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

## Funding Rate Edge

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

  if (penalty !== 0) {
    telegram_send_message({ text: `⚠️ ${coin} funding ${(rate*100).toFixed(3)}%/hr against ${direction}, conf -1` })
  }

  return { edge, funding_rate: rate, confidence_penalty: penalty }
}
```

## Adaptive Scan Interval

```javascript
async function get_scan_interval(mode) {
  const config = MODE_CONFIG[mode]
  const intervals = config.scan_interval

  // Check market volatility (BTC 1h change)
  const btc = await hyperliquid_get_candles({ coin: 'BTC', interval: '1h', limit: 2 })
  const volatility = Math.abs((btc[1].close - btc[0].close) / btc[0].close * 100)

  let interval, reason
  if (volatility > 3) {
    interval = intervals.volatile
    reason = 'high_vol'
  } else if (volatility < 0.5) {
    interval = intervals.quiet
    reason = 'low_vol'
  } else {
    interval = intervals.base
    reason = 'normal'
  }

  return { interval, reason, volatility }
}

function format_interval(seconds) {
  if (seconds >= 86400) return `${(seconds / 86400).toFixed(0)}d`
  if (seconds >= 3600) return `${(seconds / 3600).toFixed(0)}h`
  if (seconds >= 60) return `${(seconds / 60).toFixed(0)}m`
  return `${seconds}s`
}
```

## Progressive Trailing Stop Management

```javascript
async function check_trailing_stop(position, mode) {
  const config = MODE_CONFIG[mode]
  const trailing_levels = config.trailing  // Array of {profit, distance}
  const pnl_pct = position.unrealizedPnl / position.marginUsed * 100

  // Find the appropriate trailing distance based on profit level
  // Higher profit = tighter trail (progressive tightening strategy)
  let active_trail = null
  for (const level of trailing_levels) {
    if (pnl_pct >= level.profit) {
      active_trail = level
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

## Progress Calculation

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

## Parameter Selection (Confidence-Based)

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
    entry: "🎰 ${dir} ${coin} @ $${price} | ${lev}x | TP $${tp} SL $${sl}",
    win: "💰 ${coin} PRINTED +$${pnl} (+${pct}%) | $${bal} | ${progress}% to target",
    loss: "💀 ${coin} REKT -$${pnl} (${pct}%) | $${bal} | ${progress}% to target",
    scan: "👀 Scanning | ${pos}/${max} pos | $${bal} | Next: ${next_scan}",
    trail: "🔥 ${coin} MOONING +${pnl}% | Trailing at +${locked}%",
    target: "🎉🎰 TARGET HIT! $${start} → $${final} (+${ret}%) | WAGMI"
  },
  aggressive: {
    start: "🚀 Aggressive | $${bal} → $${target} (+${pct}%)",
    entry: "🟢 ${dir} ${coin} @ $${price} | ${lev}x | TP $${tp} SL $${sl}",
    win: "✅ ${coin} +$${pnl} (+${pct}%) | $${bal} | ${progress}% to target",
    loss: "❌ ${coin} -$${pnl} (${pct}%) | $${bal} | ${progress}% to target",
    scan: "🔍 Scanning | ${pos}/${max} | $${bal} | Next: ${next_scan}",
    trail: "🔒 ${coin} +${pnl}% | Trailing at +${locked}%",
    target: "🎉 TARGET! $${start} → $${final} (+${ret}%)"
  },
  balanced: {
    start: "⚖️ Balanced | $${bal} → $${target} (+${pct}%)",
    entry: "📊 ${dir} ${coin} @ $${price} | ${lev}x | R:R ${rr}",
    win: "✅ ${coin} +$${pnl} (+${pct}%) | $${bal} | ${progress}%",
    loss: "❌ ${coin} -$${pnl} (${pct}%) | $${bal}",
    scan: "📊 Portfolio: ${pos}/${max} | $${bal} | ${progress}% | Next: ${next_scan}",
    trail: "🔒 ${coin} secured at +${locked}%",
    target: "🎯 Target reached: $${start} → $${final} (+${ret}%)"
  },
  conservative: {
    start: "🛡️ Conservative | $${bal} → $${target} (+${pct}%/year)",
    entry: "🛡️ ${dir} ${coin} @ $${price} | ${lev}x | Safe entry",
    win: "✓ ${coin} +$${pnl} (+${pct}%)",
    loss: "✗ ${coin} -$${pnl} (${pct}%)",
    scan: "📅 Check | ${pos}/${max} | Cash: ${cash}% | Next: ${next_scan}",
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

## Cleanup

```javascript
async function cleanup_session(chat_id) {
  const stats = JSON.parse(await splox_kv_get({ key: `${chat_id}_stats` }) || '{}')
  const session = JSON.parse(await splox_kv_get({ key: `${chat_id}_session` }) || '{}')

  const positions = await hyperliquid_get_positions({})
  for (const p of positions) await hyperliquid_market_close({ coin: p.coin })

  if (session.subscription_id) await event_unsubscribe({ subscription_id: session.subscription_id })
  await hyperliquid_unsubscribe_webhook({})

  const balance = await hyperliquid_get_balance({})
  const ret = ((balance.accountValue - session.starting_balance) / session.starting_balance * 100)

  telegram_send_message({
    text: `🏁 Session ended
$${session.starting_balance} → $${balance.accountValue.toFixed(2)} (${ret > 0 ? '+' : ''}${ret.toFixed(1)}%)
${stats.total_trades} trades | ${stats.wins}W/${stats.losses}L`
  })
}
```

## Telegram Templates

Use `notify(mode, event, data)` for mode-specific messages. See NOTIFICATIONS object above.

Partial (shared): `💰 {COIN} Partial #{N}: 30% at +{PNL}%`

Stats (every 5 trades): `📊 {total_trades} trades | {wr}% WR | {total_pnl}% total`

## Hard Limits

- Daily loss limit → stop new positions
- 3 consecutive losses → cooldown 4-6h, size -50%
- 5 losses in day → stop 24h
- -20% drawdown → HALT ALL
- Min R:R = 2:1 always
