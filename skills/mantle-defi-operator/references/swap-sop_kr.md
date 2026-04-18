# 스왑 SOP

Mantle에서 토큰 스왑 사전 실행 분석을 위한 이 표준 절차를 사용하세요.

## 중요: 트랜잭션 구축에 CLI 사용

**항상 `mantle-cli`를 사용하여 미서명 트랜잭션을 구축하세요.** calldata를 수동으로 구성하거나, 텍스트에서 주소를 추출하거나, approve 호출을 직접 구축하지 마세요. CLI는 주소 해석, ABI 인코딩, 풀 매개변수 조회, 멀티홉 라우팅, 화이트리스트 검증을 올바르게 처리합니다.

```bash
# 스왑 트랜잭션 구축 — 직접 및 멀티홉 경로 모두 작동
mantle-cli swap build-swap --provider fluxion --in WMNT --out BSB \
  --amount 0.5 --recipient 0x... --json

# 승인이 필요한 경우
mantle-cli swap approve --token WMNT --spender <router_address> \
  --amount <exact_or_max> --owner <wallet> --json

# 사용 가능한 쌍 및 풀 매개변수 확인
mantle-cli swap pairs --provider fluxion --json
```

CLI 출력에는 `to`, `data`, `value`, `chainId`가 포함된 `unsigned_tx`가 있습니다 — `from` 필드 없이 서명자에게 직접 전달하세요.

## 멀티홉 라우팅 (내장)

직접 쌍이 없을 때 CLI가 **자동으로 멀티홉 경로를 발견합니다**. 중간 풀을 찾거나 멀티스텝 스왑을 직접 구축할 필요가 없습니다.

**작동 방식:** `--in A --out B`에 직접 풀이 없으면, CLI는 등록된 쌍 레지스트리를 사용하여 브릿지 토큰(WMNT, USDC, USDT0, USDT, USDe, WETH)을 통한 2홉 경로를 시도합니다. `A → bridge → B` 경로가 존재하면 단일 `exactInput` (V3) 또는 멀티 토큰 경로 (Merchant Moe) 트랜잭션을 구축합니다.

**자동 라우팅 스왑 예시:**

```bash
# WMNT → BSB  (자동 라우팅: Fluxion에서 WMNT → USDT0 → BSB)
mantle-cli swap build-swap --provider fluxion --in WMNT --out BSB --amount 0.5 --recipient 0x... --json

# WMNT → wTSLAx  (자동 라우팅: Fluxion에서 WMNT → USDC → wTSLAx)
mantle-cli swap build-swap --provider fluxion --in WMNT --out wTSLAx --amount 0.5 --recipient 0x... --json

# WMNT → ELSA  (자동 라우팅: Fluxion에서 WMNT → USDT0 → ELSA)
mantle-cli swap build-swap --provider fluxion --in WMNT --out ELSA --amount 0.5 --recipient 0x... --json
```

**하지 말아야 할 것:**
- 스왑을 두 개의 별도 트랜잭션으로 수동 분할하지 마세요 (예: WMNT→USDT0 후 USDT0→BSB). CLI가 더 나은 가스 효율로 단일 원자 멀티홉 트랜잭션으로 처리합니다.
- 중간 풀이나 브릿지 토큰을 직접 검색하지 마세요. CLI의 경로 발견은 검증된 쌍 레지스트리를 사용합니다.
- 애그리게이터나 외부 라우팅 서비스를 사용하지 마세요. `mantle-cli swap build-swap`만 사용하세요.

**멀티홉이 사용될 때**, 응답에는 다음이 표시됩니다:
- `intent: "swap_multihop"` (`"swap"` 대신)
- `human_summary`에 전체 경로가 표시됩니다 (예: "Fluxion에서 WMNT → USDT0 → BSB를 통해 0.5 WMNT → BSB 스왑")
- `warnings`에 경로 세부사항과 수수료 티어가 포함됩니다

## USDT vs USDT0

Mantle에는 두 가지 공식 USDT 변형이 있습니다 — 둘 다 합법적이고 깊은 DEX 유동성이 있습니다:
- **USDT** (`0x201EBa5CC46D216Ce6DC03F6a759e8E766e956aE`) — 브릿지된 Tether, Merchant Moe 및 기타 DEX에서 활성.
- **USDT0** (`0x779Ded0c9e1022225f8E0630b35a9b54bE713736`) — LayerZero OFT Tether, DEX 및 Aave V3에서 활성.

사용자가 "USDT"라고 하면 컨텍스트가 명확하지 않은 경우 어느 것을 의미하는지 확인하세요. Merchant Moe에 직접 USDT/USDT0 풀(bin_step=1)이 있어 두 가지 간 변환이 가능합니다.

## 1단계: 입력 정규화

- 토큰 입력/출력 심볼 또는 주소
- 정확한 입력 금액
- 수신자 주소
- 슬리피지 한도 (기본 0.5%)

## 2단계: 스왑 견적 받기 (필수)

```bash
mantle-cli defi swap-quote --in <token_in> --out <token_out> \
  --amount <amount> --provider best --json
```

반환 값:
- `provider`: 이 쌍에서 최고 출력을 가진 DEX
- `minimum_out_raw`: 빌드 단계에서 `--amount-out-min`으로 사용
- `router_address`: 승인 단계에서 `--spender`로 사용
- `resolved_pool_params`: 사용된 실제 `fee_tier` / `bin_step` / `pool_address`
- `source_trace`: 견적이 `onchain:*` (기본) 또는 `dexscreener:*` (폴백)에서 왔는지 표시

**중요:** 구축 전에 항상 견적을 받으세요. 견적은 샌드위치 공격을 방지하는 슬리피지 보호 값(`minimum_out_raw`)을 제공합니다. `provider` 필드는 빌드 단계에서 사용할 DEX를 알려줍니다.

**견적은 온체인 쿼터 컨트랙트를 사용합니다** (Agni QuoterV2, Moe LB Quoter) — `build-swap`이 풀 발견에 사용하는 것과 동일한 데이터 소스. 이는 견적과 빌드가 동일한 풀을 선택하도록 보장합니다.

### 네이티브 MNT 스왑

입력이 네이티브 MNT(WMNT가 아님)인 경우 먼저 래핑하세요:
```bash
mantle-cli swap wrap-mnt --amount <n> --json
# 래핑 트랜잭션 서명 및 브로드캐스트
# 그런 다음 견적과 스왑 모두에서 WMNT를 token_in으로 사용
```

## 3단계: 후보 프로토콜 선택

- 기본 선택으로 견적 응답의 `provider`를 사용하세요.
- xStocks RWA 토큰(wTSLAx, wAAPLx, wNVDAx 등)의 경우 → **Fluxion** 사용 (이 풀이 있는 유일한 DEX).
- BSB, ELSA, VOOI의 경우 → **Fluxion** 사용 (USDT0와 페어).
- 사용자가 다른 장소를 지정하면 비교하기 전에 컨트랙트를 검증하세요.

## 3단계: 토큰 메타데이터

- 레지스트리의 토큰에는 심볼을 직접 사용하세요.
- 알 수 없는 토큰에는 컨트랙트 주소를 전달하세요 — CLI가 온체인에서 소수점을 해석합니다.

## 5단계: 스왑 구축

```bash
mantle-cli swap build-swap --provider <provider_from_quote> \
  --in <token> --out <token> --amount <amount> \
  --recipient <wallet> \
  --amount-out-min <minimum_out_raw_from_quote> \
  --quote-provider <provider_from_quote> \
  --quote-fee-tier <fee_tier_from_resolved_pool_params> \
  --json
```

- 슬리피지 보호를 위해 견적의 `minimum_out_raw`에서 `--amount-out-min`을 전달하세요.
- 견적의 `resolved_pool_params`에서 `--quote-provider`와 `--quote-fee-tier`를 전달하세요 — 다른 풀을 해석하면 빌드가 경고를 발생시킵니다.
- CLI는 온체인에서 최적 풀(fee_tier, bin_step)을 자동 발견하고 LB Quoter를 통해 멀티홉 경로를 자동 라우팅합니다.
- 응답의 `pool_params`가 견적의 `resolved_pool_params`와 일치하는지 확인하세요.

## 6단계: 허용량 확인 및 승인

- 스왑 라우터 주소는 build-swap 응답의 `unsigned_tx.to` 필드에 있습니다.
- 해당 라우터에 대해 입력 토큰이 승인되었는지 확인하세요.
- 부족하면:
  ```bash
  mantle-cli swap approve --token <token> --spender <router_from_tx_to> --amount <exact_or_max> --owner <wallet> --json
  ```

## 7단계: 서명 및 브로드캐스트

- `unsigned_tx` 객체를 외부 서명자에게 직접 전달하세요.
- **`from` 필드를 추가하지 마세요.**
- **필드를 수정하지 마세요.**

## 8단계: 사후 실행 검증

- 스왑이 완료되었는지 확인하기 위해 잔액을 다시 읽으세요.
- 관찰된 출력과 예상 출력을 비교하세요.

## 흔한 함정

- **`from` 필드**: unsigned_tx에 `from`을 절대 추가하지 마세요 — Privy 및 내장 서명자를 깨뜨립니다.
- **수동 라우팅**: 풀을 수동으로 발견하거나 멀티홉을 별도 트랜잭션으로 분할하지 마세요 — CLI의 내장 라우팅을 사용하세요.
- **잘못된 풀 매개변수**: 등록된 쌍에 대해 `--fee-tier` 또는 `--bin-step`을 수동으로 지정하지 마세요 — CLI가 자동으로 해석합니다.
- **Merchant Moe 버전 열거**: CLI가 올바르게 처리합니다 (V1=0, V2.2=3); 재정의하지 마세요.
- **승인 누락**: 스왑은 라우터 컨트랙트에 대한 사전 ERC-20 승인이 필요합니다.
- **멀티홉 슬리피지**: 멀티홉 경로는 슬리피지 위험이 더 높습니다 — 가능하면 항상 먼저 견적을 받으세요.
