# 유동성 SOP

Mantle에서 LP 작업을 위한 이 절차를 사용하세요 (V3: Agni/Fluxion, LB: Merchant Moe).

## 중요: 모든 LP 작업에 CLI 사용

**항상 `mantle-cli`를 사용하여 LP 트랜잭션을 구축하고 포지션을 쿼리하세요.** CLI는 풀 해석, 틱 범위 계산, ABI 인코딩, 포지션 열거를 처리합니다.

```bash
# 읽기 작업 (서명 불필요)
mantle-cli lp top-pools --sort-by volume --limit 20 --json  # 모든 DEX에서 최고 풀 발견 (토큰 쌍 불필요)
mantle-cli lp find-pools --token-a USDC --token-b USDe --json  # 특정 토큰 쌍의 모든 풀 발견
mantle-cli lp positions --owner 0x... --json          # 모든 V3 포지션 목록
mantle-cli lp pool-state --token-a USDC --token-b WMNT --fee-tier 10000 --provider agni --json
mantle-cli lp suggest-ticks --token-a USDC --token-b WMNT --fee-tier 10000 --provider agni --json
mantle-cli lp analyze --token-a USDC --token-b WMNT --fee-tier 10000 --provider agni --investment-usd 1000 --json
mantle-cli defi lb-state --token-a USDC --token-b USDT0 --bin-step 1 --json

# 쓰기 작업 (unsigned_tx 반환)
mantle-cli lp add --provider agni --token-a USDC --token-b WMNT --amount-a 10 --amount-b 15 --recipient 0x... --json
mantle-cli lp add --provider agni --token-a USDC --token-b WMNT --amount-usd 1000 --recipient 0x... --json  # USD 모드
mantle-cli lp remove --provider agni --token-id 12345 --liquidity 1000000 --recipient 0x... --json
mantle-cli lp remove --provider agni --token-id 12345 --percentage 50 --recipient 0x... --json  # 50% 제거
mantle-cli lp collect-fees --provider agni --token-id 12345 --recipient 0x... --json
```

## 1단계: 풀 발견 — 항상 여기서 시작

**사용자가 토큰을 지정하지 않고 LP 권장사항을 원할 때**, `top-pools`를 사용하여 모든 DEX에서 최고 기회를 발견하세요:

```bash
mantle-cli lp top-pools --sort-by volume --json                     # 24시간 거래량별 상위 풀
mantle-cli lp top-pools --sort-by apr --min-tvl 10000 --json        # 최소 TVL로 가장 높은 APR
mantle-cli lp top-pools --provider fluxion --sort-by apr --json     # 최고 Fluxion 풀
```

이는 Mantle의 모든 활성 풀(밈 토큰, xStocks, 새로 출시된 쌍 포함)에 대해 DexScreener를 쿼리하고, 수수료 APR을 계산하고, 순위 목록을 반환합니다.

**사용자가 토큰 쌍을 지정할 때**, `find-pools`를 사용하여 해당 특정 쌍의 모든 사용 가능한 풀을 발견하세요:

```bash
mantle-cli lp find-pools --token-a USDC --token-b USDe --json
```

이는 DEX 공급자, 수수료 티어/빈 스텝, 풀 주소, 유동성 상태가 포함된 모든 풀을 반환합니다. USDC/USDe의 예시 출력:
- Agni fee=100 (0.01%) — $1.7M 유동성
- Merchant Moe bin=1 — 활성

**결과를 사용하여 최적 풀을 선택**한 다음 풀 분석으로 진행하세요.

## 2단계: 풀 분석 — 유동성 추가 전 항상 실행

**풀을 선택한 후, `lp analyze`를 실행하여 APR, 위험, 최적 범위를 이해하세요.**

```bash
mantle-cli lp analyze --token-a USDC --token-b WMNT --fee-tier 10000 --provider agni --investment-usd 5000 --json
```

분석이 반환하는 것:
- **수수료 APR** — 24시간 거래량 / TVL 기반 (기본 및 10개 범위 브라켓에 걸쳐 집중)
- **위험 평가**: TVL 위험, 변동성 위험, 집중 위험
- **투자 예측**: 투자 금액에 대한 일별/주별/월별 수수료 수입
- **권장 범위**: 최근 변동성 기반 자동 선택 (±3× 일일 변동)
- **멀티 범위 비교**: APR, 집중 팩터, 리밸런싱 위험과 함께 ±1%부터 ±50%

이 데이터를 사용하여 정보에 기반한 범위 결정을 내리세요. 분석을 건너뛰고 틱 범위를 추측하지 마세요.

## 3단계: 풀 상태 및 틱 제안

분석 후 정확한 틱 경계를 가져옵니다:

### V3 풀 (Agni/Fluxion)
```bash
# 1. 풀 상태 확인 — 현재 틱, 가격, 유동성 가져오기
mantle-cli lp pool-state --token-a USDC --token-b WMNT --fee-tier 10000 --provider agni --json

# 2. 틱 범위 제안 받기 (사전 계산된 광범위/중간/타이트)
mantle-cli lp suggest-ticks --token-a USDC --token-b WMNT --fee-tier 10000 --provider agni --json

# 3. 기존 포지션 목록
mantle-cli lp positions --owner 0x... --json
```

### Merchant Moe LB 쌍
```bash
# LB 쌍 상태 확인 — 활성 빈, 인근 빈 준비금 가져오기
mantle-cli defi lb-state --token-a USDC --token-b USDT0 --bin-step 1 --json
```

## 4단계: 틱/빈 범위 선택

### V3 (Agni/Fluxion)
`lp analyze` (권장 범위)와 `lp suggest-ticks` 데이터를 사용하여 범위를 선택하세요:
- **타이트 (±1-3%)**: 스테이블코인 또는 낮은 변동성 쌍 — 가장 높은 APR이지만 자주 리밸런싱
- **중간 (±5-10%)**: 대부분의 쌍에 대한 균형 잡힌 위험/보상
- **광범위 (±15-50%)**: 변동성 있는 쌍, 리밸런싱 덜 필요

틱을 수동으로 계산하지 마세요 — CLI 도구를 사용하세요.

### Merchant Moe LB
`defi lb-state`로 `active_id`를 가져온 다음:
- 스테이블코인의 경우: 활성 빈을 중심으로 `delta_ids: [-2,-1,0,1,2]` 사용
- 변동성의 경우: 더 넓은 범위 `delta_ids: [-5,-4,...,4,5]` 사용
- 분배: 균등을 위해 균일 `[1e18, 1e18, ...]` 또는 커스텀 가중치

## 5단계: 유동성 추가

### 금액 모드

**토큰 금액** (명시적 제어):
```bash
mantle-cli lp add --provider agni \
  --token-a USDC --token-b WMNT \
  --amount-a 10 --amount-b 15 \
  --tick-lower <from_analyze> --tick-upper <from_analyze> \
  --fee-tier 10000 --recipient 0x... --json
```

**USD 금액** (자동 크기 조정 — 사용자 대면 흐름에 권장):
```bash
mantle-cli lp add --provider agni \
  --token-a USDC --token-b WMNT \
  --amount-usd 1000 \
  --tick-lower <from_analyze> --tick-upper <from_analyze> \
  --fee-tier 10000 --recipient 0x... --json
```

`--amount-usd` 모드:
- DexScreener/DefiLlama에서 실시간 토큰 가격 가져오기
- 대상 틱 범위에 대한 올바른 토큰 비율 계산을 위해 풀 상태 읽기 (순진한 50/50 분할이 아님)
- 응답 경고에 계산된 금액과 가격 보고
- 전체 범위 포지션이거나 풀 읽기 실패 시 50/50으로 폴백

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

## 6단계: 수수료 수집 (V3)

```bash
# 먼저 발생한 수수료 확인
mantle-cli lp positions --owner 0x... --json
# tokens_owed0 / tokens_owed1 > 0인지 확인

# 수수료 수집
mantle-cli lp collect-fees --provider agni --token-id 12345 --recipient 0x... --json
```

## 7단계: 유동성 제거

### V3 — 정확한 금액
```bash
mantle-cli lp remove --provider agni \
  --token-id 12345 --liquidity <amount> \
  --recipient 0x... --json
```

### V3 — 비율 모드 (사용자 대면 흐름에 권장)
```bash
# 포지션의 50% 제거
mantle-cli lp remove --provider agni \
  --token-id 12345 --percentage 50 \
  --recipient 0x... --json

# 전체 제거
mantle-cli lp remove --provider agni \
  --token-id 12345 --percentage 100 \
  --recipient 0x... --json
```

`--percentage` 모드는 포지션의 현재 유동성을 온체인에서 읽고 제거할 정확한 금액을 계산합니다. 원시 유동성 수를 위해 `lp positions`를 수동으로 쿼리할 필요가 없습니다.

### Merchant Moe
```bash
mantle-cli lp remove --provider merchant_moe \
  --token-a USDC --token-b USDT0 \
  --bin-step 1 \
  --ids '[8388608,8388609,8388610]' \
  --amounts '[1000000,1000000,1000000]' \
  --recipient 0x... --json
```

## 8단계: 사후 작업 검증

- 포지션 다시 읽기: `lp positions --owner 0x... --json`
- 토큰 잔액이 예상대로 변경되었는지 확인
- V3의 경우: 시장이 이동한 경우 `in_range` 상태 확인
- Moe의 경우: `defi lb-state`로 빈 잔액 확인

## 흔한 함정

- **풀 분석 건너뛰기**: 유동성을 추가하기 전에 항상 `lp analyze`를 실행하세요 — APR, 위험, 권장 범위를 보여줍니다
- **풀 발견 건너뛰기**: 항상 `lp find-pools`를 먼저 실행하세요 — DexScreener가 놓치는 풀을 찾습니다 (예: Agni fee=0.01% 스테이블코인 풀)
- **전체 범위 V3 LP**: 극도로 자본 비효율적 — 항상 분석 + 틱 제안을 사용하여 범위를 선택하세요
- **Moe의 잘못된 active_id**: 항상 `defi lb-state`에서 새로 읽고 하드코딩하지 마세요
- **승인 누락**: 추가 전에 두 토큰 모두 라우터/포지션 매니저에 승인되어야 합니다
- **수동 금액 계산**: 토큰 분할을 가격에서 수동으로 계산하는 대신 `--amount-usd` 사용
- **제거를 위한 수동 유동성 조회**: 포지션 유동성을 수동으로 읽는 대신 `--percentage` 사용
- **`from` 필드**: unsigned_tx에 절대 추가하지 마세요
- **V3 포지션 열거**: `lp positions`는 Agni와 Fluxion 모두에서 모든 포지션을 발견합니다
- **수수료 수확**: `lp collect-fees`를 독립적으로 사용하세요 — 수수료를 수집하기 위해 유동성을 제거할 필요가 없습니다
