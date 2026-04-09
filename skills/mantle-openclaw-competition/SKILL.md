---
name: mantle-openclaw-competition
description: Use when OpenClaw needs to execute DeFi operations for the asset accumulation competition on Mantle. Covers swap, LP, and Aave lending workflows with whitelisted assets and protocols.
---

# OpenClaw Competition — DeFi Operations Guide

## Overview

This skill provides everything OpenClaw needs to execute DeFi operations in the Mantle asset accumulation competition. Each participant starts with 100 MNT in a fresh wallet and competes to grow total portfolio value (USD) through whitelisted protocol interactions.

## When to Use

- User asks to swap tokens on Mantle
- User asks to add/remove liquidity on a DEX
- User asks to supply/borrow on Aave V3
- User asks about available assets or trading pairs
- User asks how to maximize yield or portfolio value

## Whitelisted Assets

Only these assets count toward the competition score:

### Core Tokens

| Symbol | Address | Decimals |
|--------|---------|----------|
| MNT | Native gas token | 18 |
| WMNT | `0x78c1b0C915c4FAA5FffA6CAbf0219DA63d7f4cb8` | 18 |
| WETH | `0xdEAddEaDdeadDEadDEADDEAddEADDEAddead1111` | 18 |
| USDC | `0x09Bc4E0D864854c6aFB6eB9A9cdF58aC190D0dF9` | 6 |
| USDT0 | `0x779Ded0c9e1022225f8E0630b35a9b54bE713736` | 6 |
| MOE | `0x4515A45337F461A11Ff0FE8aBF3c606AE5dC00c9` | 18 |

### xStocks RWA Tokens (Fluxion V3 pools, all paired with USDC, fee_tier 3000)

| Symbol | Address | Pool (USDC pair) |
|--------|---------|-----------------|
| wTSLAx | `0x43680abf18cf54898be84c6ef78237cfbd441883` | `0x5e7935d70b5d14b6cf36fbde59944533fab96b3c` |
| wAAPLx | `0x5aa7649fdbda47de64a07ac81d64b682af9c0724` | `0x2cc6a607f3445d826b9e29f507b3a2e3b9dae106` |
| wCRCLx | `0xa90872aca656ebe47bdebf3b19ec9dd9c5adc7f8` | `0x43cf441f5949d52faa105060239543492193c87e` |
| wSPYx | `0xc88fcd8b874fdb3256e8b55b3decb8c24eab4c02` | `0x373f7a2b95f28f38500eb70652e12038cca3bab8` |
| wHOODx | `0x953707d7a1cb30cc5c636bda8eaebe410341eb14` | `0x4e23bb828e51cbc03c81d76c844228cc75f6a287` |
| wMSTRx | `0x266e5923f6118f8b340ca5a23ae7f71897361476` | `0x0e1f84a9e388071e20df101b36c14c817bf81953` |
| wNVDAx | `0x93e62845c1dd5822ebc807ab71a5fb750decd15a` | `0xa875ac23d106394d1baaae5bc42b951268bc04e2` |
| wGOOGLx | `0x1630f08370917e79df0b7572395a5e907508bbbc` | `0x66960ed892daf022c5f282c5316c38cb6f0c1333` |
| wMETAx | `0x4e41a262caa93c6575d336e0a4eb79f3c67caa06` | `0x782bd3895a6ac561d0df11b02dd6f9e023f3a497` |
| wQQQx | `0xdbd9232fee15351068fe02f0683146e16d9f2cea` | `0x505258001e834251634029742fc73b5cab4fd67d` |

## Whitelisted Protocols & Contracts

### DEX: Merchant Moe (Liquidity Book AMM)

| Contract | Address | Operations |
|----------|---------|-----------|
| MoeRouter | `0xeaEE7EE68874218c3558b40063c42B82D3E7232a` | swap |
| LB Router V2.2 | `0x013e138EF6008ae5FDFDE29700e3f2Bc61d21E3a` | swap, add/remove LP |

**Key pairs:**
- USDC/USDT0: bin_step=1 (stablecoin)
- WMNT/USDC: bin_step=20
- WMNT/USDT0: bin_step=20
- WMNT/USDe: bin_step=20
- USDe/USDT0: bin_step=1

### DEX: Agni Finance (Uniswap V3 fork)

| Contract | Address | Operations |
|----------|---------|-----------|
| SwapRouter | `0x319B69888b0d11cEC22caA5034e25FfFBDc88421` | swap |
| PositionManager | `0x218bf598D1453383e2F4AA7b14fFB9BfB102D637` | add/remove LP |

**Key pairs:**
- WETH/WMNT: fee_tier=500 (0.05%)
- USDC/WMNT: fee_tier=10000 (1%)
- USDT0/WMNT: fee_tier=500 (0.05%)
- mETH/WETH: fee_tier=500 (0.05%)

### DEX: Fluxion (Uniswap V3 fork, native to Mantle)

| Contract | Address | Operations |
|----------|---------|-----------|
| SwapRouter | `0x5628a59df0ecac3f3171f877a94beb26ba6dfaa0` | swap |
| PositionManager | `0x2b70c4e7ca8e920435a5db191e066e9e3afd8db3` | add/remove LP |

**All xStocks pools**: USDC paired, fee_tier=3000 (0.3%). See xStocks table above for pool addresses.

### Lending: Aave V3

| Contract | Address | Operations |
|----------|---------|-----------|
| Pool | `0x458F293454fE0d67EC0655f3672301301DD51422` | supply, borrow, repay, withdraw |
| ProtocolDataProvider | `0x487c5c669D9eee6057C44973207101276cf73b68` | read-only queries |

**Supported reserve assets:** WETH, WMNT, USDT0, USDC, USDe, sUSDe, FBTC, syrupUSDT, wrsETH, GHO

## DeFi Operations — Step-by-Step

### How to Swap Tokens

**Pre-condition:** You have the input token in your wallet.

```
1. mantle_getSwapPairs({ provider: "<dex>" })
   → Find the pair and its params (bin_step or fee_tier)

2. mantle_getSwapQuote({ provider: "<dex>", token_in: "X", token_out: "Y", amount_in: "10" })
   → Get the expected output and minimum_out

3. mantle_getAllowances({ owner: "<wallet>", token: "X", spender: "<router>" })
   → Check if already approved

4. IF allowance < amount:
   mantle_buildApprove({ token: "X", spender: "<router>", amount: "<amount>" })
   → Sign and broadcast

5. mantle_buildSwap({ provider: "<dex>", token_in: "X", token_out: "Y", amount_in: "10", recipient: "<wallet>", amount_out_min: "<from_quote>" })
   → Sign and broadcast
```

**For MNT → Token swaps:** Wrap MNT first with `mantle_buildWrapMnt`, then swap WMNT.

### How to Add Liquidity

**Agni / Fluxion (V3 concentrated liquidity):**
```
1. Approve both tokens for the PositionManager
2. mantle_buildAddLiquidity({
     provider: "agni",
     token_a: "WMNT", token_b: "USDC",
     amount_a: "5", amount_b: "4",
     recipient: "<wallet>",
     fee_tier: 10000,
     tick_lower: <lower>, tick_upper: <upper>
   })
3. Sign and broadcast → Receive NFT position
```

**Merchant Moe (Liquidity Book):**
```
1. Approve both tokens for LB Router (0x013e138EF6008ae5FDFDE29700e3f2Bc61d21E3a)
2. mantle_buildAddLiquidity({
     provider: "merchant_moe",
     token_a: "WMNT", token_b: "USDe",
     amount_a: "5", amount_b: "4",
     recipient: "<wallet>",
     bin_step: 20,
     active_id: <from_pool>,
     delta_ids: [-5,-4,-3,-2,-1,0,1,2,3,4,5],
     distribution_x: [0,0,0,0,0,0,1e17,1e17,2e17,2e17,3e17],
     distribution_y: [3e17,2e17,2e17,1e17,1e17,0,0,0,0,0,0]
   })
3. Sign and broadcast → Receive LB tokens
```

### How to Use Aave V3

**Supply (earn interest):**
```
1. mantle_buildApprove({ token: "USDC", spender: "0x458F293454fE0d67EC0655f3672301301DD51422", amount: "100" })
   → Sign and broadcast

2. mantle_buildAaveSupply({ asset: "USDC", amount: "100", on_behalf_of: "<wallet>" })
   → Sign and broadcast → Receive aUSDC (grows with interest)
```

**Borrow (leverage):**
```
1. Supply collateral first (see above)
2. mantle_buildAaveBorrow({ asset: "USDC", amount: "50", on_behalf_of: "<wallet>" })
   → Sign and broadcast → Receive USDC, incur variableDebtUSDC
```

**Repay:**
```
1. mantle_buildApprove({ token: "USDC", spender: "0x458F293454fE0d67EC0655f3672301301DD51422", amount: "50" })
2. mantle_buildAaveRepay({ asset: "USDC", amount: "50", on_behalf_of: "<wallet>" })
   OR amount: "max" to repay full debt
```

**Withdraw:**
```
1. mantle_buildAaveWithdraw({ asset: "USDC", amount: "50", to: "<wallet>" })
   OR amount: "max" for full balance
```

## Safety Rules

1. **Never fabricate calldata** — Always use `mantle_build*` tools. Never construct tx data manually.
2. **Always check allowance before approve** — Don't approve if already sufficient.
3. **Always get a quote before swap** — Use `mantle_getSwapQuote` to know expected output.
4. **Wait for tx confirmation** — Do not build the next tx until the previous one is confirmed on-chain.
5. **Show `human_summary`** — Present every build tool's summary to the user before signing.
6. **Value field is hex** — The `unsigned_tx.value` is hex-encoded (e.g., "0x0"). Pass it directly to the signer.
7. **MNT is gas** — All gas costs are in MNT, not ETH.

## Competition Scoring

```
Net Value (USD) = Sum(token holdings * price) + Sum(aToken balances * price) - Sum(debtToken balances * price)
```

- aToken balances grow over time (interest earned)
- debtToken balances grow over time (interest owed)
- LP positions valued by underlying token amounts
- Only interactions with whitelisted contracts count
