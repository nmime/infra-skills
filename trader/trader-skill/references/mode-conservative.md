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
| Position Size | 5-10% of account |
| Stop Loss | -2% to -3% |
| Take Profit | +5% to +8% |
| Trailing Stop | 4% after +5% profit |
| Max Positions | 6 (highly diversified) |
| Scan Interval | 3 days (72 hours) |

## Philosophy

```
┌─────────────────────────────────────────────────────────────┐
│                  CONSERVATIVE MINDSET                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✗ DON'T: Chase short-term moves                           │
│  ✓ DO:    Follow macro trends (weekly/monthly)             │
│                                                             │
│  ✗ DON'T: Use high leverage                                │
│  ✓ DO:    Trade near spot (1-2x)                           │
│                                                             │
│  ✗ DON'T: Risk more than 2% per trade                      │
│  ✓ DO:    Preserve capital above all                       │
│                                                             │
│  ✗ DON'T: Trade altcoins and shitcoins                     │
│  ✓ DO:    Stick to top 10 proven assets                    │
│                                                             │
│  ✗ DON'T: Overtrade                                        │
│  ✓ DO:    Wait for perfect setups (scan every 3 days)      │
│                                                             │
│  ✗ DON'T: Expect quick gains                               │
│  ✓ DO:    Compound slowly, beat inflation                  │
│                                                             │
│  "Slow money is safe money"                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Allowed Coins (Top 10 Only)

```
BTC  - Bitcoin
ETH  - Ethereum
SOL  - Solana
BNB  - Binance Coin
XRP  - Ripple
ADA  - Cardano
AVAX - Avalanche
DOGE - Dogecoin
LINK - Chainlink
DOT  - Polkadot
```

No exceptions. These have survived multiple cycles.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 CONSERVATIVE MODE LOOP                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   TRIGGER    │───▶│   RESEARCH   │───▶│   EXECUTE    │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                   │                   │          │
│         ▼                   ▼                   ▼          │
│  • Schedule (3 days)  • Macro trend check • 1-2x lever     │
│  • Event fires        • Weekly/monthly    • Small size     │
│                       • BTC direction     • Tight stops    │
│                       • Confidence 8+     • Top 10 only    │
│                       • Skip most scans   • Diversify      │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              ENTRY REQUIREMENTS (strict)              │  │
│  │                                                        │  │
│  │  1. MACRO TREND: Weekly chart clear direction          │  │
│  │  2. BTC ALIGNED: Trade with BTC, not against           │  │
│  │  3. NOT OVERBOUGHT: RSI/momentum not extreme           │  │
│  │  4. CONFIDENCE: 8+ out of 10                           │  │
│  │                                                        │  │
│  │  Most scans = no trade. That's expected.               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              PORTFOLIO MANAGEMENT                     │  │
│  │  • Max 6 positions (highly diversified)              │  │
│  │  • Max 10% margin per position                        │  │
│  │  • Always keep 40%+ in cash                          │  │
│  │  • Rebalance quarterly                               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              QUARTERLY CHECK                          │  │
│  │  • Every 90 days: Review performance                 │  │
│  │  • On track for +20%? Continue                       │  │
│  │  • Ahead of schedule? Maybe reduce risk              │  │
│  │  • Behind? Stay patient, don't chase                 │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

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
  context_memory_id: "{chat_id}_degen_trading_session",
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

// Small position size (5-10%)
MARGIN = accountValue * 0.08  // 8% default

// Place bracket order with very tight stops
hyperliquid_place_bracket_order({
  coin: "COIN",
  is_buy: true,  // or false for SHORT
  size: POSITION_SIZE,
  entry_price: ENTRY_PRICE,
  take_profit_price: TP_PRICE,  // +5-8%
  stop_loss_price: SL_PRICE     // -2-3%
})
```

### Step 5: Setup Monitoring

```javascript
// Subscribe for all positions
hyperliquid_subscribe_webhook({
  webhook_url: WEBHOOK_URL,
  coins: POSITION_COINS,
  events: ["fills", "orders"],
  position_alerts: [
    { condition: "pnl_pct_gt", value: 4 },   // Early profit
    { condition: "pnl_pct_lt", value: -1.5 } // Early warning
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

### On Trade Event (fill/order)

1. Check what happened (TP hit? SL hit?)
2. Update portfolio tracking
3. Log result
4. If position closed → Slot opens (but don't rush to fill)

### On Position Alert

1. +4% profit → Enable trailing stop (4%)
2. -1.5% loss → Review, consider early exit if trend changed
3. Report status

### On Schedule (3-day scan)

1. **Portfolio status report:**
   ```javascript
   "📊 3-Day Report:
      Positions: X/6
      Cash: Y%
      Total P&L: +Z%
      Annual target: $T (+20%)
      Progress: W%"
   ```

2. **Check quarterly milestone:**
   ```javascript
   days_elapsed = (now - START_DATE).days
   
   if (days_elapsed >= 90 && !Q1_CHECKED) {
     Q1_CHECKED = true
     "📅 Q1 Review: $X / $Q1_TARGET"
   }
   // ... same for Q2, Q3, Q4
   ```

3. **Research if room for trades:**
    - If < 6 positions AND cash > 40% → Research
    - If confidence 8+ found → Maybe trade
    - If unclear → Skip, that's fine

4. **Re-schedule:**
   ```javascript
   schedule({
     subscription_id: SUBSCRIPTION_ID,
     delay: 259200,  // 3 days
     message: "3-day scan: Check macro conditions"
   })
   ```

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
Per Position: 5-10% margin (8% default)
Leverage: 1-2x (2x default)
Cash Reserve: Always keep 40%+

Example ($1,000 account):
- Per position: $80 margin (8%)
- Leverage: 2x
- Notional per position: $160
- Max 6 positions = $480 margin (48%)
- Cash reserve: $520 (52%)
```

## Notifications

All actions are logged. User sees:
- "🛡️ Conservative mode started: $1,000 → Annual target: $1,200 (+20%)"
- "📊 3-day scan: Market unclear, staying in cash. Positions: 2/6"
- "Opened {LEVERAGE}x {DIRECTION} on {COIN} @ ${ENTRY} (Confidence: {CONF}/10, Weekly {TREND})"
- "📊 3-day report: ${BALANCE} (+{PNL_PCT}%), on track for +20% annual"
- "Closed {COIN} @ ${EXIT}, P&L: +${PNL} (+{PNL_PCT}%)"
- "📅 Q{QUARTER} Review: +{QUARTERLY_PCT}%, {STATUS} of +{QUARTERLY_TARGET}% target. Continuing strategy."
- "[1 year later] 🎉 Annual target hit! ${STARTING} → ${FINAL} (+{RETURN}%)"

Patient. Protected. Compounding.