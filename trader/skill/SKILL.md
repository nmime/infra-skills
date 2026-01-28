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

## Mode Configuration Reference

```javascript
const MODE_CONFIG = {
  degen: {
    leverage_min: 15, leverage_max: 25,
    position_pct: 0.25,     // 25% of account
    sl_pct: 10,             // -10%
    tp_pct: 20,             // +20%
    confidence_min: 5,
    scan_interval: 600,     // 10 min
    slippage: 0.8, max_spread: 0.5
  },
  aggressive: {
    leverage_min: 5, leverage_max: 15,
    position_pct: 0.15,     // 15%
    sl_pct: 6,              // -6%
    tp_pct: 12,             // +12%
    confidence_min: 6,
    scan_interval: 1200,    // 20 min
    slippage: 0.5, max_spread: 0.3
  },
  balanced: {
    leverage_min: 3, leverage_max: 5,
    position_pct: 0.10,     // 10%
    sl_pct: 4,              // -4%
    tp_pct: 8,              // +8%
    confidence_min: 7,
    scan_interval: 7200,    // 2 hours
    slippage: 0.3, max_spread: 0.2
  },
  conservative: {
    leverage_min: 1, leverage_max: 2,
    position_pct: 0.06,     // 6%
    sl_pct: 2.5,            // -2.5%
    tp_pct: 5,              // +5%
    confidence_min: 8,
    scan_interval: 259200,  // 3 days
    slippage: 0.2, max_spread: 0.15
  }
}
```

## Risk Management

| Rule | Implementation |
|------|----------------|
| Position Size | `account × position_pct` |
| Stop Loss | Use `sl_pct` from MODE_CONFIG |
| Take Profit | Use `tp_pct` from MODE_CONFIG (always ≥2× SL) |
| Daily Loss Limit | Mode-specific (see mode files) |

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

## Slippage Protection

**Always use limit orders with slippage tolerance instead of pure market orders.**

```javascript
// Max slippage by mode
const SLIPPAGE_TOLERANCE = {
  degen: 0.8,       // 0.8% - fast execution priority
  aggressive: 0.5,  // 0.5%
  balanced: 0.3,    // 0.3%
  conservative: 0.2 // 0.2% - price priority
}

async function place_protected_order(coin, is_buy, size, mode) {
  const current_price = await hyperliquid_get_price(coin)
  const slippage = SLIPPAGE_TOLERANCE[mode] / 100

  // Limit price with slippage buffer
  const limit_price = is_buy
    ? current_price * (1 + slippage)
    : current_price * (1 - slippage)

  return await hyperliquid_place_order({
    coin: coin,
    is_buy: is_buy,
    size: size,
    order_type: "limit",
    price: limit_price,
    time_in_force: "IOC"  // Immediate or Cancel - fill what you can, cancel rest
  })
}
```

**If IOC order only partially fills:**
- Accept partial fill if > 70% filled
- Cancel and retry with wider slippage if < 70% filled
- Report to Telegram what happened

## Liquidity & Spread Check

**Before entering ANY position, validate liquidity:**

```javascript
async function check_liquidity(coin, position_size_usd, mode) {
  const orderbook = await hyperliquid_get_orderbook({ coin: coin })

  // Calculate spread
  const best_bid = orderbook.bids[0][0]
  const best_ask = orderbook.asks[0][0]
  const spread_pct = ((best_ask - best_bid) / best_bid) * 100

  // Max acceptable spread by mode
  const MAX_SPREAD = {
    degen: 0.5,       // 0.5% max
    aggressive: 0.3,  // 0.3%
    balanced: 0.2,    // 0.2%
    conservative: 0.15 // 0.15%
  }

  if (spread_pct > MAX_SPREAD[mode]) {
    await telegram_send_message({
      chat_id: TELEGRAM_CHAT_ID,
      text: `⚠️ *${coin}* spread too wide: ${spread_pct.toFixed(2)}%
Max allowed: ${MAX_SPREAD[mode]}%
Skipping trade.`
    })
    return { ok: false, reason: 'spread_too_wide' }
  }

  // Check depth - need 10x our size in top 5 levels
  const bid_depth = orderbook.bids.slice(0, 5).reduce((sum, [price, size]) => sum + price * size, 0)
  const ask_depth = orderbook.asks.slice(0, 5).reduce((sum, [price, size]) => sum + price * size, 0)
  const min_depth = Math.min(bid_depth, ask_depth)

  if (min_depth < position_size_usd * 10) {
    await telegram_send_message({
      chat_id: TELEGRAM_CHAT_ID,
      text: `⚠️ *${coin}* low liquidity
Depth: $${min_depth.toFixed(0)} | Need: $${(position_size_usd * 10).toFixed(0)}
Skipping trade.`
    })
    return { ok: false, reason: 'low_liquidity' }
  }

  return { ok: true, spread: spread_pct, depth: min_depth }
}
```

## Time-Based Filters

**Reduce exposure during low-liquidity periods:**

```javascript
function check_trading_conditions() {
  const now = new Date()
  const hour_utc = now.getUTCHours()
  const day = now.getUTCDay()  // 0 = Sunday

  const conditions = {
    is_weekend: (day === 0 || day === 6),
    is_asia_night: (hour_utc >= 21 || hour_utc <= 5),  // Low liquidity
    is_us_open: (hour_utc >= 13 && hour_utc <= 21),    // High liquidity
    multiplier: 1.0,
    warning: null
  }

  // Weekend: reduce size, wider stops
  if (conditions.is_weekend) {
    conditions.multiplier = 0.7
    conditions.warning = "Weekend trading - reduced size"
  }

  // Asia night (low liquidity): reduce size
  if (conditions.is_asia_night && !conditions.is_weekend) {
    conditions.multiplier = 0.8
    conditions.warning = "Low liquidity hours - reduced size"
  }

  return conditions
}

// Apply in position sizing
const time_conditions = check_trading_conditions()
POSITION_SIZE *= time_conditions.multiplier

if (time_conditions.warning) {
  await telegram_send_message({
    chat_id: TELEGRAM_CHAT_ID,
    text: `⏰ ${time_conditions.warning} (×${time_conditions.multiplier})`
  })
}
```

## BTC Alignment Check (All Modes)

**For altcoins: check BTC trend before entry.**

```javascript
async function check_btc_alignment(coin, direction) {
  // Skip for BTC itself
  if (coin === 'BTC') return { aligned: true, btc_trend: 'N/A' }

  // Get BTC trend (simple: 4h price change)
  const btc_prices = await hyperliquid_get_candles({ coin: 'BTC', interval: '4h', limit: 2 })
  const btc_change = (btc_prices[1].close - btc_prices[0].close) / btc_prices[0].close * 100

  let btc_trend = 'NEUTRAL'
  if (btc_change > 1) btc_trend = 'UP'
  if (btc_change < -1) btc_trend = 'DOWN'

  // Check alignment
  const aligned = (
    (direction === 'LONG' && btc_trend !== 'DOWN') ||
    (direction === 'SHORT' && btc_trend !== 'UP')
  )

  const result = {
    aligned: aligned,
    btc_trend: btc_trend,
    btc_change: btc_change,
    confidence_penalty: aligned ? 0 : -2
  }

  if (!aligned) {
    await telegram_send_message({
      chat_id: TELEGRAM_CHAT_ID,
      text: `⚠️ *BTC misalignment*
BTC trend: ${btc_trend} (${btc_change > 0 ? '+' : ''}${btc_change.toFixed(1)}%)
Trade: ${direction} ${coin}
Confidence -2 applied`
    })
  }

  return result
}
```

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

**Liquidity/Spread Skip:**
```
⚠️ *{COIN}* {REASON}
{DETAILS}
Skipping trade.
```

**BTC Misalignment:**
```
⚠️ *BTC misalignment*
BTC trend: {TREND} ({CHANGE}%)
Trade: {DIRECTION} {COIN}
Confidence -2 applied
```

**Time Condition:**
```
⏰ {WARNING}
Size multiplier: ×{MULT}
```

**Target Reached:**
```
🎉 *TARGET REACHED!*
${STARTING} → ${FINAL}
Return: +{RETURN}%
Duration: {DURATION}
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
| Target reached | Final balance, return percentage |
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
- Daily loss limit → stop opening new positions
- Never risk more than mode's base risk per trade
- **Minimum R:R = 2:1** (TP must be ≥2× SL distance)
- Always use stop-loss orders (never trade without protection)

## Important Notes

- Check connection before any action
- Load mode parameters as guidelines
- Execute all steps from the mode file in order, none are optional
- Respect daily drawdown limits
- Always verify coin's max leverage before researching it deeply