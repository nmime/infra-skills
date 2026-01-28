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
| Leverage | Max available (up to 50x) |
| Position Size | 30-50% of account |
| Stop Loss | -10% to -15% |
| Take Profit | +15% to +25% |
| Trailing Stop | 15% after +20% profit |
| Max Positions | 3 |
| Scan Interval | 10 minutes |

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
// Set leverage (use max available, up to 50x)
hyperliquid_update_leverage({
  coin: "COIN",
  leverage: MAX_LEVERAGE,
  is_cross: true
})

// Place bracket order (entry + TP + SL)
hyperliquid_place_bracket_order({
  coin: "COIN",
  is_buy: true,  // or false for SHORT
  size: POSITION_SIZE,
  entry_price: ENTRY_PRICE,
  take_profit_price: TP_PRICE,  // +15-25%
  stop_loss_price: SL_PRICE     // -10-15%
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
    { coin: coin, condition: "pnl_pct_gt", value: 10 },
    { coin: coin, condition: "pnl_pct_lt", value: -5 }
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
2. If +20% → Consider adding trailing stop
3. If -5% → Watch closely, prepare to cut
4. Report status

### On Schedule (10min scan)

1. **Check if target reached first**
2. Check current positions
3. If < 3 positions → Run Steps 2-6 (research and open new trade)
4. Re-schedule next scan

```javascript
// Re-schedule for next 10min scan
schedule({
  subscription_id: SUBSCRIPTION_ID,
  delay: 600,
  message: "10min scan: Check positions, scan for new momentum plays"
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
Position Size: 30-50% of $X
Max Leverage: Use coin's max (check hyperliquid_get_meta)
Notional Value: Position Size × Leverage

Example ($100 account, 3 positions):
- Per position: $30 margin (30%)
- Leverage: 20x
- Notional per position: $600
- Total exposure: $1,800 (3 positions)
```

## Notifications

All actions are logged. User sees:
- "🎰 Degen mode started: $100 → Target: $275 (+175%)"
- "Opened 20x LONG on SOL @ $95, catalyst: ETF momentum"
- "SOL +15%, trailing stop active"
- "Closed SOL @ $100, P&L: +$15 (+15%)"
- "Progress: $115 / $275 (15% to target)"
- "🎉 TARGET REACHED! $100 → $280 (+180%)"

No questions asked. Execute and report.
