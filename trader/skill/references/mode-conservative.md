# Conservative Mode

**Strategy:** Macro Trends + Capital Preservation + Long-Term Compounding | Full Autonomy | Top 10 Coins Only

## Target

**Goal: +20% annually**

Agent runs indefinitely until stopped. Checks progress quarterly.

```
Example:
Starting Balance: $1,000
Annual Target: $1,200 (+20%)

Quarterly milestones:
  Q1: $1,050 (+5%)
  Q2: $1,100 (+10%)
  Q3: $1,150 (+15%)
  Q4: $1,200 (+20%) → Target hit, continue or stop
```

If target hit early → Report success, keep running (compound gains).

## Parameters

| Parameter | Value |
|-----------|-------|
| Target | +20% annually |
| Leverage | 1-2x (near spot) |
| Position Size | 5-8% of account |
| Stop Loss | -2% to -3% |
| Take Profit | +5% to +8% (min 2× SL) |
| Max Positions | 6 (highly diversified) |
| Scan Interval | 3 days (72 hours) |
| Daily Loss Limit | -5% (hard stop) |

## Dynamic Stop-Loss Levels

| Position P&L | Move Stop To | Locked Profit |
|--------------|--------------|---------------|
| +2.5% | Breakeven (-0.1%) | Risk-free |
| +4% | +2% | +2% guaranteed |
| +6%+ | Trail 2% below max | Ride the trend |

## Allowed Coins

Top 10 only: BTC, ETH, SOL, BNB, XRP, ADA, AVAX, DOGE, LINK, DOT

## Setup Flow

### Step 0: Initialize Long-Term Tracking

```javascript
// Get starting balance
hyperliquid_get_balance({})
STARTING_BALANCE = accountValue
START_DATE = now()

// Calculate annual target
TARGET_PCT = 20
TARGET_BALANCE = STARTING_BALANCE * 1.20

// Calculate quarterly milestones
Q1_TARGET = STARTING_BALANCE * 1.05  // +5%
Q2_TARGET = STARTING_BALANCE * 1.10  // +10%
Q3_TARGET = STARTING_BALANCE * 1.15  // +15%
Q4_TARGET = STARTING_BALANCE * 1.20  // +20%

// Log
"🛡️ Conservative mode started
   Starting: $X
   Annual target: $Y (+20%)
   Scanning every 3 days
   This is a marathon, not a sprint."
```

### Step 1: Create Event Hub Webhook

```javascript
event_create_webhook({
  label: "hyperliquid_conservative"
})
// Save webhook_id and webhook_url
```

### Step 2: Market Research (Macro Focus)

Always include chat_id as a prefix
```javascript
market_deepresearch({
  context_memory_id: "{chat_id}_conservative_session",
  message: `Quick scan (1-2 min max): Find the best momentum trade RIGHT NOW on Hyperliquid perpetuals.

Analyze:
1. BTC weekly trend - Is the macro direction clear?
2. Top 10 coins only: BTC, ETH, SOL, BNB, XRP, ADA, AVAX, DOGE, LINK, DOT
3. Weekly/monthly chart trends
4. Are we in accumulation, markup, distribution, or markdown phase?
5. Any major macro events coming? (halving, ETF, regulation)

Rules:
- Only LONG in uptrends, only SHORT in confirmed downtrends
- Follow BTC - don't fight the king
- Skip if unclear - most scans should result in no trade
- Confidence must be 8+ to trade

I need:
- Market assessment (bullish / bearish / unclear)
- If clear: Up to 2 trade ideas from top 10
- If unclear: Recommend waiting

Current positions: [LIST CURRENT POSITIONS]
Cash available: [CASH %]

Remember: It's OK to do nothing. Capital preservation first.`
})
```

**Trade Requirement:**
- Confidence >= 8
- Macro trend clear
- BTC aligned
- Most scans = no trade (expected)

### Step 3: Validate Coins

```javascript
// Only allow top 10
ALLOWED_COINS = ["BTC", "ETH", "SOL", "BNB", "XRP", "ADA", "AVAX", "DOGE", "LINK", "DOT"]

if (!ALLOWED_COINS.includes(coin)) {
  "Coin not in top 10, skipping"
  SKIP
}

hyperliquid_get_meta({ coin: "COIN" })
hyperliquid_get_all_prices({ coins: ["COIN"] })
hyperliquid_get_funding_rates({ coin: "COIN" })
```

### Step 4: Execute Trade (Small & Safe)

```javascript
// Check portfolio constraints
hyperliquid_get_positions({})

// Rules:
// - Max 6 positions
// - Max 60% total margin (keep 40% cash)
// - Max 10% per position

current_margin_pct = totalMarginUsed / accountValue
if (current_margin_pct > 0.60) {
  "Portfolio at max allocation (60%), keeping cash reserve"
  SKIP
}

if (positions.length >= 6) {
  "Max positions reached (6), waiting for exits"
  SKIP
}

// Set low leverage (1-2x)
LEVERAGE = 2  // Conservative default

hyperliquid_update_leverage({
  coin: "COIN",
  leverage: LEVERAGE,
  is_cross: true
})

// Small position size (5-8%)
MARGIN = accountValue * 0.06  // 6% default

// Calculate TP/SL with enforced 2:1 minimum R:R
const SL_PCT = 2.5  // -2.5%
const TP_PCT = Math.max(SL_PCT * 2, 5)  // Minimum 2× SL = +5%

const SL_PRICE = ENTRY_PRICE * (is_buy ? (1 - SL_PCT/100) : (1 + SL_PCT/100))
const TP_PRICE = ENTRY_PRICE * (is_buy ? (1 + TP_PCT/100) : (1 - TP_PCT/100))

// Place bracket order with very tight stops
hyperliquid_place_bracket_order({
  coin: "COIN",
  is_buy: true,  // or false for SHORT
  size: POSITION_SIZE,
  entry_price: ENTRY_PRICE,
  take_profit_price: TP_PRICE,  // +5-8% (2× SL minimum)
  stop_loss_price: SL_PRICE     // -2-3%
})
```

### Step 5: Setup Monitoring

```javascript
// Subscribe for all positions
// Note: If multiple coins, duplicate this call for each coin
hyperliquid_subscribe_webhook({
  webhook_url: WEBHOOK_URL,
  coins: ["COIN"],
  events: ["fills", "orders"],
  position_alerts: [
    { coin: "COIN", condition: "pnl_pct_gt", value: 2.5 },  // Breakeven trigger
    { coin: "COIN", condition: "pnl_pct_gt", value: 4 },    // +2% lock trigger
    { coin: "COIN", condition: "pnl_pct_gt", value: 6 },    // Trailing start
    { coin: "COIN", condition: "pnl_pct_lt", value: -1.5 }  // Early warning
  ]
})

// Subscribe to Event Hub
event_subscribe({
  webhook_id: WEBHOOK_ID,
  timeout: 2592000,  // 30 days (max)
  triggers: [
    { name: "trade_events", filter: "payload.type == 'fill' || payload.type == 'order'", debounce: 10 },
    { name: "position_alerts", filter: "payload.type == 'position_alert'", debounce: 10 }
  ]
})
```

### Step 6: Schedule 3-Day Scans

```javascript
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 259200,  // 3 days = 72 hours
  message: "3-day scan: Check macro conditions, review portfolio"
})
```

## Event Handling

**CRITICAL: Every event MUST end with re-schedule. Never skip.**

### On ANY Wake-up (first thing)

```javascript
// 1. Get current state
hyperliquid_get_balance({})
hyperliquid_get_positions({})

// 2. CHECK DRAWDOWN CIRCUIT BREAKER
const drawdown_check = await check_drawdown_circuit_breaker()
if (drawdown_check.halt) {
  STOP
}

// 3. Manage dynamic stops for ALL positions
for (const position of positions) {
  await manage_dynamic_stop(position, 'conservative')
}

// 4. Report to Telegram
telegram_send_message({
  chat_id: TELEGRAM_CHAT_ID,
  text: `📊 3-Day Report:
Positions: ${positions.length}/6
Cash: ${cashPct}%
Total P&L: ${totalPnl}%
Annual target: $${target} (+20%)
Progress: ${progress}%`
})
```

### On Trade Event (fill/order)

1. Report what happened (TP/SL hit?)
2. If closed by trailing stop with profit → check re-entry opportunity
3. If position closed → slot opens (don't rush to fill)

### On Position Alert

| Alert | Action |
|-------|--------|
| +2.5% | Move stop to breakeven |
| +4% | Lock +2% profit |
| +6%+ | Trail 2% behind price |
| -1.5% | Review, consider early exit |

### On Schedule (3-day scan)

1. Report portfolio status (see above)
2. Check quarterly milestone (Q1=90d, Q2=180d, Q3=270d, Q4=365d)
3. If < 6 positions AND cash > 40% AND confidence 8+ → Maybe trade
4. If unclear → Skip (that's fine for conservative)

### LAST STEP (every event, never skip)

```javascript
// Always re-schedule before ending
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 259200,  // 3 days
  message: "3-day scan"
})

// If schedule fails (subscription expired):
// → Run Step 5-6 again to recreate subscription + schedule
```

**If you don't re-schedule, the agent dies.**

## Quarterly Review

Every 90 days, comprehensive review:

```
📅 QUARTERLY REVIEW

Period: Q1 2026
Started: $1,000
Current: $1,080 (+8%)
Target pace: +5% → Ahead of schedule ✓

Trades this quarter:
  • {TOTAL_TRADES} total trades
  • {WINS} wins, {LOSSES} losses
  • Win rate: {WIN_RATE}%
  • Avg win: +{AVG_WIN}%
  • Avg loss: -{AVG_LOSS}%

Positions:
  [For each position: • {COIN} {DIRECTION}: +{PNL_PCT}%]
  • Cash: {CASH_PCT}%

Assessment: On track. Continue strategy.
Next review: Q2 2026
```

## Cleanup (on manual stop)

```javascript
// Close all positions
for each position:
  hyperliquid_market_close({ coin: position.coin })

// Cancel schedule
cancel_schedule({ schedule_id: SCHEDULE_ID })

// Unsubscribe
event_unsubscribe({ subscription_id: SUBSCRIPTION_ID })
hyperliquid_unsubscribe_webhook({})

// Final report
"Conservative mode stopped
   Duration: X months
   Starting: $Y
   Final: $Z
   Return: +W%
   Annualized: +V%
   Trades: N total (W wins, L losses)"
```

## Position Sizing

```
Account Balance: $X
Max Positions: 6
Per Position: 5-8% margin (6% default)
Leverage: 1-2x (2x default)
Cash Reserve: Always keep 40%+

Example ($1,000 account):
- Per position: $60 margin (6%)
- Leverage: 2x
- Notional per position: $120
- Max 6 positions = $360 margin (36%)
- Cash reserve: $640 (64%)
- Max loss per position (2.5% SL): $1.50
- Max loss all positions: $9 (0.9% of account)
```

## Risk Controls

| Control | Value |
|---------|-------|
| Daily Loss Limit | -5% → Stop for day |
| 3 Consecutive Losses | Cooldown 4-6 hours, size -50% |
| 5 Losses in Day | Stop for 24 hours |
| Drawdown -15% | Reduce size 50%, pause 2 hours |
| Drawdown -20% | **HALT ALL TRADING** |
| Min R:R | 2:1 (enforced) |

## Notifications

- "🛡️ Conservative: ${STARTING} → Annual: ${TARGET} (+20%) | Scan: 3d"
- "🔍 Market unclear, staying cash | Next: 3d"
- "🎯 {DIRECTION} {COIN} @ ${ENTRY} | {LEV}x | Conf: {CONF}/10 | Next: 3d"
- "📊 Report: ${BALANCE} (+{PNL_PCT}%) | Pos: {POS}/{MAX} | Next: 3d"
- "✅ WIN +${PNL} | {W}W/{L}L | Next: 3d"
- "❌ LOSS ${PNL} | Next: 3d"
- "📅 Q{Q} Review: +{PCT}% | Next: 3d"
- "🎉 ANNUAL TARGET! ${STARTING} → ${FINAL} (+{RETURN}%)"

Patient. Protected. Compounding.