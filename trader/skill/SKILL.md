---
name: trader-agent
description: Autonomous AI trading agent for Hyperliquid perpetual futures exchange. Use when user wants to trade crypto, set up trading automation, monitor positions, analyze markets, or manage a Hyperliquid account. Trigger on any trading-related message like "I want to trade", "What's my balance?", "Start trading", "Close positions", etc.
---

# Trading Agent

Autonomous trading agent for Hyperliquid that monitors markets, analyzes opportunities, and executes trades.

## Architecture

The agent operates in cycles:
1. **Monitor** - Listen for market events via webhooks
2. **Analyze** - Validate signals against strategy rules
3. **Execute** - Place trades on Hyperliquid
4. **Track** - Monitor positions and adjust

## Setup Flow

### Step 0: Get Telegram Chat ID (once per conversation)

First message in any conversation, ask for Telegram:

```
To send you trade notifications, I need your Telegram chat ID.

How to find it:
1. Message @userinfobot on Telegram
2. It will reply with your chat ID (a number like 123456789)

Your Telegram chat ID:
```

Save as `TELEGRAM_CHAT_ID` for all notifications.

If user says "skip" or "no" → `TELEGRAM_CHAT_ID = null` (no notifications).

Only ask once. Remember for entire conversation.

---

### On ANY Trading Message

**Step 1: Check if Hyperliquid is connected**

```javascript
SPLOX_SEARCH_TOOLS(query: "hyperliquid")
// Check is_user_connected field
```

**Step 2a: If NOT Connected**

Show connection instructions:

```
To trade on Hyperliquid, you need to connect an agent wallet.

1. Create an Agent Wallet (if you don't have one)
   - Go to: https://app.hyperliquid.xyz/API
   - Click "Create API Wallet"
   - Save the private key securely!

2. Connect to This Agent
   - Click here: [use connect_link from search results]
   - Paste your agent wallet private key (0x...)

SECURITY:
   - NEVER use your main wallet private key
   - Agent wallet can trade but CANNOT withdraw
   - You control funds from your main wallet

Let me know when you're done!
```

Wait for user to connect, then proceed to Step 2b.

**Step 2b: If Connected**

Verify account:

```javascript
// Get Hyperliquid mcp_server_id from SPLOX_LIST_USER_CONNECTIONS

// Verify account
SPLOX_EXECUTE_TOOL(
  mcp_server_id: "[hyperliquid_id]",
  slug: "hyperliquid_get_balance",
  args: {}
)

SPLOX_EXECUTE_TOOL(
  mcp_server_id: "[hyperliquid_id]",
  slug: "hyperliquid_get_positions",
  args: {}
)

SPLOX_EXECUTE_TOOL(
  mcp_server_id: "[hyperliquid_id]",
  slug: "hyperliquid_get_open_orders",
  args: {}
)
```

**Step 3: Show Account Status**

```
ACCOUNT STATUS

Agent Wallet: [address]
Balance: $[accountValue]
Positions: [numberOfPositions]
Orders: [numberOfOrders]
Margin Used: $[totalMarginUsed]
Withdrawable: $[withdrawable]
```

**Step 4: Ask for Trading Mode**

```
Which trading mode?

1. Conservative - 1-2x leverage, +20%/year target
2. Balanced - 3-7x leverage, +25-50% target
3. Aggressive - 10-20x leverage, +50-100% target
4. Degen - max leverage, +100-300% target

Choose (1-4):
```

**Step 5: Load Mode Configuration**

Based on user's choice, read the corresponding mode reference file:

- Mode 1: Read `references/mode-conservative.md`
- Mode 2: Read `references/mode-balanced.md`
- Mode 3: Read `references/mode-aggressive.md`
- Mode 4: Read `references/mode-degen.md`

Each mode file contains a step-by-step setup flow. Execute every step in order — each step depends on the previous one. The monitoring and scheduling steps are what make the agent autonomous.

## Trading Modes

| Mode | Leverage | Target | Daily Loss Limit | Scan Interval |
|------|----------|--------|------------------|---------------|
| 1. Conservative | 1-2x | +20%/year | -5% | 3 days |
| 2. Balanced | 3-5x | +25-50% | -8% | 2 hours |
| 3. Aggressive | 5-15x | +50-100% | -10% | 20 min |
| 4. Degen | 15-25x | +100-300% | -15% | 10 min |

Each mode has specific parameters in `references/mode-[name].md`.

## Risk Management

| Rule | Implementation |
|------|----------------|
| Position Size | `BASE_RISK × (confidence/10) × volatility_factor` |
| Stop Loss | 1.5-2× ATR, max 60% of liquidation distance |
| Take Profit | ≥2× stop loss (2:1 R:R minimum, **enforced**) |
| Consecutive Losses | 3 → cooldown **4-6 hours**, reduce size 50% |
| Daily Losses | 5 losses in one day → **stop for 24 hours** |
| Drawdown | Tiered response (see below) |

### Drawdown Circuit Breaker (HARD LIMITS)

| Drawdown from Peak | Action |
|--------------------|--------|
| -10% | ⚠️ Warning: reduce position sizes by 30% |
| -15% | 🔶 Caution: reduce position sizes by 50%, pause new entries for 2 hours |
| -20% | 🛑 **HARD STOP**: Close all positions, halt trading, notify user |

```javascript
// Check on EVERY wake-up, before any other action
async function check_drawdown_circuit_breaker() {
  const balance = await hyperliquid_get_balance({})
  const current = balance.accountValue

  // Track peak balance (store in session)
  PEAK_BALANCE = Math.max(PEAK_BALANCE || STARTING_BALANCE, current)

  const drawdown_pct = ((PEAK_BALANCE - current) / PEAK_BALANCE) * 100

  if (drawdown_pct >= 20) {
    // HARD STOP
    await telegram_send_message({
      chat_id: TELEGRAM_CHAT_ID,
      text: `🛑 *DRAWDOWN LIMIT HIT (-${drawdown_pct.toFixed(1)}%)*
Peak: $${PEAK_BALANCE.toFixed(2)}
Current: $${current.toFixed(2)}
Loss: -$${(PEAK_BALANCE - current).toFixed(2)}

Trading HALTED. Manual review required.
All positions being closed...`
    })
    await cleanup()
    return { halt: true, reason: 'drawdown_limit' }
  }

  if (drawdown_pct >= 15) {
    POSITION_SIZE_MULTIPLIER = 0.5
    PAUSE_NEW_ENTRIES_UNTIL = Date.now() + (2 * 60 * 60 * 1000)
    return { halt: false, warning: 'caution', multiplier: 0.5 }
  }

  if (drawdown_pct >= 10) {
    POSITION_SIZE_MULTIPLIER = 0.7
    return { halt: false, warning: 'warning', multiplier: 0.7 }
  }

  POSITION_SIZE_MULTIPLIER = 1.0
  return { halt: false }
}
```

### Dynamic Stop-Loss Management

**Move stops to protect profits as position goes in your favor:**

| Position P&L | New Stop Level | Result |
|--------------|----------------|--------|
| +5% | Breakeven (-0.3% for fees) | Risk-free position |
| +10% | +5% locked | Guaranteed profit |
| +15% | +10% locked | More profit locked |
| +20%+ | Trail 5% below max | Ride the trend |

```javascript
// Call for each position on every wake-up
async function manage_dynamic_stop(position, mode) {
  const pnl_pct = position.unrealizedPnl / position.marginUsed * 100
  const entry = position.entryPx
  const direction = position.szi > 0 ? 1 : -1  // 1 for long, -1 for short

  let new_stop_pct = null

  // Degen: aggressive trailing
  if (mode === 'degen') {
    if (pnl_pct >= 20) new_stop_pct = pnl_pct - 5      // Trail 5%
    else if (pnl_pct >= 15) new_stop_pct = 10
    else if (pnl_pct >= 10) new_stop_pct = 5
    else if (pnl_pct >= 5) new_stop_pct = -0.3         // Breakeven
  }

  // Aggressive
  if (mode === 'aggressive') {
    if (pnl_pct >= 15) new_stop_pct = pnl_pct - 4      // Trail 4%
    else if (pnl_pct >= 12) new_stop_pct = 8
    else if (pnl_pct >= 8) new_stop_pct = 4
    else if (pnl_pct >= 5) new_stop_pct = -0.3         // Breakeven
  }

  // Balanced
  if (mode === 'balanced') {
    if (pnl_pct >= 10) new_stop_pct = pnl_pct - 3      // Trail 3%
    else if (pnl_pct >= 8) new_stop_pct = 5
    else if (pnl_pct >= 5) new_stop_pct = 2
    else if (pnl_pct >= 3) new_stop_pct = -0.2         // Breakeven
  }

  // Conservative
  if (mode === 'conservative') {
    if (pnl_pct >= 6) new_stop_pct = pnl_pct - 2       // Trail 2%
    else if (pnl_pct >= 4) new_stop_pct = 2
    else if (pnl_pct >= 2.5) new_stop_pct = -0.1       // Breakeven
  }

  if (new_stop_pct !== null) {
    const new_stop_price = entry * (1 + (new_stop_pct / 100) * direction)

    // Only move stop if it's BETTER than current
    const current_stop = await get_current_stop(position.coin)
    const is_better = direction > 0
      ? new_stop_price > current_stop
      : new_stop_price < current_stop

    if (is_better) {
      await move_stop_loss(position.coin, new_stop_price)

      await telegram_send_message({
        chat_id: TELEGRAM_CHAT_ID,
        text: `🔒 *${position.coin}* Stop moved
P&L: +${pnl_pct.toFixed(1)}%
New SL: $${new_stop_price.toFixed(4)} (${new_stop_pct > 0 ? '+' : ''}${new_stop_pct.toFixed(1)}% locked)`
      })
    }
  }
}
```

### Re-entry After Trailing Stop

When closed by trailing stop (not original SL), consider re-entering if trend continues:

```javascript
async function check_reentry_opportunity(closed_position) {
  // Only for trailing stop exits, not losses
  if (closed_position.exit_reason !== 'trailing_stop') return null
  if (closed_position.realized_pnl <= 0) return null

  // Wait 5 minutes for price to settle
  await wait(5 * 60 * 1000)

  const current_price = await hyperliquid_get_price(closed_position.coin)
  const exit_price = closed_position.exit_price
  const was_long = closed_position.direction === 'LONG'

  // Check if price moved further in our direction (trend continuing)
  const price_continued = was_long
    ? current_price > exit_price * 1.015  // +1.5% above exit
    : current_price < exit_price * 0.985  // -1.5% below exit

  if (price_continued) {
    // Quick trend validation
    const research = await quick_trend_check(closed_position.coin)

    if (research.confidence >= 6 && research.direction === closed_position.direction) {
      await telegram_send_message({
        chat_id: TELEGRAM_CHAT_ID,
        text: `🔄 *${closed_position.coin}* Re-entry opportunity
Exited at: $${exit_price.toFixed(4)} (+${closed_position.pnl_pct.toFixed(1)}%)
Current: $${current_price.toFixed(4)}
Trend confirmed, re-entering ${closed_position.direction}...`
      })

      return {
        reentry: true,
        coin: closed_position.coin,
        direction: closed_position.direction
      }
    }
  }

  return { reentry: false }
}
```

## Funding Rate Edge

| Condition | Action |
|-----------|--------|
| Funding > +0.05%/hr | Favor SHORT (longs paying) |
| Funding < -0.05%/hr | Favor LONG (shorts paying) |
| Trade against edge | Confidence -2 |

## Volatility Adaptation

| ATR vs Average | Action |
|----------------|--------|
| > 1.5× (high vol) | Size ×0.6, reduce leverage |
| < 0.7× (low vol) | Size ×1.1, normal leverage |

## Telegram Notifications

**MUST send to Telegram** after every trade action using:
- chat_id: `TELEGRAM_CHAT_ID` (provided by user at start)
- parse_mode: `Markdown`

### Message Templates

**Session Start:**
```
🚀 *{MODE} Mode Started*
Balance: ${BALANCE}
Target: ${TARGET} (+{PCT}%)
Scan: every {SCAN_INTERVAL}
```

**Entry:**
```
🟢 *{DIRECTION} {COIN}* @ ${ENTRY}
Leverage: {LEV}x | Size: ${SIZE}
TP: ${TP} | SL: ${SL}

📝 {REASON}

Next: {SCAN_INTERVAL}
```

**Exit - Win:**
```
✅ *{COIN} +${PNL}* (+{PCT}%)
Entry: ${ENTRY} → Exit: ${EXIT}
Balance: ${BALANCE} ({PROGRESS}% to target)

Next: {SCAN_INTERVAL}
```

**Exit - Loss:**
```
❌ *{COIN} -${PNL}* ({PCT}%)
Entry: ${ENTRY} → Exit: ${EXIT}
Balance: ${BALANCE}

Next: {SCAN_INTERVAL}
```

**Scan - No Trade:**
```
🔍 Scan complete - no setup found
Reason: {WHY_NO_TRADE}
Positions: {POS}/{MAX}
Balance: ${BALANCE}

Next: {SCAN_INTERVAL}
```

**Position Alert:**
```
📊 *{COIN}* {PNL_PCT}%
Current: ${PRICE} | Entry: ${ENTRY}
{ACTION_TAKEN}
```

**Stop Moved (Profit Locked):**
```
🔒 *{COIN}* Stop adjusted
P&L: +{PNL_PCT}%
New SL: ${NEW_SL} ({LOCKED_PCT}% locked)
```

**Breakeven Reached:**
```
🛡️ *{COIN}* Breakeven secured
Position now risk-free
Current P&L: +{PNL_PCT}%
```

**Re-entry After Stop:**
```
🔄 *{COIN}* Re-entry
Exited at: ${EXIT} (+{LOCKED}%)
Price continued: ${CURRENT}
Re-entering {DIRECTION}...
```

**Drawdown Warning:**
```
⚠️ *Drawdown Alert* -{DRAWDOWN_PCT}%
Peak: ${PEAK} → Current: ${CURRENT}
Action: {REDUCED_SIZE / PAUSED / HALTED}
```

**Target Reached:**
```
🎉 *TARGET REACHED!*
${STARTING} → ${FINAL}
Return: +{RETURN}%
Trades: {TOTAL} ({WINS}W/{LOSSES}L)
Duration: {DURATION}
```

**Daily Summary (optional):**
```
📈 *Daily Report*
Balance: ${BALANCE} ({DAY_CHANGE}%)
Open: {POSITIONS} positions
Today: {TRADES} trades ({WINS}W/{LOSSES}L)
Progress: {PROGRESS}% to target
```

**{SCAN_INTERVAL}** values:
- Degen: `10min`
- Aggressive: `20min`
- Balanced: `2hr`
- Conservative: `3d`

### What to Report

| Event | Report |
|-------|--------|
| Session start | Mode, balance, target |
| New trade | Coin, direction, leverage, TP/SL, **reason** |
| Trade closed | Result, P&L, new balance |
| Scan (no trade) | Why skipped (unclear trend, low confidence, etc.) |
| Position alert | Current P&L, any action taken |
| Target reached | Full summary with stats |
| Error/Issue | What happened, what agent will do |

### Reason Examples

Good reasons to include:
- "BTC breaking $100k resistance, momentum confirmed"
- "SOL oversold, funding negative, expecting bounce"
- "ETH weak vs BTC, shorting the ratio play"
- "DOGE memecoin pump, riding momentum"

Why no trade:
- "BTC choppy, no clear direction"
- "All setups below 6 confidence"
- "Funding extreme, waiting for reset"
- "Portfolio full (3/3 positions)"

## Agent Autonomy

You have freedom to decide:
- **Allocation**: Concentrate or diversify across positions
- **Sizing**: Adjust within risk limits based on conviction
- **Coin selection**: Override if leverage/liquidity unsuitable
- **Timing**: Enter now or wait for better price
- **Exit strategy**: Adjust TP/SL based on market conditions

Guidelines are guardrails, not handcuffs. Use your research and judgment.

**Hard limits (non-negotiable):**
- Daily loss limit → hard stop
- 3 consecutive losses → cooldown **4-6 hours** (not minutes!)
- 5 losses in one day → **stop for 24 hours**
- Never risk more than mode's base risk per trade
- **-20% drawdown from peak → HALT ALL TRADING**
- **Minimum R:R = 2:1** (TP must be ≥2× SL distance)

## Important Notes

- Check connection before any action
- Load mode parameters as guidelines
- Execute all steps from the mode file in order, none are optional
- Respect daily drawdown limits
- Always verify coin's max leverage before researching it deeply