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
    leverage: 'MAX',              // Use coin's max leverage
    position_pct: [0.30, 0.50],   // 30-50% based on confidence
    sl_pct: [10, 15],             // 10-15%
    tp_pct: [15, 25],             // 15-25%
    trailing: { trigger: 20, distance: 15 },  // Trail 15% after +20%
    confidence_min: 5,
    scan_interval: { base: 7200, volatile: 1800, quiet: 14400 },  // 2hr/30min/4hr
    daily_loss_limit: 15,
    max_positions: 2,
    // Risk profile: YOLO
    skip_btc_check: true,
    skip_time_filter: true,
    skip_funding_check: true,
    slippage: 1.0, max_spread: 1.0,
    min_liquidity_mult: 5
  },
  aggressive: {
    leverage: [10, 20],           // 10-20x
    position_pct: [0.15, 0.25],   // 15-25%
    sl_pct: [5, 8],               // 5-8%
    tp_pct: [12, 20],             // 12-20%
    trailing: { trigger: 15, distance: 4 },
    confidence_min: 6,
    scan_interval: { base: 3600, volatile: 1200, quiet: 7200 },  // 1hr/20min/2hr
    daily_loss_limit: 10,
    max_positions: 3,
    // Risk profile: High
    skip_btc_check: false,        // Check but don't hard skip
    skip_time_filter: false,
    skip_funding_check: false,
    slippage: 0.5, max_spread: 0.5,
    min_liquidity_mult: 8
  },
  balanced: {
    leverage: [3, 7],             // 3-7x
    position_pct: [0.08, 0.12],   // 8-12%
    sl_pct: [3, 5],               // 3-5%
    tp_pct: [8, 12],              // 8-12%
    trailing: { trigger: 10, distance: 3 },
    confidence_min: 7,
    scan_interval: { base: 7200, volatile: 3600, quiet: 14400 },  // 2hr/1hr/4hr
    daily_loss_limit: 8,
    max_positions: 4,
    // Risk profile: Moderate
    btc_alignment: 'REQUIRED',
    skip_time_filter: false,
    skip_funding_check: false,
    slippage: 0.3, max_spread: 0.2,
    min_liquidity_mult: 10
  },
  conservative: {
    leverage: [1, 2],             // 1-2x
    position_pct: [0.05, 0.08],   // 5-8%
    sl_pct: [2, 3],               // 2-3%
    tp_pct: [5, 8],               // 5-8%
    trailing: { trigger: 6, distance: 2 },
    confidence_min: 8,
    scan_interval: { base: 259200, volatile: 86400, quiet: 432000 },  // 3d/1d/5d
    daily_loss_limit: 5,
    max_positions: 6,
    max_margin: 0.60,
    // Risk profile: Capital preservation
    btc_alignment: 'REQUIRED',
    funding_check: 'STRICT',      // Hard skip if against
    weekend_trading: false,
    slippage: 0.2, max_spread: 0.15,
    min_liquidity_mult: 15
  }
}
```

## Risk Profiles

```javascript
function get_risk_checks(mode) {
  const config = MODE_CONFIG[mode]
  return {
    check_btc: !config.skip_btc_check,
    check_time: !config.skip_time_filter,
    check_funding: !config.skip_funding_check,
    btc_hard_skip: config.btc_alignment === 'REQUIRED',
    funding_hard_skip: config.funding_check === 'STRICT',
    min_liquidity: config.min_liquidity_mult
  }
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

  if (volatility > 3) return intervals.volatile    // High vol: scan faster
  if (volatility < 0.5) return intervals.quiet     // Low vol: scan slower
  return intervals.base                             // Normal: base interval
}
```

## Trailing Stop Management

```javascript
async function check_trailing_stop(position, mode) {
  const config = MODE_CONFIG[mode]
  const trailing = config.trailing
  const pnl_pct = position.unrealizedPnl / position.marginUsed * 100

  if (pnl_pct >= trailing.trigger) {
    const trail_price = position.markPx * (position.szi > 0
      ? (1 - trailing.distance / 100)
      : (1 + trailing.distance / 100))

    const current_stop = await get_current_stop(position.coin)
    const is_better = position.szi > 0
      ? trail_price > current_stop
      : trail_price < current_stop

    if (is_better) {
      await move_stop_loss(position.coin, trail_price)
      return { moved: true, new_stop: trail_price, locked: pnl_pct - trailing.distance }
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

  // Higher confidence = higher end of range
  const pick = (range) => Array.isArray(range)
    ? range[0] + (range[1] - range[0]) * conf_factor
    : range

  return {
    leverage: config.leverage === 'MAX' ? 'MAX' : pick(config.leverage),
    position_pct: pick(config.position_pct),
    sl_pct: pick(config.sl_pct),
    tp_pct: pick(config.tp_pct)
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
