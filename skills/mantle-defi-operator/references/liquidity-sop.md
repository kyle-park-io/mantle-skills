# Liquidity SOP

Use this flow for LP operations on Mantle (V3: Agni/Fluxion, LB: Merchant Moe).

## CRITICAL: Use CLI for All LP Operations

**ALWAYS use `mantle-cli` to build LP transactions and query positions.** The CLI handles pool resolution, tick range calculation, ABI encoding, and position enumeration.

```bash
# Read operations (no signing needed)
mantle-cli lp positions --owner 0x... --json          # List all V3 positions
mantle-cli lp pool-state --token-a USDC --token-b WMNT --fee-tier 10000 --provider agni --json
mantle-cli lp suggest-ticks --token-a USDC --token-b WMNT --fee-tier 10000 --provider agni --json
mantle-cli defi lb-state --token-a USDC --token-b USDT0 --bin-step 1 --json

# Write operations (returns unsigned_tx)
mantle-cli lp add --provider agni --token-a USDC --token-b WMNT --amount-a 10 --amount-b 15 --recipient 0x... --json
mantle-cli lp remove --provider agni --token-id 12345 --liquidity 1000000 --recipient 0x... --json
mantle-cli lp collect-fees --provider agni --token-id 12345 --recipient 0x... --json
```

## Step 1: Pre-operation Discovery

Before any LP operation, gather pool state and existing positions:

### V3 Pools (Agni/Fluxion)
```bash
# 1. Check pool state — get current tick, price, liquidity
mantle-cli lp pool-state --token-a USDC --token-b WMNT --fee-tier 10000 --provider agni --json

# 2. Get tick range suggestions
mantle-cli lp suggest-ticks --token-a USDC --token-b WMNT --fee-tier 10000 --provider agni --json

# 3. List existing positions
mantle-cli lp positions --owner 0x... --json
```

### Merchant Moe LB Pairs
```bash
# Check LB pair state — get active bin, nearby bin reserves
mantle-cli defi lb-state --token-a USDC --token-b USDT0 --bin-step 1 --json
```

## Step 2: Tick/Bin Range Selection

### V3 (Agni/Fluxion)
Use `lp suggest-ticks` to get pre-calculated ranges:
- **wide** (±200×tickSpacing): volatile pairs, less rebalancing
- **moderate** (±50×tickSpacing): balanced risk/reward
- **tight** (±10×tickSpacing): stablecoins, high concentration

Do NOT manually calculate ticks — use the CLI suggestion tool.

### Merchant Moe LB
Use `defi lb-state` to get the `active_id`, then:
- For stablecoins: use `delta_ids: [-2,-1,0,1,2]` centered on active bin
- For volatile: use wider range `delta_ids: [-5,-4,...,4,5]`
- Distribution: uniform `[1e18, 1e18, ...]` for even, or custom weights

## Step 3: Add Liquidity

### V3
```bash
mantle-cli lp add --provider agni \
  --token-a USDC --token-b WMNT \
  --amount-a 10 --amount-b 15 \
  --tick-lower <from_suggest> --tick-upper <from_suggest> \
  --fee-tier 10000 --recipient 0x... --json
```

### Merchant Moe
```bash
mantle-cli lp add --provider merchant_moe \
  --token-a USDC --token-b USDT0 \
  --amount-a 100 --amount-b 100 \
  --bin-step 1 --active-id <from_lb_state> \
  --delta-ids '[-2,-1,0,1,2]' \
  --distribution-x '[200000000000000000,200000000000000000,200000000000000000,200000000000000000,200000000000000000]' \
  --distribution-y '[200000000000000000,200000000000000000,200000000000000000,200000000000000000,200000000000000000]' \
  --recipient 0x... --json
```

## Step 4: Fee Collection (V3)

```bash
# Check accrued fees first
mantle-cli lp positions --owner 0x... --json
# Look for tokens_owed0 / tokens_owed1 > 0

# Collect fees
mantle-cli lp collect-fees --provider agni --token-id 12345 --recipient 0x... --json
```

## Step 5: Remove Liquidity

### V3
```bash
# First: check position details
mantle-cli lp positions --owner 0x... --json
# Note the token_id and liquidity amount

mantle-cli lp remove --provider agni \
  --token-id 12345 --liquidity <amount> \
  --recipient 0x... --json
```

### Merchant Moe
```bash
mantle-cli lp remove --provider merchant_moe \
  --token-a USDC --token-b USDT0 \
  --bin-step 1 \
  --ids '[8388608,8388609,8388610]' \
  --amounts '[1000000,1000000,1000000]' \
  --recipient 0x... --json
```

## Step 6: Post-operation Verification

- Re-read positions: `lp positions --owner 0x... --json`
- Check token balances changed as expected
- For V3: verify `in_range` status if market moved
- For Moe: verify bin balances via `defi lb-state`

## Common Pitfalls

- **Full-range V3 LP**: extremely capital-inefficient — always use `lp suggest-ticks` to pick a range
- **Wrong active_id for Moe**: always read fresh from `defi lb-state`, never hardcode
- **Missing approve**: both tokens must be approved for the router/position manager before adding
- **`from` field**: NEVER add to unsigned_tx
- **V3 position enumeration**: `lp positions` discovers all positions across both Agni and Fluxion
- **Fee harvesting**: use `lp collect-fees` standalone — no need to remove liquidity to collect fees
