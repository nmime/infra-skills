# Agent Checklist — Quick Reference

Чёткий пошаговый чеклист для агента. Используй при каждом wake-up.

---

## Session Variables (сохранять между wake-up)

```javascript
// Инициализировать при старте сессии
SESSION = {
  // Основные
  MODE: 'degen',                    // Режим торговли
  STARTING_BALANCE: 0,              // Начальный баланс
  TARGET_BALANCE: 0,                // Целевой баланс
  PEAK_BALANCE: 0,                  // Максимальный баланс (для drawdown)

  // Webhook/Schedule IDs
  WEBHOOK_ID: null,
  WEBHOOK_URL: null,
  SUBSCRIPTION_ID: null,
  SCHEDULE_ID: null,

  // Trade tracking
  trade_history: [],                // История сделок
  consecutive_losses: 0,            // Убытки подряд
  daily_losses: 0,                  // Убытки за день
  last_trade_date: null,            // Дата последней сделки

  // Position tracking
  partials_taken: {},               // { 'BTC': { p50: true, p75: false }, ... }

  // Multipliers
  POSITION_SIZE_MULTIPLIER: 1.0,    // Множитель размера (drawdown/time)
  PAUSE_NEW_ENTRIES_UNTIL: null,    // Timestamp паузы

  // Stats
  session_stats: {
    total_trades: 0,
    wins: 0,
    losses: 0,
    total_pnl: 0,
    largest_win: 0,
    largest_loss: 0,
    current_streak: 0
  }
}
```

---

## TP/SL Values by Mode

```javascript
const MODE_CONFIG = {
  degen: {
    leverage_min: 15,
    leverage_max: 25,
    position_pct: 0.25,      // 25% of account
    sl_pct: 10,              // -10%
    tp_pct: 20,              // +20% (min 2× SL)
    confidence_min: 5,
    scan_interval: 600,      // 10 min
    slippage: 0.8,
    max_spread: 0.5
  },
  aggressive: {
    leverage_min: 5,
    leverage_max: 15,
    position_pct: 0.15,      // 15%
    sl_pct: 6,               // -6%
    tp_pct: 12,              // +12%
    confidence_min: 6,
    scan_interval: 1200,     // 20 min
    slippage: 0.5,
    max_spread: 0.3
  },
  balanced: {
    leverage_min: 3,
    leverage_max: 5,
    position_pct: 0.10,      // 10%
    sl_pct: 4,               // -4%
    tp_pct: 8,               // +8%
    confidence_min: 7,
    scan_interval: 7200,     // 2 hours
    slippage: 0.3,
    max_spread: 0.2
  },
  conservative: {
    leverage_min: 1,
    leverage_max: 2,
    position_pct: 0.06,      // 6%
    sl_pct: 2.5,             // -2.5%
    tp_pct: 5,               // +5%
    confidence_min: 8,
    scan_interval: 259200,   // 3 days
    slippage: 0.2,
    max_spread: 0.15
  }
}
```

---

## On Wake-Up Checklist

### Step 1: Get State
```javascript
const balance = await hyperliquid_get_balance({})
const positions = await hyperliquid_get_positions({})
const orders = await hyperliquid_get_open_orders({})
```

### Step 2: Update Peak Balance
```javascript
SESSION.PEAK_BALANCE = Math.max(SESSION.PEAK_BALANCE, balance.accountValue)
```

### Step 3: Check Drawdown Circuit Breaker
```javascript
const drawdown_pct = ((SESSION.PEAK_BALANCE - balance.accountValue) / SESSION.PEAK_BALANCE) * 100

if (drawdown_pct >= 20) {
  // HALT - close all, notify, stop
  await cleanup()
  STOP
}
if (drawdown_pct >= 15) {
  SESSION.POSITION_SIZE_MULTIPLIER = 0.5
  SESSION.PAUSE_NEW_ENTRIES_UNTIL = Date.now() + 2 * 60 * 60 * 1000
}
if (drawdown_pct >= 10) {
  SESSION.POSITION_SIZE_MULTIPLIER = 0.7
}
```

### Step 4: Manage Positions
```javascript
for (const position of positions) {
  // Dynamic stops
  await manage_dynamic_stop(position, SESSION.MODE)

  // Partial takes - pass TP from config
  const config = MODE_CONFIG[SESSION.MODE]
  await manage_partial_takes(position, config.tp_pct)
}
```

### Step 5: Check Target
```javascript
if (balance.accountValue >= SESSION.TARGET_BALANCE) {
  await cleanup()
  await send_target_reached_notification()
  STOP
}
```

### Step 6: Report Status
```javascript
await telegram_send_message({
  chat_id: TELEGRAM_CHAT_ID,
  text: build_status_message(positions, balance)
})
```

### Step 7: Look for New Trades (if slots available)
```javascript
const config = MODE_CONFIG[SESSION.MODE]
const max_positions = 3

if (positions.length < max_positions && !is_paused()) {
  // Run Steps 2-4 from mode file
}
```

### Step 8: Re-schedule (ALWAYS!)
```javascript
await schedule({
  subscription_id: SESSION.SUBSCRIPTION_ID,
  delay: MODE_CONFIG[SESSION.MODE].scan_interval,
  message: "Scheduled scan"
})
```

---

## Pre-Trade Checklist

Before EVERY trade, verify ALL:

```javascript
async function pre_trade_checks(coin, direction, confidence, mode) {
  const config = MODE_CONFIG[mode]
  const margin = balance.accountValue * config.position_pct * SESSION.POSITION_SIZE_MULTIPLIER

  // 1. Liquidity check
  const liquidity = await check_liquidity(coin, margin, mode)
  if (!liquidity.ok) return { ok: false, reason: liquidity.reason }

  // 2. BTC alignment
  const btc = await check_btc_alignment(coin, direction)
  confidence += btc.confidence_penalty

  // 3. Time conditions
  const time = check_trading_conditions()
  const final_margin = margin * time.multiplier

  // 4. Pause check
  if (SESSION.PAUSE_NEW_ENTRIES_UNTIL && Date.now() < SESSION.PAUSE_NEW_ENTRIES_UNTIL) {
    return { ok: false, reason: 'paused_due_to_drawdown' }
  }

  // 5. Confidence check
  if (confidence < config.confidence_min) {
    return { ok: false, reason: `confidence_too_low: ${confidence}` }
  }

  return {
    ok: true,
    final_margin: final_margin,
    final_confidence: confidence,
    time_multiplier: time.multiplier
  }
}
```

---

## Trade Execution Checklist

```javascript
async function execute_trade(coin, direction, mode) {
  const config = MODE_CONFIG[mode]
  const is_buy = direction === 'LONG'

  // 1. Set leverage
  const leverage = Math.min(maxLeverage, config.leverage_max)
  await hyperliquid_update_leverage({ coin, leverage, is_cross: true })

  // 2. Calculate prices
  const current_price = await hyperliquid_get_price(coin)
  const sl_price = current_price * (is_buy ? (1 - config.sl_pct/100) : (1 + config.sl_pct/100))
  const tp_price = current_price * (is_buy ? (1 + config.tp_pct/100) : (1 - config.tp_pct/100))

  // 3. Place entry with slippage protection
  const entry = await place_protected_order(coin, is_buy, size, mode)

  if (!entry.filled || entry.fill_pct < 0.7) {
    await telegram_send_message({ text: `⚠️ ${coin} order fill: ${entry.fill_pct * 100}%` })
    // Decide: retry or skip
  }

  // 4. Place TP/SL
  await hyperliquid_place_order({
    coin, is_buy: !is_buy, size: entry.filled_size,
    order_type: "take_profit", trigger_price: tp_price, reduce_only: true
  })
  await hyperliquid_place_order({
    coin, is_buy: !is_buy, size: entry.filled_size,
    order_type: "stop_loss", trigger_price: sl_price, reduce_only: true
  })

  // 5. Initialize partials tracking
  SESSION.partials_taken[coin] = { p50: false, p75: false }

  // 6. Notify
  await telegram_send_message({
    text: build_entry_message(coin, direction, entry, tp_price, sl_price)
  })
}
```

---

## Trade Close Checklist

```javascript
async function on_trade_closed(closed_position) {
  // 1. Record trade
  const pnl_pct = closed_position.realizedPnl / closed_position.marginUsed * 100

  SESSION.trade_history.push({
    coin: closed_position.coin,
    direction: closed_position.direction,
    pnl_pct: pnl_pct,
    exit_reason: closed_position.exit_reason,
    timestamp: Date.now()
  })

  // 2. Update stats
  SESSION.session_stats.total_trades++
  SESSION.session_stats.total_pnl += pnl_pct

  if (pnl_pct > 0) {
    SESSION.session_stats.wins++
    SESSION.consecutive_losses = 0
  } else {
    SESSION.session_stats.losses++
    SESSION.consecutive_losses++
    SESSION.daily_losses++
  }

  // 3. Check cooldown triggers
  if (SESSION.consecutive_losses >= 3) {
    SESSION.PAUSE_NEW_ENTRIES_UNTIL = Date.now() + 4 * 60 * 60 * 1000  // 4 hours
    SESSION.POSITION_SIZE_MULTIPLIER *= 0.5
    await telegram_send_message({ text: "⚠️ 3 losses in a row. Cooldown 4h, size -50%" })
  }

  if (SESSION.daily_losses >= 5) {
    SESSION.PAUSE_NEW_ENTRIES_UNTIL = Date.now() + 24 * 60 * 60 * 1000  // 24 hours
    await telegram_send_message({ text: "🛑 5 losses today. Stopping for 24h." })
  }

  // 4. Clean up partials tracking
  delete SESSION.partials_taken[closed_position.coin]

  // 5. Check re-entry opportunity (only for trailing stop exits)
  if (closed_position.exit_reason === 'trailing_stop' && pnl_pct > 0) {
    const reentry = await check_reentry_opportunity(closed_position)
    if (reentry.reentry) {
      // Re-enter the trade
    }
  }

  // 6. Performance report every 5 trades
  if (SESSION.session_stats.total_trades % 5 === 0) {
    await send_performance_report()
  }
}
```

---

## Dynamic Stop Levels Quick Reference

| Mode | +5% → | +8% → | +10% → | +12% → | +15% → | +20%+ → |
|------|-------|-------|--------|--------|--------|---------|
| Degen | BE | - | +5% | - | +10% | Trail 5% |
| Aggressive | BE | +4% | - | +8% | Trail 4% | Trail 4% |
| Balanced | +2% | +5% | Trail 3% | Trail 3% | Trail 3% | Trail 3% |
| Conservative | BE | +2% | Trail 2% | Trail 2% | Trail 2% | Trail 2% |

*BE = Breakeven (-0.2% to -0.3% for fees)*

---

## Partial Takes Quick Reference

| Progress to TP | Action |
|----------------|--------|
| 50% | Close 30% of position |
| 75% | Close 30% of position |
| 100% | Close remaining 40% |

---

## Error Handling

```javascript
// If schedule fails
if (schedule_error) {
  // Recreate subscription and schedule
  await recreate_subscription()
  await telegram_send_message({ text: "⚠️ Subscription expired. Recreated." })
}

// If position close fails
if (close_error) {
  await telegram_send_message({ text: `❌ Failed to close ${coin}: ${error}` })
  // Retry or alert user
}

// If order not filled
if (fill_pct < 0.7) {
  // Option 1: Retry with wider slippage
  // Option 2: Skip this trade
  // Option 3: Accept partial fill
}
```

---

## Daily Reset

```javascript
// Check if new day (UTC)
const today = new Date().toISOString().split('T')[0]
if (SESSION.last_trade_date !== today) {
  SESSION.daily_losses = 0
  SESSION.last_trade_date = today
}
```

---

## Never Forget

1. **ALWAYS re-schedule** — If you don't, agent dies
2. **ALWAYS check drawdown first** — Before any action
3. **NEVER move stop DOWN** — Only up or keep
4. **NEVER trade during pause** — Check PAUSE_NEW_ENTRIES_UNTIL
5. **ALWAYS notify user** — Every action to Telegram
