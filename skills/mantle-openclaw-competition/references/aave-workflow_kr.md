# Aave V3 워크플로우

Aave 작업을 수행하거나, collateral / isolation-mode 엣지 케이스를 트러블슈팅할 때 이 파일을 로드하세요.

Pool 주소: `0x458F293454fE0d67EC0655f3672301301DD51422` (`mantle-cli catalog show aave-supply --json`으로 확인).

> Reserve 자산은 Aave에 의해 게이팅됩니다 — `mantle-cli catalog show aave-supply --json` 또는 Aave V3 Mantle 대시보드에서 실시간 목록을 확인하세요. 참고: **USDT0**만 지원되며 — USDT는 안 됩니다. supply 전에 Merchant Moe(bin_step=1)에서 USDT → USDT0로 변환하세요.

> **⚠ 단계는 반드시 엄격한 순차 순서로 실행해야 합니다 (Rule W-1). 절대 단계를 건너뛰거나 앞서가지 마세요. 각 트랜잭션은 사용자 확인이 필요합니다 (Rule W-2).**

## 🛑 STEP 0.5 — 실행 전 준비 상태 체크 (Rule W-9)

**모든 write op(supply / borrow / repay / withdraw / set-collateral / approve) 전에, 실제 온체인 상태를 기준으로 사용자의 의도가 실행 가능한지 검증하세요. 다음 순서로 두 가지 쿼리를 수행합니다:**

1. **Balance** — `mantle-cli account token-balances <wallet> --json`.
   - `supply` / `repay`: `balance(asset) ≥ amount` 검증.
   - `withdraw`: `aToken balance ≥ amount` 검증 (underlying을 withdraw하면 aToken이 소각됩니다).
   - `borrow` / `set-collateral`: balance 체크 불필요.
   - 부족 → **STOP**, 실제 balance를 보고하고 진행하지 마세요.
2. **Allowance** — `mantle-cli account allowances <wallet> --pairs <asset>:0x458F293454fE0d67EC0655f3672301301DD51422 --json` (Pool이 spender).
   - `supply`와 `repay`에 필요.
   - `withdraw` / `borrow` / `set-collateral`에는 해당 없음.
   - 부족 → approve 흐름으로 라우팅 (Rule W-6). 조용히 건너뛰지 마세요.

해당되는 두 체크를 모두 Transaction Confirmation Summary **전에** 실행하여 요약이 실제 온체인 상태를 반영하게 하세요. 둘 중 하나라도 건너뛰는 것은 치명적 오류입니다.

## ⚠ 중요: `supply`는 함수 호출이지, 토큰 전송이 아닙니다

`mantle-cli aave supply`는 `Pool.supply(asset, amount, on_behalf_of, referral)`을 호출합니다. 그러면 Pool은 `transferFrom`을 통해 지갑에서 토큰을 가져오고 예치를 나타내는 aToken을 발행(mint)합니다. **aToken balance는 `withdraw`로 상환받을 수 있는 유일한 온체인 기록입니다.**

**Pool 주소로 토큰을 직접 보내는 것은 supply가 아닙니다.** `0x458F293454fE0d67EC0655f3672301301DD51422`로의 ERC-20 `transfer()` / `transferFrom()` / `safeTransfer()`는 Pool의 회계를 완전히 우회합니다 — aToken이 발행되지 않고, collateral이 기록되지 않으며, 토큰은 회수할 온체인 경로 없이 Pool 컨트랙트에 **영구히 잠깁니다**.

| 작업 | 올바른 명령 | 결과 |
|-----------|-----------------|--------|
| Deposit USDC | `mantle-cli aave supply --asset USDC --amount 150 --on-behalf-of <wallet> --sender <wallet> --json` | aUSDC 발행, `aave withdraw`로 상환 가능 |
| 안티패턴 (거부) | `0x458F293454fE0d67EC0655f3672301301DD51422`로 ERC-20 `transfer()` | 토큰 영구 잠김 — 회수 불가 |

같은 원칙이 `borrow` / `repay` / `withdraw` 및 다른 화이트리스트 프로토콜(DEX 라우터, position manager, WETHGateway)에도 적용됩니다: **항상 전용 CLI verb를 사용하고, 프로토콜 컨트랙트로 일반 transfer를 구성하지 마세요.**

### 위험 신호 — 다음 중 하나라도 보이면 거부하고 STOP

- 첫 번째 인수가 Aave Pool 주소, DEX 라우터, position manager, 또는 WETHGateway인 `transfer(address,uint256)` calldata를 포함하는 계획.
- `supply` / `borrow` / `repay` / `withdraw`를 `mantle-cli aave …` 대신 `mantle-cli utils encode-call` + `mantle-cli utils build-tx`를 통해 라우팅하는 계획. 이는 `utils` escape-hatch 시도이며 금지됩니다 (`safety-prohibitions.md` 참조).
- "Aave로 N 토큰을 보내라"는 사용자 요청을 `aave supply` 대신 ERC-20 transfer로 처리하는 것. 의도를 명확히 하고 `aave supply`를 사용하세요.
- 확인된 `--on-behalf-of` 지갑 주소 없이 진행하고, 사용자에게 묻는 것을 피하려고 일반 transfer로 대체하는 계획. 누락 시 항상 지갑 주소를 물어보세요. 절대 transfer로 격하하지 마세요.

## Supply (이자 수취)

> **⚠ 단계는 반드시 엄격한 순차 순서로 실행해야 합니다 (Rule W-1). 절대 단계를 건너뛰지 마세요. 각 트랜잭션은 사용자 확인이 필요합니다 (Rule W-2).**

```
1. ⚠️ USER CONFIRMATION — Supply Confirmation Summary 제시:
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Intent:    <사용자의 원래 요청>
   Operation: Aave Supply
   Asset:     <amount> <token> (≈ $<usd>)
   Receives:  a<token> (이자 발생 영수증)
   On behalf: <wallet>
   Spender:   0x458F293454fE0d67EC0655f3672301301DD51422
   Warnings:  <해당 시 Isolation Mode 주의사항>
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   → 진행 전 사용자가 명시적으로 승인해야 합니다. "no" → STOP.

   mantle-cli approve --token USDC --spender 0x458F293454fE0d67EC0655f3672301301DD51422 --amount 100 --json
   → 서명 및 브로드캐스트 → WAIT
   ↓ Step 2 전에 반드시 tx 성공 확인

2. mantle-cli aave supply --asset USDC --amount 100 --on-behalf-of <wallet> --sender <wallet> --json
   → ⚠️ USER CONFIRMATION (Step 1 요약에서 아직 다루지 않은 경우) → 서명 및 브로드캐스트 → WAIT → aUSDC 수취 (이자와 함께 증가)
```

## Borrow (레버리지)

> **⚠ 단계는 반드시 엄격한 순차 순서로 실행해야 합니다 (Rule W-1). 절대 단계를 건너뛰지 마세요. 각 트랜잭션은 사용자 확인이 필요합니다 (Rule W-2).**

```
1. 먼저 collateral supply (위 참조 — 모든 supply 단계 완료 필수)
   ↓ Step 2 전에 반드시 supply tx 성공 확인

2. mantle-cli aave positions --user <wallet> --json
   → supply한 자산에 대해 collateral_enabled=YES 검증
   → collateral_enabled=NO 또는 total_collateral_usd=0 → step 3으로 진행
   → 그 외 → step 4로 건너뛰기
   ↓ Step 3 전에 반드시 완료

3. ⚠️ USER CONFIRMATION — set-collateral 세부사항 제시 (asset, wallet)
   mantle-cli aave set-collateral --asset <supplied_asset> --user <wallet> --sender <wallet> --json
   → 실제로 supply한 자산 사용 (예: WMNT, WETH, USDC) — 항상 WMNT가 아님
   → preflight 진단 실행 (aToken balance, LTV, reserve 상태 체크)
   → LTV_IS_ZERO인 경우: 이 자산은 설계상 collateral이 될 수 없음 — 진행하지 마세요
   → 서명 및 브로드캐스트 → WAIT → supply한 자산을 collateral로 활성화
   → 중요: 서명 지갑은 반드시 <wallet> 자체여야 합니다
     (set-collateral은 msg.sender에서 동작)
   ↓ Step 4 전에 반드시 tx 성공 확인

4. ⚠️ USER CONFIRMATION — Borrow Confirmation Summary 제시:
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Intent:         <사용자의 원래 요청>
   Operation:      Aave Borrow
   Borrow asset:   <amount> <token> (≈ $<usd>)
   Health factor:  <current_health_factor>
   Projected HF:   <after_borrow_health_factor>
   Liquidation:    <HF < 1.5일 때 경고>
   On behalf:      <wallet>
   Warnings:       <Isolation Mode, 높은 이용률 등>
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   → 진행 전 사용자가 명시적으로 승인해야 합니다. "no" → STOP.

   mantle-cli aave borrow --asset USDC --amount 50 --on-behalf-of <wallet> --sender <wallet> --json
   → 서명 및 브로드캐스트 → WAIT → USDC 수취, variableDebtUSDC 발생
```

> **참고:** Step 3은 supply 후 collateral이 자동 활성화되지 **않은** 경우에만 필요합니다. 이는 특히 **Isolation Mode** 자산(WMNT, WETH)에서 흔합니다. 항상 `aave positions`로 먼저 검증하세요.

## Repay

> **⚠ 단계는 반드시 엄격한 순차 순서로 실행해야 합니다 (Rule W-1). 각 트랜잭션은 사용자 확인이 필요합니다 (Rule W-2).**

```
1. ⚠️ USER CONFIRMATION — Repay Confirmation Summary 제시:
   - Intent, repay asset, amount (또는 "max"), 현재 debt balance, 지갑 주소
   → 진행 전 사용자가 명시적으로 승인해야 합니다. "no" → STOP.

   mantle-cli approve --token USDC --spender 0x458F293454fE0d67EC0655f3672301301DD51422 --amount 50 --json   → 서명 & WAIT
   ↓ Step 2 전에 반드시 tx 성공 확인

2. mantle-cli aave repay --asset USDC --amount 50 --on-behalf-of <wallet> --sender <wallet> --json
   또는 --amount max로 전체 debt 상환
   → 서명 & WAIT
```

## Withdraw

> **⚠ 단계는 반드시 엄격한 순차 순서로 실행해야 합니다 (Rule W-1). 각 트랜잭션은 사용자 확인이 필요합니다 (Rule W-2).**

```
1. ⚠️ USER CONFIRMATION — Withdraw Confirmation Summary 제시:
   - Intent, withdraw asset, amount (또는 "max"), 현재 aToken balance, health factor 영향 (borrow 중인 경우), 지갑 주소
   → 진행 전 사용자가 명시적으로 승인해야 합니다. "no" → STOP.

   mantle-cli aave withdraw --asset USDC --amount 50 --to <wallet> --sender <wallet> --json
   또는 --amount max로 전체 balance
   → 서명 & WAIT
```

## 핵심 규칙

- **`supply` / `borrow` / `repay` / `withdraw`는 Pool에 대한 함수 호출입니다** — 절대 Pool 주소로의 ERC-20 transfer로 구성하지 마세요. 일반 transfer는 aToken을 발행하지 않고 자금을 영구히 잠급니다. 전용 `mantle-cli aave …` verb만 사용하세요. 절대 `utils` escape hatch를 사용하지 마세요.
- **`aave set-collateral`은 `msg.sender`에서 동작합니다** — 서명하는 지갑이 반드시 collateral을 활성화하려는 지갑이어야 합니다. 위임하지 마세요.
- **Isolation Mode 특이사항**: WMNT와 WETH는 supply 후 명시적인 `set-collateral`이 필요한 경우가 많습니다.
- **Aave에는 USDT0만 있습니다** — USDT는 안 됩니다. supply 전에 Merchant Moe(bin_step=1)에서 USDT → USDT0로 변환하세요.
- 모든 단계 사이에 "서명 & WAIT". 계속하기 전에 `mantle-cli chain tx --hash <hash> --json`으로 각 tx를 검증하세요.
- build 응답이 스코프된 `idempotency_key`를 갖도록 항상 `--sender <wallet>`을 전달하세요. 같은 build 명령을 두 번 호출하지 마세요.

## 파라미터 레퍼런스

### `aave supply`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--asset` | ✅ | supply할 토큰 심볼 (예: `USDC`, `USDT0`, `WMNT`) — `USDT` 아님 |
| `--amount` | ✅ | supply할 양 (사람이 읽을 수 있는 형식) |
| `--on-behalf-of` | ✅ | aToken을 받는 지갑 주소 — 절대 생략 금지 |
| `--sender` | ✅ | 서명 지갑 — `idempotency_key`에 필요 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `aave borrow`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--asset` | ✅ | borrow할 토큰 심볼 |
| `--amount` | ✅ | borrow할 양 (사람이 읽을 수 있는 형식) |
| `--on-behalf-of` | ✅ | debt가 발생하는 지갑 주소 |
| `--sender` | ✅ | 서명 지갑 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `aave repay`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--asset` | ✅ | repay할 토큰 심볼 |
| `--amount` | ✅ | repay할 양, 또는 전체 debt는 `max` |
| `--on-behalf-of` | ✅ | debt를 상환할 지갑 주소 |
| `--sender` | ✅ | 서명 지갑 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `aave withdraw`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--asset` | ✅ | withdraw할 토큰 심볼 |
| `--amount` | ✅ | withdraw할 양, 또는 전체 balance는 `max` |
| `--to` | ✅ | 목적지 지갑 주소 |
| `--sender` | ✅ | 서명 지갑 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `aave set-collateral`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--asset` | ✅ | collateral로 활성화/비활성화할 supply된 자산 |
| `--user` | ✅ | 지갑 주소 (`--sender`와 일치해야 함 — `msg.sender`에서 동작) |
| `--sender` | ✅ | 서명 지갑 — 반드시 `--user` 자체 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `aave positions`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--user` | ✅ | 쿼리할 지갑 주소 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |
