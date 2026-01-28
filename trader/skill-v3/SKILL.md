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
    leverage_min: 15, leverage_max: 25,
    position_pct: 0.25,
    sl_pct: 10, tp_pct: 20,
    confidence_min: 5,
    scan_interval: 600,
    slippage: 0.8, max_spread: 0.5,
    daily_loss_limit: 15
  },
  aggressive: {
    leverage_min: 5, leverage_max: 15,
    position_pct: 0.15,
    sl_pct: 6, tp_pct: 12,
    confidence_min: 6,
    scan_interval: 1200,
    slippage: 0.5, max_spread: 0.3,
    daily_loss_limit: 10
  },
  balanced: {
    leverage_min: 3, leverage_max: 5,
    position_pct: 0.10,
    sl_pct: 4, tp_pct: 8,
    confidence_min: 7,
    scan_interval: 7200,
    slippage: 0.3, max_spread: 0.2,
    daily_loss_limit: 8
  },
  conservative: {
    leverage_min: 1, leverage_max: 2,
    position_pct: 0.06,
    sl_pct: 2.5, tp_pct: 5,
    confidence_min: 8,
    scan_interval: 259200,
    slippage: 0.2, max_spread: 0.15,
    daily_loss_limit: 5
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

## Slippage Protection

```javascript
async function place_protected_order(coin, is_buy, size, mode) {
  const slippage = { degen: 0.008, aggressive: 0.005, balanced: 0.003, conservative: 0.002 }[mode]
  const price = await hyperliquid_get_price(coin)
  const limit = is_buy ? price * (1 + slippage) : price * (1 - slippage)

  return await hyperliquid_place_order({
    coin, is_buy, size,
    order_type: "limit",
    price: limit,
    time_in_force: "IOC"
  })
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

Session start: `🚀 {MODE} | ${BAL} → ${TARGET} (+{PCT}%) | {INTERVAL}`

Entry: `🟢 {DIR} {COIN} @ ${PRICE} | {LEV}x | TP ${TP} SL ${SL}`

Win: `✅ {COIN} +${PNL} (+{PCT}%) | ${BAL} | {W}W/{L}L`

Loss: `❌ {COIN} -${PNL} ({PCT}%) | ${BAL} | {W}W/{L}L`

Stop moved: `🔒 {COIN} +{PNL}% → SL at {LOCKED}%`

Partial: `💰 {COIN} Partial #{N}: 30% at +{PNL}%`

Target: `🎉 TARGET! ${START} → ${FINAL} (+{RET}%) | {W}W/{L}L`

## Hard Limits

- Daily loss limit → stop new positions
- 3 consecutive losses → cooldown 4-6h, size -50%
- 5 losses in day → stop 24h
- -20% drawdown → HALT ALL
- Min R:R = 2:1 always
