# Degen Mode

**Strategy:** Momentum + News | Full Autonomy | Any Coin

## Target

Agent chooses target at start: **+100% to +300%**

| Account Size | Target Range | Reasoning |
|--------------|--------------|----------|
| < $100 | +200-300% | Small stack, swing big |
| $100-1000 | +150-200% | Moderate growth |
| > $1000 | +100-150% | Preserve gains |

Minimum target: **+100% (2x)** — never lower.

Agent runs until target reached or stopped manually.

```
Example:
Starting Balance: $50
Agent picks: +250% target
Target Balance: $175

Agent trades until $175 hit.
```

## Parameters

| Parameter | Value |
|-----------|-------|
| Target | +100% to +300% (agent chooses) |
| Leverage | 15-25x (max 25x) |
| Position Size | 20-30% of account |
| Stop Loss | -8% to -12% |
| Take Profit | +20% to +30% (min 2× SL) |
| Max Positions | 3 |
| Scan Interval | 10 minutes |
| Daily Loss Limit | -15% |

## Dynamic Stop-Loss Levels

| Position P&L | Move Stop To | Locked Profit |
|--------------|--------------|---------------|
| +5% | Breakeven (-0.3%) | Risk-free |
| +10% | +5% | +5% guaranteed |
| +15% | +10% | +10% guaranteed |
| +20%+ | Trail 5% below max | Ride the trend |

## Setup Flow

### Step 0: Set Target

```javascript
// Get starting balance
hyperliquid_get_balance({})
STARTING_BALANCE = accountValue

// Agent chooses target based on account size
if (STARTING_BALANCE < 100) {
  TARGET_PCT = 250  // +250% for small accounts
} else if (STARTING_BALANCE < 1000) {
  TARGET_PCT = 175  // +175% for medium
} else {
  TARGET_PCT = 125  // +125% for larger
}

TARGET_BALANCE = STARTING_BALANCE * (1 + TARGET_PCT / 100)

// Log
"🎰 Degen mode: $X → Target: $Y (+Z%)"
```

### Step 1: Create Event Hub Webhook

```javascript
event_create_webhook({
  label: "hyperliquid_degen"
})
// Save webhook_id and webhook_url
```

### Step 2: Market Research

Always include chat_id as a prefix
```javascript
market_deepresearch({
  context_memory_id: "{chat_id}_degen_trading_session",
  message: `Quick scan (1-2 min max): Find the best momentum trade RIGHT NOW on Hyperliquid perpetuals.

Check:
1. Top movers in last 1-4 hours (biggest pumps/dumps)
2. Any breaking news or catalysts for crypto coins
3. Extreme funding rates (crowded trades to fade or ride)

I need ONE trade recommendation:
- Coin (available on Hyperliquid perps)
- Direction (LONG or SHORT)
- Why (catalyst/momentum reason)
- Confidence (1-10)

Looking for volatile plays - shitcoins welcome.`
})
```

### Step 3: Validate Coin on Hyperliquid

```javascript
// Check if coin exists and get max leverage
hyperliquid_get_meta({ coin: "COIN" })

// Get current price
hyperliquid_get_all_prices({ coins: ["COIN"] })

// Check funding rate
hyperliquid_get_funding_rates({ coin: "COIN" })
```

### Step 4: Execute Trade

```javascript
// Set leverage (15-25x, cap at 25x)
const LEVERAGE = Math.min(maxLeverage, 25)

hyperliquid_update_leverage({
  coin: "COIN",
  leverage: LEVERAGE,
  is_cross: true
})

// Calculate position size (20-30% of account)
const MARGIN = accountValue * 0.25  // 25% default

// Calculate TP/SL with enforced 2:1 minimum R:R
const SL_PCT = 10  // -10%
const TP_PCT = Math.max(SL_PCT * 2, 20)  // Minimum 2× SL = +20%

const SL_PRICE = ENTRY_PRICE * (is_buy ? (1 - SL_PCT/100) : (1 + SL_PCT/100))
const TP_PRICE = ENTRY_PRICE * (is_buy ? (1 + TP_PCT/100) : (1 - TP_PCT/100))

// Place bracket order (entry + TP + SL)
hyperliquid_place_bracket_order({
  coin: "COIN",
  is_buy: true,  // or false for SHORT
  size: POSITION_SIZE,
  entry_price: ENTRY_PRICE,
  take_profit_price: TP_PRICE,  // +20-30% (2× SL minimum)
  stop_loss_price: SL_PRICE     // -8-12%
})
```

### Step 5: Setup Monitoring

```javascript
// Get all current position coins
hyperliquid_get_positions({})
ALL_COINS = positions.map(p => p.coin)

// Subscribe all coins to webhook with dynamic stop levels
hyperliquid_subscribe_webhook({
  webhook_url: WEBHOOK_URL,
  coins: ALL_COINS,
  events: ["fills", "orders"],
  position_alerts: ALL_COINS.map(coin => [
    { coin: coin, condition: "pnl_pct_gt", value: 5 },   // Breakeven trigger
    { coin: coin, condition: "pnl_pct_gt", value: 10 },  // +5% lock trigger
    { coin: coin, condition: "pnl_pct_gt", value: 15 },  // +10% lock trigger
    { coin: coin, condition: "pnl_pct_gt", value: 20 },  // Trailing start
    { coin: coin, condition: "pnl_pct_lt", value: -5 }   // Danger zone
  ]).flat()
})

// Subscribe to Event Hub to wake up on events
event_subscribe({
  webhook_id: WEBHOOK_ID,
  timeout: 86400,  // 24 hours
  triggers: [
    { name: "trade_events", filter: "payload.type == 'fill' || payload.type == 'order'", debounce: 5 },
    { name: "position_alerts", filter: "payload.type == 'position_alert'", debounce: 5 }
  ]
})
// Save subscription_id
```

### Step 6: Schedule Periodic Scans

```javascript
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 600,  // 10 minutes
  message: "10min scan: Check positions, scan for new momentum plays"
})
// Save schedule_id
```

## Event Handling

**CRITICAL: Every event MUST end with re-schedule. Never skip.**

### On ANY Wake-up (first thing)

```javascript
// 1. Get current state
hyperliquid_get_balance({})
hyperliquid_get_positions({})

// 2. CHECK DRAWDOWN CIRCUIT BREAKER (before anything else!)
const drawdown_check = await check_drawdown_circuit_breaker()
if (drawdown_check.halt) {
  // Trading halted, cleanup already done
  STOP
}

// 3. Manage dynamic stops for ALL positions
for (const position of positions) {
  await manage_dynamic_stop(position, 'degen')
}

// 4. Report ALL positions to Telegram
telegram_send_message({
  chat_id: TELEGRAM_CHAT_ID,
  text: `📊 Positions:
${positions.map(p => `• ${p.coin} ${p.direction}: ${p.roe}% ROE`).join('\n')}
Balance: $${balance} (${sessionChange}%)
Progress: ${progress}% to target`
})

// 5. Check target
if (accountValue >= TARGET_BALANCE) {
  cleanup()
  STOP
}
```

### On Trade Event (fill/order)

1. Report what happened (TP/SL hit?)
2. If closed by trailing stop with profit:
   - Check re-entry opportunity (see SKILL.md `check_reentry_opportunity`)
   - If trend continues → re-enter
3. If position closed → research new trade (Steps 2-4)

### On Position Alert

Dynamic stops are managed automatically, but on specific alerts:

| Alert | Action |
|-------|--------|
| +5% | Move stop to breakeven |
| +10% | Lock +5% profit |
| +15% | Lock +10% profit |
| +20%+ | Trail 5% behind price |
| -5% | Watch closely, DO NOT move stop down |

### On Schedule (10min scan)

1. Report position status (see above)
2. If < 3 positions → Research new trade
3. If 3 positions → Just monitor

### LAST STEP (every event, never skip)

```javascript
// Always re-schedule before ending
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 600,  // 10 minutes
  message: "10min scan"
})

// If schedule fails (subscription expired):
// → Run Step 5-6 again to recreate subscription + schedule
```

**If you don't re-schedule, the agent dies.**

## Cleanup (on stop or target reached)

```javascript
// Close all positions
hyperliquid_get_positions({})
for (pos of positions) {
  hyperliquid_market_close({ coin: pos.coin })
}

// Cancel schedule
cancel_schedule({ schedule_id: SCHEDULE_ID })

// Unsubscribe from events
event_unsubscribe({ subscription_id: SUBSCRIPTION_ID })

// Remove Hyperliquid webhook
hyperliquid_unsubscribe_webhook({})

// Final report
hyperliquid_get_balance({})
"Session ended: $STARTING → $FINAL (±X%)"
```

## Position Sizing

```
Account Balance: $X
Position Size: 20-30% of $X (25% default)
Max Leverage: 15-25x (capped at 25x)
Notional Value: Position Size × Leverage

Example ($100 account, 3 positions):
- Per position: $25 margin (25%)
- Leverage: 20x
- Notional per position: $500
- Total exposure: $1,500 (3 positions)
- Max loss per position (10% SL): $2.50
- Max loss all positions: $7.50 (7.5% of account)
```

## Risk Controls

| Control | Value |
|---------|-------|
| Daily Loss Limit | -15% → Stop for day |
| 3 Consecutive Losses | Cooldown 4-6 hours, size -50% |
| 5 Losses in Day | Stop for 24 hours |
| Drawdown -15% | Reduce size 50%, pause 2 hours |
| Drawdown -20% | **HALT ALL TRADING** |
| Min R:R | 2:1 (enforced) |

## Notifications

All actions are logged. User sees:
- "🎰 Degen mode started: $100 → Target: $275 (+175%)"
- "Opened 20x LONG on SOL @ $95, catalyst: ETF momentum"
- "SOL +15%, trailing stop active"
- "Closed SOL @ $100, P&L: +$15 (+15%)"
- "Progress: $115 / $275 (15% to target)"
- "🎉 TARGET REACHED! $100 → $280 (+180%)"

No questions asked. Execute and report.
