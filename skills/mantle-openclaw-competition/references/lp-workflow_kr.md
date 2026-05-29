# 유동성 공급 워크플로우

유동성을 추가/제거하거나, 풀을 발견하거나, tick 범위를 제안할 때 이 파일을 로드하세요.

> **⚠ 단계는 반드시 엄격한 순차 순서로 실행해야 합니다 (Rule W-1). 절대 단계를 건너뛰거나 앞서가지 마세요. 각 트랜잭션은 사용자 확인이 필요합니다 (Rule W-2).**

## 🛑 STEP 0.5 — 실행 전 준비 상태 체크 (Rule W-9)

**모든 write op(add / remove / collect-fees / approve) 전에, 사용자의 의도가 실행 가능한지 검증하세요. 다음 순서로 두 가지 쿼리를 수행합니다:**

1. **Balance** — `mantle-cli account token-balances <wallet> --json`. 관련된 각 토큰에 대해 `balance(token) ≥ 계획된 입력`을 검증하세요 (V3와 LB add는 두 토큰을 받습니다; remove는 balance 체크가 필요 없습니다). 부족 → **STOP**, 실제 balance를 보고하고 진행하지 마세요.
2. **Allowance** — `mantle-cli account allowances <wallet> --pairs <tokenA>:<position_manager>,<tokenB>:<position_manager> --json`. 토큰별로 `allowance ≥ 계획된 입력`을 검증하세요. 부족 → approve 흐름으로 라우팅 (Rule W-6). 조용히 건너뛰지 마세요.

두 체크를 모두 Transaction Confirmation Summary **전에** 실행하여 요약이 실제 온체인 상태를 반영하게 하세요. 둘 중 하나라도 건너뛰는 것은 치명적 오류입니다.

## 풀 발견 & 분석 (LP 추가 전에 실행)

```
0. mantle-cli lp top-pools --sort-by apr --min-tvl 10000 --json
   → 모든 DEX에서 최고의 풀 발견 (토큰 쌍 불필요)
   → 사용자가 "best LP" 또는 "어디에 유동성을 공급할지" 물을 때 사용
   ↓ Step 1 전에 반드시 완료

1. mantle-cli lp find-pools --token-a WMNT --token-b USDC --json
   → Agni, Fluxion, Merchant Moe에서 특정 쌍에 사용 가능한 모든 풀 발견
   ↓ Step 2 전에 반드시 완료

2. mantle-cli defi analyze-pool --token-a WMNT --token-b USDC --fee-tier 3000 --provider agni --investment 1000 --json
   → fee APR, 다중 범위 비교, 위험 평가, 투자 예측 확인
   ↓ Step 3 전에 반드시 완료

3. mantle-cli lp suggest-ticks --token-a WMNT --token-b USDC --fee-tier 3000 --provider agni --json
   → tick 범위 제안 확인 (wide / moderate / tight 전략)
```

## 유동성 추가 — Agni / Fluxion (V3 concentrated)

> **⚠ 위의 모든 발견 단계(0-3)가 반드시 유동성 추가 전에 완료되어야 합니다.**

```
1. ⚠️ USER CONFIRMATION — LP Confirmation Summary 제시:
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Intent:    <사용자의 원래 요청>
   Operation: Add Liquidity (V3 Concentrated)
   DEX:       <provider>
   Token A:   <amount_a> <tokenA> (≈ $<usd>)
   Token B:   <amount_b> <tokenB> (≈ $<usd>)
   Fee Tier:  <fee_tier>
   Tick Range: <tick_lower> ~ <tick_upper> (<strategy: wide/moderate/tight>)
   Est. APR:  <apr>%
   Warnings:  <IL 위험, 좁은 범위 경고 등>
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   → approve로 진행하기 전 사용자가 명시적으로 승인해야 합니다. "no" → STOP.

   PositionManager에 두 토큰 approve:
   mantle-cli approve --token <tokenA> --spender <position_manager> --amount <n> --json   → 서명 & WAIT
   ↓ 반드시 tx 성공 확인
   mantle-cli approve --token <tokenB> --spender <position_manager> --amount <n> --json   → 서명 & WAIT
   ↓ Step 2 전에 반드시 tx 성공 확인

2. mantle-cli lp add \
     --provider agni \
     --token-a WMNT --token-b USDC \
     --amount-a 5 --amount-b 4 \
     --recipient <wallet> \
     --fee-tier 10000 \
     --tick-lower <lower> --tick-upper <upper> \
     --sender <wallet> \
     --json
   ↓ 반드시 tx 성공 확인

3. 서명 및 브로드캐스트 → WAIT → NFT 포지션 수취
```

각 provider의 PositionManager 주소는 `mantle-cli lp find-pools --json`이 반환하며 `mantle-cli catalog show lp-add --json`에 나열되어 있습니다.

## 유동성 추가 — Merchant Moe (Liquidity Book)

LB Router V2.2 주소: `0x013e138EF6008ae5FDFDE29700e3f2Bc61d21E3a`

> **⚠ 모든 발견 단계(0-3)가 반드시 진행 전에 완료되어야 합니다. 아래 단계는 엄격히 순차적입니다.**

```
1. ⚠️ USER CONFIRMATION — LP Confirmation Summary 제시:
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Intent:       <사용자의 원래 요청>
   Operation:    Add Liquidity (Liquidity Book)
   DEX:          Merchant Moe
   Token A:      <amount_a> <tokenA> (≈ $<usd>)
   Token B:      <amount_b> <tokenB> (≈ $<usd>)
   Bin Step:     <bin_step>
   Active ID:    <active_id>
   Delta IDs:    <delta_ids>
   Distribution: X=<distribution_x>, Y=<distribution_y>
   Warnings:     <IL 위험, bin 집중 경고 등>
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   → 진행 전 사용자가 명시적으로 승인해야 합니다. "no" → STOP.

   LB Router에 두 토큰 approve:
   mantle-cli approve --token <tokenA> --spender 0x013e138EF6008ae5FDFDE29700e3f2Bc61d21E3a --amount <n> --json   → 서명 & WAIT
   ↓ 반드시 tx 성공 확인
   mantle-cli approve --token <tokenB> --spender 0x013e138EF6008ae5FDFDE29700e3f2Bc61d21E3a --amount <n> --json   → 서명 & WAIT
   ↓ Step 2 전에 반드시 tx 성공 확인

2. mantle-cli lp add \
     --provider merchant_moe \
     --token-a WMNT --token-b USDe \
     --amount-a 5 --amount-b 4 \
     --recipient <wallet> \
     --bin-step 20 \
     --active-id <from_pool> \
     --delta-ids '[-5,-4,-3,-2,-1,0,1,2,3,4,5]' \
     --distribution-x '[0,0,0,0,0,0,1e17,1e17,2e17,2e17,3e17]' \
     --distribution-y '[3e17,2e17,2e17,1e17,1e17,0,0,0,0,0,0]' \
     --sender <wallet> \
     --json
   ↓ 반드시 tx 성공 확인

3. 서명 및 브로드캐스트 → WAIT → LB 토큰 수취
```

## 유동성 제거 / 수수료 수집

> **⚠ 각 remove/collect 작업은 실행 전 사용자 확인(Rule W-2)이 필요합니다.**

`mantle-cli lp remove`와 `mantle-cli lp collect-fees`를 사용하세요 — 인수는 `mantle-cli catalog show <tool-id> --json`을 참조하세요. 서명 전에 확인 요약(포지션 ID, 토큰, 양)을 사용자에게 제시하세요.

## 핵심 규칙

- **LP 작업은 position-manager / router 함수 호출이지, 토큰 전송이 아닙니다.** ERC-20 `transfer()`로 토큰을 PositionManager(Agni / Fluxion)나 LB Router(Merchant Moe)에 직접 보내는 것은 LP 포지션을 생성하지 **않습니다** — 토큰은 회수 경로 없이 컨트랙트에 **영구히 잠깁니다**. 항상 올바른 `mint()` / `increaseLiquidity()` / `addLiquidity()` 호출을 구성하는 `mantle-cli lp add`를 사용하세요. 사용자가 "position manager에 토큰을 보내라"거나 "LP 컨트랙트에 입금하라"고 하면 거부하고 대신 `lp add`를 사용하세요. 절대 `utils encode-call` + `build-tx`나 다른 어떤 방법으로도 position manager나 router로의 transfer를 구성하지 마세요.
- **항상 `--sender <wallet>`을 전달**하여 build 응답이 스코프된 `idempotency_key`를 갖도록 하세요.
- **타임아웃 후 절대 재빌드 금지** — 먼저 receipt를 확인하세요; 재빌드는 다른 nonce를 생성하여 그것도 실행됩니다.
- 모든 단계 사이에 "서명 & WAIT". 여러 LP 트랜잭션을 미리 빌드하지 마세요.
- xStocks LP는 USDC 쌍의 Fluxion에서만 작동합니다 (fee_tier=3000). 특정 풀은 `mantle-cli lp find-pools --token-a <xstock> --token-b USDC --json`으로 발견하세요.

## 파라미터 레퍼런스

### `lp top-pools`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--sort-by` | Optional | 정렬 키: `apr`, `tvl`, `volume` (기본값: `apr`) |
| `--min-tvl` | Optional | USD 기준 최소 TVL 필터 (예: `10000`) |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `lp find-pools`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--token-a` | ✅ | 첫 번째 토큰 심볼 |
| `--token-b` | ✅ | 두 번째 토큰 심볼 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `defi analyze-pool`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--token-a` | ✅ | 첫 번째 토큰 심볼 |
| `--token-b` | ✅ | 두 번째 토큰 심볼 |
| `--fee-tier` | ✅ | Fee tier (예: `3000`, `10000`) — V3 풀용 |
| `--provider` | ✅ | DEX provider (`agni`, `fluxion`, `merchant_moe`) |
| `--investment` | Optional | 예측용 USD 투자 금액 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `lp suggest-ticks`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--token-a` | ✅ | 첫 번째 토큰 심볼 |
| `--token-b` | ✅ | 두 번째 토큰 심볼 |
| `--fee-tier` | ✅ | Fee tier — V3 풀용 |
| `--provider` | ✅ | DEX provider |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `lp add` (V3 — Agni / Fluxion)

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--provider` | ✅ | `agni` 또는 `fluxion` |
| `--token-a` | ✅ | 첫 번째 토큰 심볼 |
| `--token-b` | ✅ | 두 번째 토큰 심볼 |
| `--amount-a` | ✅ | 토큰 A의 양 (사람이 읽을 수 있는 형식) |
| `--amount-b` | ✅ | 토큰 B의 양 (사람이 읽을 수 있는 형식) |
| `--recipient` | ✅ | NFT 포지션을 받을 주소 |
| `--fee-tier` | ✅ | 풀 fee tier (예: `3000`, `10000`) |
| `--tick-lower` | ✅ | 하한 tick (`lp suggest-ticks`에서) |
| `--tick-upper` | ✅ | 상한 tick (`lp suggest-ticks`에서) |
| `--sender` | ✅ | 서명 지갑 — `idempotency_key`에 필요 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `lp add` (LB — Merchant Moe)

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--provider` | ✅ | `merchant_moe` |
| `--token-a` | ✅ | 첫 번째 토큰 심볼 |
| `--token-b` | ✅ | 두 번째 토큰 심볼 |
| `--amount-a` | ✅ | 토큰 A의 양 (사람이 읽을 수 있는 형식) |
| `--amount-b` | ✅ | 토큰 B의 양 (사람이 읽을 수 있는 형식) |
| `--recipient` | ✅ | LB 토큰을 받을 주소 |
| `--bin-step` | ✅ | LB 풀의 bin step |
| `--active-id` | ✅ | Active bin ID (풀 상태에서) |
| `--delta-ids` | ✅ | active-id로부터의 bin 오프셋 JSON 배열 |
| `--distribution-x` | ✅ | 토큰-X 분포 가중치 JSON 배열 |
| `--distribution-y` | ✅ | 토큰-Y 분포 가중치 JSON 배열 |
| `--sender` | ✅ | 서명 지갑 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |
