---
name: mantle-openclaw-competition
version: 0.1.10
description: "OpenClaw이 Mantle 자산 축적 경쟁에서 DeFi 작업을 실행해야 할 때 사용합니다. 화이트리스트된 자산과 프로토콜로 스왑, LP, Aave 대출 워크플로우를 다룹니다."
---

# OpenClaw 경쟁 — DeFi 운영 가이드

## 개요

이 스킬은 OpenClaw가 Mantle 자산 축적 경쟁에서 DeFi 작업을 실행하는 데 필요한 모든 것을 제공합니다. 각 참가자는 새 지갑에서 100 MNT로 시작하여 화이트리스트된 프로토콜 상호작용을 통해 총 포트폴리오 가치(USD)를 성장시켜 경쟁합니다.

## 툴링 — CLI 전용 (필수)

**`mantle-mcp` MCP 서버를 활성화하거나 연결하지 마세요.** 모든 온체인 작업은 `--json` 출력이 있는 `mantle-cli` 커맨드라인 도구를 통해 수행해야 합니다. 이는 MCP 도구 스키마 오버헤드를 제거하고 세션당 토큰 비용을 줄입니다.

### 설치

```bash
# mantle-cli 설치 (CLI만, MCP 서버 오버헤드 없음)
npm install @mantleio/mantle-cli
npx mantle-cli --help   # 확인
```

또는 이 스킬 저장소가 `package.json`과 함께 클론된 경우:

```bash
cd <skills-repo-root> && npm install
npx mantle-cli --help   # 확인
```

### 핵심 규칙

- **항상 `--json`을 추가하세요** — 출력이 기계로 파싱 가능한 JSON이 됩니다.
- **MCP 서버를 시작하거나 연결하지 마세요.** MCP 클라이언트 설정에서 `mantle-mcp`를 구성하지 마세요.
- **calldata를 조작하거나 트랜잭션을 수동으로 구성하지 마세요** — 항상 `mantle-cli` 빌드 명령을 사용하세요. 이에는 Python `encode_abi`, JS `encodeFunctionData`, 수동 hex `0xa9059cbb` 선택자, 기타 모든 방법이 포함됩니다. CLI는 모든 전송을 처리합니다: MNT는 `transfer send-native`, 모든 ERC-20은 `transfer send-token`.
- **미서명 트랜잭션에 `from` 필드를 절대 추가하지 마세요** — 서명자가 `from`을 결정합니다.

### 사용 가능한 명령 발견

CLI를 처음 사용하기 전에 실행하세요:

```bash
mantle-cli catalog list --json          # 카테고리, 인증, CLI 명령 템플릿이 있는 모든 37개 기능 나열
mantle-cli catalog search "swap" --json # 스왑 관련 기능 찾기
mantle-cli catalog show <tool-id> --json # 특정 기능 전체 세부사항
```

각 카탈로그 항목에는 다음이 포함됩니다:
- `category`: `query` (읽기 전용) | `analyze` (계산된 인사이트) | `execute` (미서명 트랜잭션 구축)
- `auth`: `none` | `optional` | `required` (지갑 주소 필요 여부)
- `cli_command`: 플레이스홀더가 있는 정확한 CLI 명령 템플릿
- `workflow_before`: 이 도구 전에 호출할 도구

## 사용 시점

- 사용자가 Mantle에서 토큰을 스왑하려는 경우
- 사용자가 DEX에서 유동성을 추가/제거하려는 경우
- 사용자가 Aave V3에서 공급/차입하려는 경우
- 사용자가 사용 가능한 자산이나 거래 쌍에 대해 묻는 경우
- 사용자가 수익률 또는 포트폴리오 가치를 최대화하는 방법을 묻는 경우

## 화이트리스트된 자산

다음 자산만 경쟁 점수에 포함됩니다:

### 핵심 토큰

| 심볼 | 주소 | 소수점 |
|--------|---------|----------|
| MNT | 네이티브 가스 토큰 | 18 |
| WMNT | `0x78c1b0C915c4FAA5FffA6CAbf0219DA63d7f4cb8` | 18 |
| WETH | `0xdEAddEaDdeadDEadDEADDEAddEADDEAddead1111` | 18 |
| USDC | `0x09Bc4E0D864854c6aFB6eB9A9cdF58aC190D0dF9` | 6 |
| USDT | `0x201EBa5CC46D216Ce6DC03F6a759e8E766e956aE` | 6 |
| USDT0 | `0x779Ded0c9e1022225f8E0630b35a9b54bE713736` | 6 |
| MOE | `0x4515A45337F461A11Ff0FE8aBF3c606AE5dC00c9` | 18 |

> **USDT vs USDT0:** 둘 다 Mantle의 공식 USDT입니다. 둘 다 DEX 유동성이 있습니다. USDT0만 Aave V3에 있습니다. Merchant Moe에서 USDT↔USDT0 스왑 (bin_step=1).

### xStocks RWA 토큰 (Fluxion V3 풀, 모두 USDC 페어, fee_tier 3000)

| 심볼 | 주소 | 풀 (USDC 페어) |
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

## 화이트리스트된 프로토콜 및 컨트랙트

### DEX: Merchant Moe (Liquidity Book AMM)

| 컨트랙트 | 주소 | 작업 |
|----------|---------|-----------|
| MoeRouter | `0xeaEE7EE68874218c3558b40063c42B82D3E7232a` | 스왑 |
| LB Router V2.2 | `0x013e138EF6008ae5FDFDE29700e3f2Bc61d21E3a` | 스왑, LP 추가/제거 |

**핵심 쌍:**
- USDC/USDT0: bin_step=1 (스테이블코인)
- USDC/USDT: bin_step=1 (스테이블코인)
- USDT/USDT0: bin_step=1 (스테이블코인 변환)
- USDe/USDT0: bin_step=1
- USDe/USDT: bin_step=1
- WMNT/USDC: bin_step=20
- WMNT/USDT0: bin_step=20
- WMNT/USDT: bin_step=15
- WMNT/USDe: bin_step=20

### DEX: Agni Finance (Uniswap V3 포크)

| 컨트랙트 | 주소 | 작업 |
|----------|---------|-----------|
| SwapRouter | `0x319B69888b0d11cEC22caA5034e25FfFBDc88421` | 스왑 |
| PositionManager | `0x218bf598D1453383e2F4AA7b14fFB9BfB102D637` | LP 추가/제거 |

**핵심 쌍:**
- WETH/WMNT: fee_tier=500 (0.05%)
- USDC/WMNT: fee_tier=10000 (1%)
- USDT0/WMNT: fee_tier=500 (0.05%)
- mETH/WETH: fee_tier=500 (0.05%)

### DEX: Fluxion (Uniswap V3 포크, Mantle 네이티브)

| 컨트랙트 | 주소 | 작업 |
|----------|---------|-----------|
| SwapRouter | `0x5628a59df0ecac3f3171f877a94beb26ba6dfaa0` | 스왑 |
| PositionManager | `0x2b70c4e7ca8e920435a5db191e066e9e3afd8db3` | LP 추가/제거 |

**모든 xStocks 풀**: USDC 페어, fee_tier=3000 (0.3%). 풀 주소는 위 xStocks 표 참조.

### 대출: Aave V3

| 컨트랙트 | 주소 | 작업 |
|----------|---------|-----------|
| Pool | `0x458F293454fE0d67EC0655f3672301301DD51422` | 공급, 차입, 상환, 출금 |
| ProtocolDataProvider | `0x487c5c669D9eee6057C44973207101276cf73b68` | 읽기 전용 쿼리 |

**지원 준비금 자산:** WETH, WMNT, USDT0, USDC, USDe, sUSDe, FBTC, syrupUSDT, wrsETH, GHO
> USDT0만 지원됩니다 — USDT는 지원되지 않습니다. 필요한 경우 먼저 USDT → USDT0로 스왑하세요.

## DeFi 작업 — 단계별

### 토큰 전송 방법

**네이티브 MNT 전송:**
```
mantle-cli transfer send-native --to <recipient> --amount <n> --json
→ 서명 및 브로드캐스트
```

**ERC-20 토큰 전송 (USDC, WMNT, USDT 등):**
```
mantle-cli transfer send-token --token <symbol> --to <recipient> --amount <n> --json
→ 서명 및 브로드캐스트
```

> **중요:** 전송을 위해 wei 값을 수동으로 계산하거나 금액을 hex 인코딩하지 마세요. 항상 위의 CLI 전송 명령을 사용하세요 — 소수점 변환을 결정론적으로 처리합니다.

### 토큰 스왑 방법

**사전 조건:** 지갑에 입력 토큰이 있어야 합니다.

```
1. mantle-cli swap pairs --json
   → 쌍과 매개변수 찾기 (bin_step 또는 fee_tier)

2. mantle-cli defi swap-quote --in X --out Y --amount 10 --provider best --json
   → 예상 출력 및 minimum_out 받기

3. mantle-cli account allowances <wallet> --pairs X:<router> --json
   → 이미 승인되었는지 확인

4. 허용량 < 금액이면:
   mantle-cli swap approve --token X --spender <router> --amount <amount> --json
   → 서명 및 브로드캐스트

5. mantle-cli swap build-swap --provider <dex> --in X --out Y --amount 10 --recipient <wallet> --amount-out-min <from_quote> --json
   → 서명 및 브로드캐스트
```

**MNT → 토큰 스왑의 경우:** 먼저 `mantle-cli swap wrap-mnt --amount <n> --json`으로 MNT를 래핑한 다음 WMNT로 스왑하세요.

### 유동성 추가 방법

**Agni / Fluxion (V3 집중 유동성):**
```
1. 두 토큰 모두 PositionManager에 승인
   mantle-cli swap approve --token <tokenA> --spender <position_manager> --amount <n> --json
   mantle-cli swap approve --token <tokenB> --spender <position_manager> --amount <n> --json

2. mantle-cli lp add \
     --provider agni \
     --token-a WMNT --token-b USDC \
     --amount-a 5 --amount-b 4 \
     --recipient <wallet> \
     --fee-tier 10000 \
     --tick-lower <lower> --tick-upper <upper> \
     --json

3. 서명 및 브로드캐스트 → NFT 포지션 수령
```

**Merchant Moe (Liquidity Book):**
```
1. 두 토큰 모두 LB Router에 승인 (0x013e138EF6008ae5FDFDE29700e3f2Bc61d21E3a)
   mantle-cli swap approve --token <tokenA> --spender 0x013e138EF6008ae5FDFDE29700e3f2Bc61d21E3a --amount <n> --json
   mantle-cli swap approve --token <tokenB> --spender 0x013e138EF6008ae5FDFDE29700e3f2Bc61d21E3a --amount <n> --json

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
     --json

3. 서명 및 브로드캐스트 → LB 토큰 수령
```

### Aave V3 사용 방법

**공급 (이자 획득):**
```
1. mantle-cli swap approve --token USDC --spender 0x458F293454fE0d67EC0655f3672301301DD51422 --amount 100 --json
   → 서명 및 브로드캐스트

2. mantle-cli aave supply --asset USDC --amount 100 --on-behalf-of <wallet> --json
   → 서명 및 브로드캐스트 → aUSDC 수령 (이자와 함께 증가)
```

**차입 (레버리지):**
```
1. 먼저 담보 공급 (위 참조)

2. mantle-cli aave positions --user <wallet> --json
   → 공급된 자산의 collateral_enabled=YES 확인
   → collateral_enabled=NO이거나 total_collateral_usd=0이면:

3. mantle-cli aave set-collateral --asset <supplied_asset> --user <wallet> --json
   → 공급한 실제 자산 사용 (예: WMNT, WETH, USDC) — 항상 WMNT가 아닙니다
   → 사전 비행 진단 실행 (aToken 잔액, LTV, 준비금 상태 확인)
   → LTV_IS_ZERO이면: 이 자산은 설계상 담보로 사용할 수 없습니다 — 진행하지 마세요
   → 서명 및 브로드캐스트 → 공급된 자산을 담보로 활성화
   → 중요: 서명 지갑이 반드시 <wallet> 자체여야 합니다 (set-collateral은 msg.sender에서 작동)

4. mantle-cli aave borrow --asset USDC --amount 50 --on-behalf-of <wallet> --json
   → 서명 및 브로드캐스트 → USDC 수령, variableDebtUSDC 발생
```

참고: 3단계는 공급 후 담보가 자동으로 활성화되지 않은 경우에만 필요합니다. 격리 모드 자산(WMNT, WETH)에서 특히 흔합니다. 항상 먼저 positions로 확인하세요.

**상환:**
```
1. mantle-cli swap approve --token USDC --spender 0x458F293454fE0d67EC0655f3672301301DD51422 --amount 50 --json
2. mantle-cli aave repay --asset USDC --amount 50 --on-behalf-of <wallet> --json
   또는 전체 부채 상환을 위해 --amount max
```

**출금:**
```
1. mantle-cli aave withdraw --asset USDC --amount 50 --to <wallet> --json
   또는 전체 잔액을 위해 --amount max
```

### LP 전 풀 분석 방법

유동성을 추가하기 전에 풀을 분석하여 최적 범위를 선택하고 수익을 추정하세요:

```
0. mantle-cli lp top-pools --sort-by apr --min-tvl 10000 --json
   → 모든 DEX에서 최고 풀 발견 (토큰 쌍 불필요) — 사용자가 "최고 LP" 또는 "어디에 유동성을 제공할지" 물을 때 사용

1. mantle-cli lp find-pools --token-a WMNT --token-b USDC --json
   → Agni, Fluxion, Merchant Moe에서 특정 쌍의 모든 풀 발견

2. mantle-cli defi analyze-pool --token-a WMNT --token-b USDC --fee-tier 3000 --provider agni --investment 1000 --json
   → 수수료 APR, 멀티 범위 비교, 위험 평가, 투자 예측 받기

3. mantle-cli lp suggest-ticks --token-a WMNT --token-b USDC --fee-tier 3000 --provider agni --json
   → 틱 범위 제안 받기 (광범위/중간/타이트 전략)
```

## 안전 규칙

**⛔ 절대 금지 — 수동 트랜잭션 구성 ⛔**

어떠한 상황에서도 다음을 절대 하지 마세요:
- calldata, 함수 선택자, 또는 ABI 인코딩 매개변수를 직접 계산 (Python, JS, 수동 hex, 기타)
- 토큰 금액이나 wei 값을 수동으로 hex 인코딩
- `mantle-cli` 대신 `unsigned_tx` 객체를 수동으로 구성
- 트랜잭션 데이터를 구축하는 Python/JS 스크립트 사용
- 수동으로 구성된 데이터로 `sign evm-transaction`, `eth_sendRawTransaction`, 직접 브로드캐스트 도구 호출
- "CLI가 이 작업을 지원하지 않는다"는 이유로 수동 구성 정당화

**이 금지에는 예외가 없습니다.** CLI가 지원하지 않는다고 생각하면 틀린 것입니다 — 먼저 카탈로그를 확인하세요. 정말로 없으면 아래 안전한 인코딩 유틸리티를 사용하세요. Python/JS를 사용하지 마세요.

**안전한 인코딩 유틸리티 (지원되지 않는 작업을 위한 탈출구):**
```
mantle-cli utils parse-units --amount <decimal> --decimals <n> --json   # 1단계: 소수 → raw/wei
mantle-cli utils format-units --amount-raw <raw> --decimals <n> --json  # 역방향: Raw/wei → 소수
mantle-cli utils encode-call --abi '<abi>' --function <name> --args '<json>' --json  # 2단계: ABI 인코딩 → hex calldata
mantle-cli utils build-tx --to <addr> --data <hex> [--value <mnt>] --json  # 3단계: calldata를 unsigned_tx로 래핑
```

**실제 사례**: 에이전트가 USDC 전송에 `mantle-cli transfer send-token`을 우회하고 Python으로 calldata를 수동 계산하여 잘못된 인코딩을 생성 — 자금 손실 발생. CLI 명령 `mantle-cli transfer send-token --token USDC --to <addr> --amount <n>`이 올바르고 안전하게 처리했을 것입니다.

---

0. **같은 트랜잭션을 절대 두 번 구성하지 마세요 (중요 — 자금 안전):**
   - 각 빌드 명령을 사용자 요청 액션당 정확히 한 번만 호출하세요. "확인" 또는 "재시도"를 위해 동일한 매개변수로 같은 빌드 명령을 두 번 절대 호출하지 마세요 — 구성된 각 트랜잭션은 서명되고 브로드캐스트될 수 있어 **중복 전송 및 돌이킬 수 없는 자금 손실**을 초래합니다.
   - 모든 빌드 응답에는 서명 지갑 범위의 `idempotency_key`가 포함됩니다. 빌드 도구 호출 시 항상 `sender=<signing_wallet>`을 전달하세요. 실수로 빌더를 두 번 호출하고 같은 키를 받으면 서명자는 하나만 실행해야 합니다.
   - 트랜잭션이 타임아웃되거나 추적을 잃으면 재구성하지 마세요. 먼저 영수증을 확인하세요: `mantle-cli chain tx --hash <hash>`. 원본이 이미 채굴되었을 수 있습니다. 재구성은 다른 nonce를 가진 새 트랜잭션을 생성하여 같이 실행됩니다.
   - **실제 사례**: 중복 빌드 호출로 인해 동일한 수신자에게 2× 전송 발생 — 0.2 MNT가 두 번, 0.608 MNT가 두 번 전송됨.
1. **CLI만 — MCP 절대 사용하지 마세요** — `mantle-cli ... --json`을 통한 모든 작업. MCP 서버를 활성화하거나 연결하지 마세요.
2. **CLI 외부에서 calldata를 절대 조작하지 마세요** — 항상 `mantle-cli` 빌드 명령 또는 `mantle-cli utils encode-call` + `mantle-cli utils build-tx`를 사용하세요.
3. **hex/wei 값을 절대 수동으로 계산하지 마세요** — `amount * 10**decimals` 또는 hex 인코딩에 Python, JS, 또는 암산을 절대 사용하지 마세요.
4. **승인 전 항상 허용량 확인** — 이미 충분하면 승인하지 마세요.
5. **스왑 전 항상 견적 받기** — 예상 출력과 슬리피지 보호를 위한 `minimum_out_raw`를 받으려면 `mantle-cli defi swap-quote`를 사용하세요.
6. **프로덕션에서 allow_zero_min 절대 설정하지 마세요** — 항상 스왑 견적에서 `amount_out_min`을 전달하세요. 슬리피지 보호 없는 스왑은 샌드위치 공격과 MEV 추출에 취약합니다.
7. **트랜잭션 확인 대기** — 이전 트랜잭션이 확인될 때까지 다음 트랜잭션을 구성하지 마세요.
8. **`human_summary` 표시** — 서명 전에 모든 빌드 명령의 요약을 사용자에게 제시하세요.
9. **Value 필드는 hex** — `unsigned_tx.value`는 hex 인코딩 (예: "0x0"). 서명자에게 직접 전달하세요.
10. **MNT는 가스, ERC-20이 아님** — MNT는 네이티브 가스 토큰입니다. MNT를 스왑하려면 먼저 WMNT로 래핑하세요 (`mantle-cli swap wrap-mnt`). MNT를 전송하려면 `mantle-cli transfer send-native`를 사용하세요.
11. **xStocks 토큰은 Fluxion 전용** — 모든 xStocks RWA 토큰은 Fluxion에서만 USDC 쌍 (fee_tier=3000)으로 유동성이 있습니다. Agni 또는 Merchant Moe에서 xStocks를 스왑하지 마세요 — 풀이 없어 트랜잭션이 실패합니다.
12. **브로드캐스트 후 트랜잭션 확인** — 사용자가 서명하고 브로드캐스트한 후 항상 `mantle-cli chain tx --hash <tx_hash> --json`으로 결과를 확인하세요. `status`가 `"success"`인지 확인하세요.
13. **서명 전 가스 추정** — 크거나 복잡한 작업의 경우 `mantle-cli chain estimate-gas --to <addr> --data <hex> --value <hex> --json`을 사용하여 서명 전에 예상 수수료를 MNT로 표시하세요.

## 다단계 작업 순서

많은 DeFi 작업은 엄격한 순서로 여러 트랜잭션이 필요합니다. **단계를 건너뛰거나 순서를 바꾸지 마세요** — 온체인 되돌리기와 가스 낭비를 초래합니다.

### 스왑 (MNT → 토큰)
```
1. mantle-cli swap wrap-mnt --amount <n>           → 서명 & 확인 대기
2. mantle-cli defi swap-quote --in WMNT --out X    → minimum_out_raw 받기
3. mantle-cli account allowances <wallet> --pairs WMNT:<router>  → 허용량 확인
4. 부족하면: mantle-cli swap approve ...    → 서명 & 확인 대기
5. mantle-cli swap build-swap ...                  → 서명 & 확인 대기
```

### Aave 공급 → 차입
```
1. mantle-cli swap approve --token X --spender 0x458F... --amount <n>  → 서명 & 대기
2. mantle-cli aave supply --asset X --amount <n> --on-behalf-of <wallet>  → 서명 & 대기
3. mantle-cli aave positions --user <wallet>       → collateral_enabled 확인
4. collateral_enabled=NO이면: mantle-cli aave set-collateral --asset X --user <wallet>  → 서명 & 대기
5. mantle-cli aave borrow --asset Y --amount <n> --on-behalf-of <wallet>  → 서명 & 대기
```

### 유동성 추가 (V3)
```
1. mantle-cli lp find-pools --token-a A --token-b B   → 풀 발견
2. mantle-cli defi analyze-pool ...                    → APR, 위험 확인
3. mantle-cli lp suggest-ticks ...                     → 틱 범위 권장 받기
4. mantle-cli swap approve --token A --spender <pm>    → 서명 & 대기
5. mantle-cli swap approve --token B --spender <pm>    → 서명 & 대기
6. mantle-cli lp add ...                               → 서명 & 대기
```

> **중요**: "서명 & 대기"는 다음 단계로 진행하기 전에 온체인 확인을 기다려야 함을 의미합니다 (`mantle-cli chain tx --hash <hash>`로 `status: success` 확인). 여러 미서명 트랜잭션을 파이프라인하지 마세요.

## 경쟁 점수

```
순 가치 (USD) = Sum(토큰 보유량 * 가격) + Sum(aToken 잔액 * 가격) - Sum(debtToken 잔액 * 가격)
```

- aToken 잔액은 시간이 지남에 따라 증가 (이자 획득)
- debtToken 잔액은 시간이 지남에 따라 증가 (이자 지불)
- LP 포지션은 기초 토큰 금액으로 가치 평가
- 화이트리스트된 컨트랙트와의 상호작용만 포함

## ⚠️ CLI 범위 경계 — 반드시 읽으세요

`mantle-cli`는 다음 **검증된 안전** 작업을 다룹니다:

| 카테고리 | 지원 작업 |
|----------|---------------------|
| **전송** | 네이티브 MNT 전송, ERC-20 토큰 전송 |
| **스왑** | Agni, Fluxion, Merchant Moe (직접 + 멀티홉) |
| **LP** | V3 추가/제거/수수료 수집 (Agni, Fluxion), LB 추가/제거 (Merchant Moe) |
| **대출** | Aave V3 공급, 차입, 상환, 출금, 담보 설정 |
| **유틸리티** | MNT 래핑/언래핑, ERC-20 승인, 트랜잭션 영수증, 가스 추정 |
| **읽기 전용** | 잔액, 견적, 풀 상태, 포지션, 가격, 체인 상태 |

**위에 나열되지 않은 작업은 CLI 지원이 없습니다.** 다음이 포함되지만 이에 한정되지 않습니다:
- 화이트리스트되지 않은 프로토콜 또는 컨트랙트와 상호작용
- 임의의 스마트 컨트랙트 함수 호출
- 화이트리스트되지 않은 지출자에 대한 토큰 승인
- 브릿지 작업, NFT 작업, 거버넌스/투표 작업

### 지원되지 않는 작업을 요청받으면

반드시 이 정확한 프로토콜을 따르세요:

1. **즉시 사용자에게 알리세요**: "⚠️ 이 작업은 검증된 CLI 툴링에서 지원되지 않습니다."
2. **위험을 명확히 설명하세요**: "이 요청의 모든 후속 작업은 수동으로 구성됩니다. 수동 트랜잭션 구성은 CLI의 안전 보장을 우회합니다."
3. **대안 제안**: 지원되는 작업으로 동일한 목표를 달성할 수 있으면 권장하세요.
4. **명시적 확인 요청**: 사용자가 위험을 이해하고 계속하려는 것을 명시적으로 말할 때까지 진행하지 마세요.
5. **사용자가 확인하면**: 수동으로 구성된 모든 트랜잭션에 `⚠️ UNVERIFIED MANUAL CONSTRUCTION`을 접두사로 붙이세요.
6. **의심스러우면 거부하세요**: 지원되지 않는 작업을 거부하는 것이 항상 자금 위험을 감수하는 것보다 안전합니다.
