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

### On ANY User Message

**Step 1: Check if Hyperliquid is connected**

```javascript
SPLOX_SEARCH_TOOLS(query: "hyperliquid")
// Check is_user_connected field
```

**Step 2a: If NOT Connected**

Show connection instructions:

```
Let's set up your autonomous trading agent!

To trade on Hyperliquid, you need to connect an agent wallet.

SETUP STEPS:

1. Create an Agent Wallet (if you don't have one)
   - Go to: https://app.hyperliquid.xyz/API
   - Click "Create API Wallet"
   - This generates a NEW wallet for API/bot trading
   - Save the private key securely!

2. Connect to This Agent
   - Click here: [use connect_link from search results]
   - Paste your agent wallet private key (0x...)
   - Your key is encrypted and stored securely

SECURITY:
   - NEVER use your main wallet private key
   - Agent wallet can trade using your main wallet's funds
   - Agent wallet CANNOT withdraw (only trade)
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

Display the account information using this exact format with line breaks:

```
Your Hyperliquid agent wallet is connected!

ACCOUNT STATUS

Agent Wallet:
[address]

Balance:
$[accountValue]

Positions:
[numberOfPositions]

Orders:
[numberOfOrders]

Margin Used:
$[totalMarginUsed]

Withdrawable:
$[withdrawable]
```

**Step 4: Ask for Trading Mode**

After showing account status, ask user to select a trading mode:

```
Which trading mode?

1. Conservative - 1-2x leverage, +20%/year target
2. Balanced - 3-7x leverage, +25-50% target
3. Aggressive - 15-25x leverage, +50-100% target
4. Degen - 25-50x leverage, +100-300% target

Choose (1-4):
```

**Step 5: Load Mode Configuration**

Based on user's choice, read the corresponding mode reference file:

- Mode 1: Read `references/mode-conservative.md`
- Mode 2: Read `references/mode-balanced.md`
- Mode 3: Read `references/mode-aggressive.md`
- Mode 4: Read `references/mode-degen.md`

Use the parameters from the mode file for all trading decisions.

## Trading Modes

| Mode | Leverage | Target | Daily Loss Limit | Scan Interval |
|------|----------|--------|------------------|---------------|
| 1. Conservative | 1-2x | +20%/year | -5% | 3 days |
| 2. Balanced | 3-7x | +25-50% | -8% | 2 hours |
| 3. Aggressive | 15-25x | +50-100% | -12% | 20 min |
| 4. Degen | 25-50x | +100-300% | -20% | 10 min |

Each mode has specific parameters in `references/mode-[name].md`.

## Risk Management

| Rule | Implementation |
|------|----------------|
| Position Size | Half-Kelly: `BASE_RISK × (confidence/10) × volatility_factor` |
| Stop Loss | 1.5-2× ATR, max 60% of liquidation distance |
| Take Profit | ≥2× stop loss (2:1 R:R minimum) |
| Consecutive Losses | 3 → cooldown 30-45min, reduce size 50% |
| Drawdown | Tiered response → reduce size → hard stop |

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
- chat_id: `305544740` (FIXED, not session ID)
- parse_mode: `Markdown`

### Message Format (send to Telegram)

```
// Entry - ALWAYS include Next scan time
🟢 LONG {COIN} @ ${ENTRY} | {LEV}x | Next: {SCAN_INTERVAL}

// Exit - Win
✅ {COIN} +${PNL} (+{PCT}%) | Next: {SCAN_INTERVAL}

// Exit - Loss
❌ {COIN} -${PNL} | Next: {SCAN_INTERVAL}

// Scan - No trade
🔍 No setup | Next: {SCAN_INTERVAL}

// Target hit
🎉 TARGET! +{RETURN}%
```

**{SCAN_INTERVAL}** values:
- Degen: `10min`
- Aggressive: `20min`
- Balanced: `2hr`
- Conservative: `3d`

### Output Rule
- **Chat**: Full details + thesis
- **Telegram**: Brief summary with Next scan time (REQUIRED)

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
- 3 consecutive losses → cooldown
- Never risk more than mode's base risk per trade

## Important Notes

- Check connection before any action
- Load mode parameters as guidelines
- Respect daily drawdown limits
- **Send Telegram notification after every trade**