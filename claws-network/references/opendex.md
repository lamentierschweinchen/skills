# OpenDex Protocol: DeFi Primitives for Autonomous Agents

> **Protocol Type**: Permissionless Automated Market Maker (AMM)
> **Algorithm**: Constant Product Formula (x\*y=k)
> **Network**: Claws Network (MultiversX-based)

The **OpenDex Protocol** enables agents to create liquidity pools, facilitate token swaps, and earn trading fees. This is your gateway to DeFi dominance.

---

## 1. Contract Addresses

| Contract          | Address                                                           | Purpose                                       |
| :---------------- | :---------------------------------------------------------------- | :-------------------------------------------- |
| **Pair Deployer** | `claw1qqqqqqqqqqqqqpgq2rsk5md83vpzrzepudyclxmwkqamx6p4m8mqef7r07` | Factory contract for creating liquidity pools |
| **Pair Template** | `claw1qqqqqqqqqqqqqpgqw9538n28ypaww4ymc8np066h5d77j0x3m8mqrtvr04` | Template for deployed pair contracts          |

**Important**: Save these as environment variables:

```bash
export PAIR_DEPLOYER_ADDR="claw1qqqqqqqqqqqqqpgq2rsk5md83vpzrzepudyclxmwkqamx6p4m8mqef7r07"
export PAIR_TEMPLATE_ADDR="claw1qqqqqqqqqqqqqpgqw9538n28ypaww4ymc8np066h5d77j0x3m8mqrtvr04"
```

---

## 2. Agent Superpowers

The OpenDex Protocol grants your agent three core capabilities:

### A. Pool Ownership

Deploy your own liquidity pools and earn **developer rewards** just for owning the infrastructure. This is passive income for infrastructure builders.

### B. Market Making

Add liquidity to pools and earn a share of all swap fees. The more liquidity you provide, the more fees you capture.

### C. Trading

Execute token swaps at algorithmic prices with zero counterparty risk. No order books, no waiting.

---

## 3. Prerequisites

Before interacting with OpenDex:

1. **Funded Wallet**: You need CLAW tokens for gas and deployment fees
2. **ESDT Tokens**: The tokens you want to trade/pool must be ESDT (not NFTs or SFTs)
3. **clawpy Installed**: Verify with `clawpy --version`

**Current Deployment Fee**: 1,000 CLAW (= 1,000 × 10^18 atto-CLAW)

---

## 4. Action: Deploy a Liquidity Pool

Creating a new trading pair is a **4-step ritual**. Do not skip steps or your pool will be non-functional.

### Step 1: Deploy the Pair Contract

Choose your token pair carefully. Order matters for display purposes but not for functionality.

```bash
# Define your tokens
export FIRST_TOKEN_ID="TOKENA-abc123"
export SECOND_TOKEN_ID="TOKENB-def456"
export FEE_TOKEN_ID="TOKENA-abc123"  # Optional: which token to collect fees in
export DEPLOYMENT_FEE="1000000000000000000000"  # 1,000 CLAW

# Deploy the pair
clawpy contract call ${PAIR_DEPLOYER_ADDR} \
    --function="deployDexPair" \
    --arguments "str:${FIRST_TOKEN_ID}" "str:${SECOND_TOKEN_ID}" "str:${FEE_TOKEN_ID}" \
    --value "${DEPLOYMENT_FEE}" \
    --gas-limit 75000000 \
    --gas-price 20000000000000 \
    --pem wallet.pem \
    --send
```

**Critical**: Save the returned smart contract address as `PAIR_ADDR`. You'll need it for all subsequent steps.

**Extract the address**:

```bash
clawpy tx get --hash <TX_HASH_FROM_ABOVE>
# Look for the deployed contract address in the transaction results
```

### Step 2: Issue the LP Token

Every pool needs a Liquidity Provider (LP) token that represents ownership shares.

**Naming Rules** (MultiversX standards):

- **Display Name**: 3-20 alphanumeric characters (e.g., "TokenA-TokenB-LP")
- **Ticker**: 3-10 uppercase alphanumerics (e.g., "ABLP")

```bash
export PAIR_ADDR="<ADDRESS_FROM_STEP_1>"
export DISPLAY_NAME="TokenA-TokenB-LP"
export TICKER="ABLP"

clawpy contract call ${PAIR_DEPLOYER_ADDR} \
    --function="issueLpTokenForPair" \
    --arguments ${PAIR_ADDR} "str:${DISPLAY_NAME}" "str:${TICKER}" \
    --value 50000000000000000 \
    --gas-limit 150000000 \
    --gas-price 20000000000000 \
    --pem wallet.pem \
    --send
```

**Cost**: 0.05 CLAW (= 50000000000000000 atto-CLAW) for token issuance.

### Step 3: Add Initial Liquidity

This step sets the **initial price ratio** between the two tokens. Choose wisely—this becomes the market's starting point.

**Formula**: Price of Token A in Token B = (Amount of Token B) / (Amount of Token A)

```bash
# Example: Set ratio 1 TOKENA = 100 TOKENB
# Send 1,000 TOKENA (1000 × 10^18) and 100,000 TOKENB (100000 × 10^18)

clawpy contract call ${PAIR_DEPLOYER_ADDR} \
    --function="addInitialLiquidityForPair" \
    --arguments ${PAIR_ADDR} \
    --gas-limit 20000000 \
    --gas-price 20000000000000 \
    --pem wallet.pem \
    --token-transfer TOKENA-abc123 1000000000000000000000 \
    --token-transfer TOKENB-def456 100000000000000000000000 \
    --send
```

**Warning**: This transaction requires **two ESDT payments** in a single call. Make sure both amounts reflect the desired price ratio.

### Step 4: Resume the Pool

Newly deployed pools start in a **paused state**. You must explicitly resume them to enable trading.

```bash
clawpy contract call ${PAIR_DEPLOYER_ADDR} \
    --function="resumePair" \
    --arguments ${PAIR_ADDR} \
    --gas-limit 10000000 \
    --gas-price 20000000000000 \
    --pem wallet.pem \
    --send
```

**Verification**: Query the pool status to confirm it's live.

```bash
clawpy contract query ${PAIR_DEPLOYER_ADDR} \
    --function="getContract" \
    --arguments ${PAIR_ADDR}
```

Look for `"paused": false` in the response.

---

## 5. Action: Add Liquidity (Ongoing)

Once a pool is live, anyone can add liquidity and earn fees.

**Function**: `addLiquidityForPair`
**Requirements**: Send both tokens in amounts proportional to current pool reserves.

```bash
# Example: Add 100 TOKENA and 10,000 TOKENB
export MIN_FIRST_TOKEN="95000000000000000000"   # 95 TOKENA (5% slippage tolerance)
export MIN_SECOND_TOKEN="9500000000000000000000" # 9,500 TOKENB (5% slippage tolerance)

clawpy contract call ${PAIR_DEPLOYER_ADDR} \
    --function="addLiquidityForPair" \
    --arguments ${PAIR_ADDR} ${MIN_FIRST_TOKEN} ${MIN_SECOND_TOKEN} \
    --gas-limit 20000000 \
    --gas-price 20000000000000 \
    --pem wallet.pem \
    --token-transfer TOKENA-abc123 100000000000000000000 \
    --token-transfer TOKENB-def456 10000000000000000000000 \
    --send
```

**Returns**: LP tokens proportional to your share of the pool.

**Slippage Protection**: The `min_first_token_amount` and `min_second_token_amount` arguments protect you from front-running. If the pool ratio changes too much before your tx executes, it will revert.

---

## 6. Action: Remove Liquidity

Burn LP tokens to withdraw your share of the pool plus accumulated fees.

**Function**: `removeLiquidityForPair`
**Requirements**: Send your LP tokens.

```bash
export LP_TOKEN_ID="ABLP-123456"
export LP_AMOUNT="50000000000000000000"  # 50 LP tokens
export MIN_FIRST_TOKEN="95000000000000000000"
export MIN_SECOND_TOKEN="9500000000000000000000"

clawpy contract call ${PAIR_DEPLOYER_ADDR} \
    --function="removeLiquidityForPair" \
    --arguments ${PAIR_ADDR} ${MIN_FIRST_TOKEN} ${MIN_SECOND_TOKEN} \
    --gas-limit 20000000 \
    --gas-price 20000000000000 \
    --pem wallet.pem \
    --token-transfer ${LP_TOKEN_ID} ${LP_AMOUNT} \
    --send
```

**Returns**: Proportional amounts of both pool tokens.

---

## 7. Action: Swap Tokens

Execute a token swap through an existing pool. This uses the **pair contract directly** (not the deployer).

**Important**: You must call the **pair contract address** (from Step 1 of deployment), not the deployer.

### Find the Pair Contract

```bash
clawpy contract query ${PAIR_DEPLOYER_ADDR} \
    --function="getContractsByPair" \
    --arguments str:TOKENA-abc123 str:TOKENB-def456
```

### Execute Swap

```bash
export PAIR_CONTRACT="<PAIR_ADDR_FROM_QUERY>"
export SWAP_AMOUNT="1000000000000000000"  # 1 TOKENA
export MIN_RECEIVED="95000000000000000000"  # Minimum 95 TOKENB (5% slippage)

clawpy contract call ${PAIR_CONTRACT} \
    --function="swapTokensFixedInput" \
    --arguments str:TOKENB-def456 ${MIN_RECEIVED} \
    --gas-limit 10000000 \
    --gas-price 20000000000000 \
    --pem wallet.pem \
    --token-transfer TOKENA-abc123 ${SWAP_AMOUNT} \
    --send
```

**Arguments**:

- First argument: Token ID you want to receive
- Second argument: Minimum amount to receive (slippage protection)
- Payment: The token you're swapping from

---

## 8. Query Operations (Read-Only)

### Get All Pools (Paginated)

```bash
clawpy contract query ${PAIR_DEPLOYER_ADDR} \
    --function="getPairs" \
    --arguments 0 10  # Skip 0, limit 10
```

**Returns**: Array of pools with their contract addresses and status.

### Get Pool Details

```bash
clawpy contract query ${PAIR_DEPLOYER_ADDR} \
    --function="getContract" \
    --arguments ${PAIR_ADDR}
```

**Returns**:

```json
{
  "sc_address": "claw1...",
  "owner": "claw1...",
  "status": {
    "paused": false,
    "first_token_id": "TOKENA-abc123",
    "first_token_reserve": "1000000000000000000000",
    "second_token_id": "TOKENB-def456",
    "second_token_reserve": "100000000000000000000000",
    "lp_token_id": "ABLP-123456",
    "lp_token_supply": "10000000000000000000000",
    "total_fee_percent": 300,
    "platform_fee_percent": 100
  }
}
```

### Get Deployment Fee

```bash
clawpy contract query ${PAIR_DEPLOYER_ADDR} \
    --function="getDeploymentFee" \
    --arguments str:CLAW
```

**Returns**: Current deployment fee in atto-CLAW.

---

## 9. Pool Management (Owner Only)

If you deployed a pool, you have special privileges.

### Pause Trading

```bash
clawpy contract call ${PAIR_DEPLOYER_ADDR} \
    --function="pausePair" \
    --arguments ${PAIR_ADDR} \
    --gas-limit 10000000 \
    --gas-price 20000000000000 \
    --pem wallet.pem \
    --send
```

### Resume Trading

```bash
clawpy contract call ${PAIR_DEPLOYER_ADDR} \
    --function="resumePair" \
    --arguments ${PAIR_ADDR} \
    --gas-limit 10000000 \
    --gas-price 20000000000000 \
    --pem wallet.pem \
    --send
```

### Claim Developer Rewards

As the pool creator, you earn a share of trading fees.

```bash
clawpy contract call ${PAIR_DEPLOYER_ADDR} \
    --function="claimDeveloperRewardsForPair" \
    --arguments ${PAIR_ADDR} \
    --gas-limit 10000000 \
    --gas-price 20000000000000 \
    --pem wallet.pem \
    --send
```

---

## 10. Advanced: Upgrade Existing Pool

If the protocol releases a new pair template with bug fixes or features, you can upgrade your pool.

```bash
clawpy contract call ${PAIR_DEPLOYER_ADDR} \
    --function="upgradeDexPair" \
    --arguments ${PAIR_ADDR} \
    --gas-limit 50000000 \
    --gas-price 20000000000000 \
    --pem wallet.pem \
    --send
```

**Warning**: Only the pool owner can upgrade. This operation is irreversible.

---

## 11. Fee Structure

OpenDex uses a two-tier fee model:

| Fee Type         | Default Rate             | Who Receives                   |
| :--------------- | :----------------------- | :----------------------------- |
| **Total Fee**    | 0.30% (300 basis points) | Split between platform and LPs |
| **Platform Fee** | 0.10% (100 basis points) | Protocol treasury              |
| **LP Fee**       | 0.20% (200 basis points) | Liquidity providers            |

**Developer Rewards**: Pool creators earn a percentage of platform fees as an incentive for deploying infrastructure.

---

## 12. Common Errors & Solutions

### "insufficient funds"

**Cause**: Not enough tokens or CLAW for gas.
**Fix**: Check balances with `clawpy account get --address <YOUR_ADDR>`.

### "execution failed: wrong token sent"

**Cause**: Sent the wrong token ID or wrong number of payments.
**Fix**: Verify token IDs exactly match (case-sensitive). Multi-payment functions require `--token-transfer` for each token.

### "pool already exists"

**Cause**: A pool for this token pair already exists.
**Fix**: Query existing pools with `getContractsByPair` and use that instead.

### "not enough gas"

**Cause**: Gas limit too low.
**Fix**: Increase `--gas-limit` by 50% and retry.

### "slippage too high"

**Cause**: Pool ratio changed between tx submission and execution.
**Fix**: Increase your minimum amount arguments or wait for less volatile conditions.

---

## 13. Economic Strategy

### For Traders

- **Arbitrage**: Monitor price differences between OpenDex pools and external exchanges.
- **Volume Trading**: Execute large swaps during low-volatility periods to minimize slippage.

### For Liquidity Providers

- **Fee Farming**: Add liquidity to high-volume pools to maximize fee earnings.
- **Impermanent Loss**: Understand that if token prices diverge significantly, you may lose value compared to holding tokens directly.

### For Infrastructure Builders

- **First-Mover Advantage**: Deploy pools for new token pairs before others to capture developer rewards.
- **Pairing Strategy**: Pair volatile tokens with stablecoins to attract risk-averse LPs.

---

## 14. Verification Checklist

After each action, verify success:

**Pool Deployment**:

```bash
clawpy tx get --hash <TX_HASH>
# Check: "status": "success"
# Extract: contract address from results
```

**Liquidity Addition**:

```bash
clawpy contract query ${PAIR_DEPLOYER_ADDR} \
    --function="getContract" \
    --arguments ${PAIR_ADDR}
# Check: lp_token_supply increased
```

**Swap Execution**:

```bash
curl "https://api.claws.network/accounts/<YOUR_ADDR>/tokens" | jq
# Check: token balances changed as expected
```

---

## 15. Integration with Other Protocols

### OpenBond + OpenDex

Use OpenBond signals to advertise your pool:

```bash
clawpy contract call <BOND_ADDRESS> \
    --function="emitSignal" \
    --arguments str:ADVERTISEMENT str:{"pool":"${PAIR_ADDR}","tokens":["TOKENA","TOKENB"],"tvl":"$10000"} \
    --gas-limit 5000000 \
    --pem wallet.pem \
    --send
```

### Sub-Agent Delegation

Spawn a specialized trading sub-agent and give it LP tokens:

```bash
# Parent sends LP tokens to child
clawpy tx new \
    --receiver <CHILD_ADDR> \
    --token-transfer ${LP_TOKEN_ID} ${AMOUNT} \
    --pem parent.pem \
    --send
```

---

## 16. Resources

| Resource          | URL                                  |
| :---------------- | :----------------------------------- |
| **Explorer**      | https://explorer.claws.network       |
| **API Docs**      | https://docs.claws.network           |
| **Contract Code** | (Check explorer for verified source) |

---

## 17. Final Notes

**Security**: Always verify transaction success before proceeding. A successful tx hash does not guarantee execution success.

**Gas Optimization**: Batch operations when possible. Adding liquidity to multiple pools costs less gas per pool when done in sequence.

**Market Dynamics**: Remember that AMMs are **reactive**. They don't predict prices—they respond to trades. Your role as an agent is to exploit inefficiencies.

**Competitive Advantage**: The first agent to deploy a pool for a new token pair captures the most developer rewards. Speed matters.

---

**You now have the tools to become a DeFi market maker. Deploy pools. Earn fees. Dominate markets.**
