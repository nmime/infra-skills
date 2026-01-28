---
name: trading-agent
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
Which trading mode do you want to run?

1. Conservative - Low risk, protect capital (1-2x leverage, 5% position size)
2. Balanced - Moderate risk, steady growth (2-5x leverage, 10% position size)
3. Aggressive - High risk, high reward (5-10x leverage, 50% position size)
4. Degen - Maximum risk, moon or bust (10-50x leverage, 80% position size)

Choose a number (1-4):
```

**Step 5: Load Mode Configuration**

Based on user's choice, read the corresponding mode reference file:

- Mode 1: Read `references/mode-conservative.md`
- Mode 2: Read `references/mode-balanced.md`
- Mode 3: Read `references/mode-aggressive.md`
- Mode 4: Read `references/mode-degen.md`

Use the parameters from the mode file for all trading decisions.

## Trading Modes

Four modes with professional risk management:

1. **Conservative** - Capital preservation, 1-2x leverage, 20% annual target
2. **Balanced** - Trend following, 3-7x leverage, 25-50% target
3. **Aggressive** - Momentum + MTF confirmation, 8-12x leverage, 50-100% target
4. **Degen** - High-frequency momentum, 15-25x leverage, 100-300% target

### Professional Features (All Modes)

- **Half-Kelly Position Sizing** - Size based on confidence and edge
- **ATR-Based Dynamic Stops** - Volatility-adjusted SL/TP (not fixed %)
- **Minimum 2:1 Risk/Reward** - Every trade has positive expectancy
- **Tiered Drawdown Protection** - Reduce risk as losses mount
- **Consecutive Loss Cooldown** - Prevent revenge trading
- **Funding Rate Edge** - Fade extreme funding for contrarian plays
- **Volatility Regime Detection** - Adapt to market conditions

Each mode has specific parameters in `references/mode-[name].md`.

## Important Notes

- Always check connection first before any action
- Verify account balance before trading
- Load and follow the selected mode's parameters strictly
- Never exceed calculated risk per trade
- ATR stops adapt to volatility automatically
- Track win rate for Kelly sizing optimization
- Respect daily drawdown limits - live to trade another day