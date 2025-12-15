# COTI Blockchain Transaction Notifier

A lightweight n8n workflow that monitors COTI blockchain transactions and sends real-time notifications to Telegram.

## Overview

This guide walks you through building an automated notification system that:

- Polls the COTI blockchain for new transactions
- Detects incoming and outgoing transfers to your wallet
- Sends formatted alerts to your Telegram account
- Prevents duplicate notifications using transaction tracking

## Prerequisites

Before starting, ensure you have:

1. **n8n Account** - Sign up at [n8n.io](https://n8n.io) or self-host
2. **Telegram Account** - With a bot created via [@BotFather](https://t.me/BotFather)
3. **Telegram Bot Token** - Obtained from BotFather during bot creation
4. **Telegram Bot @userinfobot (optional)** - Obtaining Chat ID
## Architecture

```
Schedule Trigger --> HTTP Request --> Code --> IF --> Telegram
     (5 min)         (COTI API)     (Logic)  (Filter)  (Alert)
```

## Manual Setup Instructions

### Step 1: Create Schedule Trigger Node

This node initiates the workflow at regular intervals.

| Setting | Value |
|---------|-------|
| Trigger Interval | Every 5 Minutes |

### Step 2: Create HTTP Request Node

Connect this node to the Schedule Trigger. It fetches transaction data from COTI.

| Setting | Value |
|---------|-------|
| Method | GET |
| URL | `https://mainnet.cotiscan.io/api/v2/addresses/{YOUR_WALLET}/transactions` |

Replace `{YOUR_WALLET}` with your COTI wallet address.

Step 3: Create Code Node
Connect this node to the HTTP Request. It handles transaction tracking and message formatting.
The Code node performs three functions:

Compares incoming transactions against previously seen transactions
Identifies new transactions involving your wallet
Formats transaction data for Telegram output

Use the following prompt to generate the JavaScript code:
```
```
Write n8n Code node JavaScript for tracking COTI blockchain transactions.

Input structure from HTTP Request node:
[{
    "items": [{
        "hash": "0x...",
        "value": "2000000000000000000",
        "from": { "hash": "0x..." },
        "to": { "hash": "0x..." },
        "status": "ok",
        "block_number": 4628668,
        "timestamp": "2025-12-15T17:12:36.000000Z",
        "fee": { "value": "42000000147000" }
    }]
}]

Wallet to monitor: YOUR_WALLET_ADDRESS

Required behavior:
1. Get response using: const response = $input.first().json;
2. Get static data using: const staticData = $getWorkflowStaticData('global');
3. Check if first run: if seenTxHashes does not exist, initialize it, store ALL current transaction hashes, then return [{ json: { hasNew: false } }] immediately (silent initialization)
4. For subsequent runs: find transactions not in seenTxHashes that involve the wallet
5. Update seenTxHashes with new hashes (limit to last 500)
6. If no new transactions: return [{ json: { hasNew: false } }]
7. For each new transaction, return object with:
   - hasNew: true
   - message: formatted Telegram message
   - txHash: transaction hash
   - isIncoming: boolean
   - amount: number in COTI
   - status: transaction status

Message format (use template literal with backticks):
- Line 1: Direction emoji and text (incoming or outgoing) + status emoji
- Line 2: Amount in COTI (convert from Wei by dividing by 1e18)
- Line 3: From address (first 8 chars + ... + last 6 chars)
- Line 4: To address (truncated same way)
- Line 5: Block number
- Line 6: Fee in COTI
- Line 7: Timestamp formatted
- Line 8: Explorer link https://explorer.coti.io/tx/{hash}

CRITICAL n8n Code node v2 syntax:
- Static data: $getWorkflowStaticData('global') - NOT this.getWorkflowStaticData()
- Input access: $input.first().json - NOT $json
- Return format: MUST wrap in json property - return [{ json: { hasNew: false } }]
- Multiple returns: return array.map(item => ({ json: { ... } }))

Code requirements:
- Use const/let not var
- Use Number() for value conversion: Number(tx.value) / 1e18
- Access nested properties safely: tx.from?.hash
- Provide complete working JavaScript code only
```
Replace `YOUR_WALLET_ADDRESS` with your actual COTI wallet address.


### Step 4: Create IF Node

Connect this node to the Code node. It filters out empty polling results.

| Setting | Value |
|---------|-------|
| Condition | `{{ $json.hasNew }}` |
| Operation | equals |
| Value | true |

### Step 5: Create Telegram Node

Connect this node to the IF node (true branch). It sends the notification.

| Setting | Value |
|---------|-------|
| Credential | Your Telegram Bot Token |
| Operation | Send Message |
| Chat ID | Your Telegram Chat ID |
| Text | `{{ $json.message }}` |
| Parse Mode | MarkdownV2 |

To obtain your Chat ID, message [@userinfobot](https://t.me/userinfobot) on Telegram.

## AI Prompt for Workflow Generation

Use the following prompt with Claude or your preferred AI assistant to generate the Code node logic:

```
You are an expert n8n workflow developer. Create a complete COTI blockchain transaction notification workflow for Telegram.

Workflow requirements:
1. Schedule Trigger - Poll every 5 minutes
2. HTTP Request - GET https://mainnet.cotiscan.io/api/v2/addresses/YOUR_WALLET_ADDRESS/transactions
3. Code Node - JavaScript that:
   - Tracks seen transactions using workflow static data (no duplicates)
   - Silent first run: on initialization, store all current transaction hashes and return hasNew: false (no notifications)
   - After first run: detect new incoming/outgoing transactions for the wallet
   - Handles multiple new transactions per poll
   - Formats message with: direction indicator, amount in COTI, truncated from/to addresses, block number, fee, timestamp, CotiScan explorer link
   - Returns hasNew flag for filtering
4. IF Node - Continue only when hasNew equals true
5. Telegram Node - Send formatted message using bot credentials

API response format:
[{
    "items": [{
        "hash": "0x...",
        "value": "2000000000000000000",
        "from": { "hash": "0x..." },
        "to": { "hash": "0x..." },
        "status": "ok",
        "block_number": 4628668,
        "timestamp": "2025-12-15T17:12:36.000000Z",
        "fee": { "value": "42000000147000" }
    }]
}]

Wallet: YOUR_WALLET_ADDRESS

Provide:
1. Complete n8n workflow JSON that can be imported directly (use n8n export format with nodes array and connections object)
2. Instructions for adding Telegram credentials after import

Critical n8n Code node v2 syntax requirements:
- Get static data: $getWorkflowStaticData('global') - NOT this.getWorkflowStaticData()
- Get input: $input.first().json - NOT $json or items[0].json (JavaScript Node)
- Return format: return [{ json: { key: value } }] - objects MUST be wrapped in { json: { } }
- Multiple items: return array.map(item => ({ json: { ... } }))

Important n8n JSON structure requirements:
- Root object must contain: name, nodes (array), connections (object), settings
- Each node must have: id, name, type, position (array [x, y]), parameters, typeVersion
- Node type values: n8n-nodes-base.scheduleTrigger, n8n-nodes-base.httpRequest, n8n-nodes-base.code, n8n-nodes-base.if, n8n-nodes-base.telegram
- Connections format: { "Node Name": { "main": [[{ "node": "Next Node Name", "type": "main", "index": 0 }]] } }
- IF node has two outputs: connections use index 0 for true branch, index 1 for false branch
- Code node: use jsCode parameter, typeVersion 2
- Telegram node: use chatId parameter, text parameter with expression {{ $json.message }}
```

Replace `YOUR_WALLET_ADDRESS` with your actual COTI wallet address.

## Sample Output

When a new transaction is detected, you will receive a Telegram message in this format:

```
INCOMING TRANSACTION

Amount: 2.0000 COTI
From: 0xAcfE00...CBD88
To: 0xcFAEf2...0B497
Block: 4628668
Fee: 0.000042 COTI
Time: Dec 15, 2025, 5:12 PM

View on CotiScan: https://explorer.coti.io/tx/0x...
```

## Configuration Notes

- **Polling Interval**: Default is 5 minutes. Adjust based on your needs. (30 seconds is safe zone, lower than this could prevent you from spamming)
- **Transaction History**: The workflow tracks the last 500 transactions to prevent memory issues.
- **First Run**: On initial execution, all transactions in the API response will trigger notifications.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| No notifications received | Verify bot token and chat ID are correct |
| Duplicate notifications | Clear workflow static data and restart |
| Parse errors in Telegram | Ensure Parse Mode is set to MarkdownV2 |
| Empty API response | Confirm wallet address is valid and has transactions |

## Resources

- [COTI Network](https://coti.io)
- [CotiScan Explorer](https://mainnet.cotiscan.io)
- [n8n Documentation](https://docs.n8n.io)
- [Telegram Bot API](https://core.telegram.org/bots/api)

## License

MIT License - Feel free to modify and distribute.
