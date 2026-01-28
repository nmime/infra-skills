# Aggressive Mode

**Strategy:** Momentum + Trend Confirmation | Full Autonomy | Top 50 Coins

## Target

Agent chooses target at start: **+50% to +100%**

| Account Size | Target Range | Reasoning |
|--------------|--------------|----------|
| < $100 | +100% | Small stack, aim to double |
| $100-1000 | +75% | Solid growth |
| > $1000 | +50% | Meaningful gains |

Minimum target: **+50%** — never lower.

Agent runs until target reached or stopped manually.

```
Example:
Starting Balance: $200
Agent picks: +75% target
Target Balance: $350

Agent trades until $350 hit.
```

## Parameters

| Parameter | Value |
|-----------|-------|
| Target | +50% to +100% (agent chooses) |
| Leverage | 10-20x |
| Position Size | 15-25% of account |
| Stop Loss | -5% to -8% |
| Take Profit | +10% to +15% |
| Trailing Stop | 10% after +12% profit |
| Max Positions | 3 |
| Scan Interval | 20 minutes |

## Setup Flow

### Step 0: Set Target

```javascript
// Get starting balance
hyperliquid_get_balance({})
STARTING_BALANCE = accountValue

// Agent chooses target based on account size
if (STARTING_BALANCE < 100) {
  TARGET_PCT = 100  // +100% for small accounts
} else if (STARTING_BALANCE < 1000) {
  TARGET_PCT = 75   // +75% for medium
} else {
  TARGET_PCT = 50   // +50% for larger
}

TARGET_BALANCE = STARTING_BALANCE * (1 + TARGET_PCT / 100)

// Log
"🚀 Aggressive mode: $X → Target: $Y (+Z%)"
```

### Step 1: Create Event Hub Webhook

```javascript
event_create_webhook({
  label: "hyperliquid_aggressive"
})
// Save webhook_id and webhook_url
```

### Step 2: Market Research

Always include chat_id as a prefix
```javascript
market_deepresearch({
  context_memory_id: "{chat_id}_aggressive_trading_session",
  message: `Quick scan (1-2 min max): Find the best momentum trade RIGHT NOW on Hyperliquid perpetuals.

Requirements:
1. Top 50 coins by volume only (no ultra-low liquidity)
2. Clear trend direction (up or down)
3. Recent catalyst or news preferred
4. Check funding rates for crowd positioning

I need ONE trade recommendation:
- Coin (top 50 on Hyperliquid)
- Direction (LONG or SHORT)
- Why (trend + catalyst)
- Confidence (must be 6+ out of 10 to trade)

Looking for confirmed momentum, not speculation.`
})
```

**Trade Requirement:** Only execute if confidence >= 6

### Step 3: Validate Coin on Hyperliquid

```javascript
// Check if coin exists and get max leverage
hyperliquid_get_meta({ coin: "COIN" })

// Get current price
hyperliquid_get_all_prices({ coins: ["COIN"] })

// Check funding rate
hyperliquid_get_funding_rates({ coin: "COIN" })

// Verify max leverage >= 10x, otherwise skip
if (maxLeverage < 10) {
  "Skipping COIN - leverage too low for aggressive mode"
  // Research another coin
}
```

### Step 4: Execute Trade

```javascript
// Set leverage (10-20x, use min of maxLeverage and 20)
LEVERAGE = Math.min(maxLeverage, 20)
LEVERAGE = Math.max(LEVERAGE, 10)  // At least 10x

hyperliquid_update_leverage({
  coin: "COIN",
  leverage: LEVERAGE,
  is_cross: true
})

// Calculate position size (15-25% of account)
MARGIN = accountValue * 0.20  // 20% default

// Place bracket order (entry + TP + SL)
hyperliquid_place_bracket_order({
  coin: "COIN",
  is_buy: true,  // or false for SHORT
  size: POSITION_SIZE,
  entry_price: ENTRY_PRICE,
  take_profit_price: TP_PRICE,  // +10-15%
  stop_loss_price: SL_PRICE     // -5-8%
})
```

### Step 5: Setup Monitoring

```javascript
// Get all current position coins
hyperliquid_get_positions({})
ALL_COINS = positions.map(p => p.coin)

// Subscribe all coins to webhook
hyperliquid_subscribe_webhook({
  webhook_url: WEBHOOK_URL,
  coins: ALL_COINS,
  events: ["fills", "orders"],
  position_alerts: ALL_COINS.map(coin => [
    { coin: coin, condition: "pnl_pct_gt", value: 8 },
    { coin: coin, condition: "pnl_pct_lt", value: -4 }
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
  delay: 1200,  // 20 minutes
  message: "20min scan: Check positions, scan for new momentum plays"
})
// Save schedule_id
```

## Event Handling

### On Trade Event (fill/order)

1. Check what happened (TP hit? SL hit? Partial fill?)
2. Update position tracking
3. **Check if target reached:**
   ```javascript
   hyperliquid_get_balance({})
   if (accountValue >= TARGET_BALANCE) {
     // TARGET HIT!
     cleanup()
     "🎉 TARGET REACHED! $X → $Y (+Z%)"
     STOP
   }
   ```
4. If position closed and target NOT reached → Run Steps 2-6 to find next trade
5. If position still open → Continue monitoring

### On Position Alert

1. Check P&L status
2. If +12% → Add trailing stop at 10%
3. If -4% → Watch closely, consider cutting early
4. Report status

### On Schedule (20min scan)

1. **Check if target reached first**
2. Check current positions
3. If < 3 positions → Run Steps 2-6 (research and open new trade)
4. Re-schedule next scan

```javascript
// Re-schedule for next 20min scan
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 1200,
  message: "20min scan: Check positions, scan for new momentum plays"
})
```

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
Position Size: 15-25% of $X (20% default)
Leverage: 10-20x
Notional Value: Position Size × Leverage

Example ($100 account):
- Position Size: $20 margin (20%)
- Leverage: 15x
- Notional: $300
- Max 3 positions = $60 margin total (60% account)
```

## Coin Selection

Only trade coins that meet:
- Top 50 by volume on Hyperliquid
- Max leverage >= 10x
- Clear trend or catalyst

Examples: BTC, ETH, SOL, AVAX, ARB, OP, DOGE, LINK, etc.

Avoid: Ultra-low liquidity, brand new listings, < 10x leverage

## Notifications

All actions are logged. User sees:
- "🚀 Aggressive mode started: $100 → Target: $175 (+75%)"
- "Opened 15x LONG on SOL @ $124, trend: breakout above resistance"
- "SOL +8%, trailing stop active (Balance: $116, Target: $175)"
- "Closed SOL @ $130, P&L: +$18 (+15.6%)"
- "Progress: $118 / $175 (24% to target)"
- "🎉 TARGET REACHED! $100 → $178 (+78%)"

No questions asked. Execute and report.
