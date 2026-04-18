---
name: mantle-defi-operator
version: 0.1.10
description: "Mantle DeFi 작업에서 발견(discovery), 장소 비교(venue comparison), 검증된 컨트랙트·사전 비행 증거·외부 핸드오프가 포함된 실행 준비 계획이 필요할 때 사용합니다."
---

# Mantle DeFi 운영자

## 개요

Mantle DeFi 인텐트의 결정론적 사전 실행 계획을 조율합니다. 이 스킬은 전문적인 주소, 위험, 포트폴리오 분석을 중복하지 않고 검증된 주소 조회, 사전 비행 증거, 실행 핸드오프 단계를 조율해야 합니다.

## CLI 우선 트랜잭션 구축

**항상 `mantle-cli` 명령어를 사용하여 미서명 트랜잭션을 구축하세요.** CLI는 검증된 주소와 ABI 인코딩으로 올바른 calldata를 생성합니다. 다음을 하지 마세요:
- calldata 또는 함수 선택자를 수동으로 구성하거나 hex 인코딩
- 텍스트 응답에서 컨트랙트 주소를 추출하여 직접 트랜잭션 구축
- 미서명 트랜잭션에 `from` 필드 추가 — Privy 및 다른 서명자를 깨뜨립니다

### 기능 카탈로그를 통한 도구 발견

트랜잭션을 구축하기 전에 **기능 카탈로그**를 CLI로 조회하여 사용 가능한 도구, 읽기/쓰기 특성, 지갑 요구사항, 호출 순서를 확인하세요:

```bash
mantle-cli catalog list --json                    # 모든 기능
mantle-cli catalog list --category execute --json  # 트랜잭션 구축 명령만
mantle-cli catalog search "swap" --json            # 키워드 검색
mantle-cli catalog show mantle_buildSwap --json    # 전체 세부사항 + CLI 명령 템플릿
```

각 항목에는 다음이 포함됩니다:
- `category: query` — 읽기 전용, 상태 변경 없음, 대부분 지갑 불필요
- `category: analyze` — 계산된 인사이트 (APR, 위험, 권장사항), 읽기 전용
- `category: execute` — 미서명 트랜잭션 구축, 지갑 주소 필요
- `workflow_before` — 특정 도구 전에 호출할 도구
- `cli_command` — 플레이스홀더가 포함된 정확한 CLI 명령 템플릿

### DeFi 작업에 사용 가능한 CLI 명령:

```bash
# 토큰 전송
mantle-cli transfer send-native --to <addr> --amount <n> --json
mantle-cli transfer send-token --token <token> --to <addr> --amount <n> --json

# 스왑 작업
mantle-cli swap build-swap --provider <dex> --in <token> --out <token> --amount <n> --recipient <addr> --json
mantle-cli swap approve --token <token> --spender <router> --amount <n> --json
mantle-cli swap wrap-mnt --amount <n> --json
mantle-cli swap unwrap-mnt --amount <n> --json
mantle-cli swap pairs --json

# Aave V3 대출
mantle-cli aave supply --asset <token> --amount <n> --on-behalf-of <addr> --json
mantle-cli aave set-collateral --asset <token> [--user <addr>] [--disable] --json  # 담보 활성화/비활성화 + 진단
mantle-cli aave borrow --asset <token> --amount <n> --on-behalf-of <addr> --json
mantle-cli aave repay --asset <token> --amount <n|max> --on-behalf-of <addr> --json
mantle-cli aave withdraw --asset <token> --amount <n|max> --to <addr> --json
mantle-cli aave positions --user <addr> --json   # 포지션 + 준비금별 collateral_enabled
mantle-cli aave markets --json

# 유동성 공급
mantle-cli lp top-pools [--sort-by volume|apr|tvl] [--limit <n>] [--provider <dex>] [--min-tvl <usd>] --json  # 최적 시작점 — 모든 DEX에서 상위 LP 기회 발견, 토큰 쌍 불필요
mantle-cli lp find-pools --token-a <t> --token-b <t> --json           # 특정 토큰 쌍의 모든 풀 발견
mantle-cli lp pool-state <pool-or-tokens> --json
mantle-cli lp suggest-ticks <pool-or-tokens> --json
mantle-cli lp analyze <pool-or-tokens> [--investment-usd <n>] --json  # 심층 풀 분석: APR, 위험, 범위 비교
mantle-cli lp positions --owner <addr> --json
mantle-cli lp add --provider <dex> --token-a <t> --token-b <t> (--amount-a <n> --amount-b <n> | --amount-usd <n>) --recipient <addr> --json
mantle-cli lp remove --provider <dex> --recipient <addr> --token-id <id> (--liquidity <n> | --percentage <1-100>) --json
mantle-cli lp collect-fees --provider <p> --token-id <id> --recipient <addr> --json

# 읽기 작업 (서명 불필요)
mantle-cli defi swap-quote --in <token> --out <token> --amount <n> --provider best --json
mantle-cli defi lending-markets --json
mantle-cli account balance <addr> --tokens USDC,USDT0 --json
```

모든 `--json` 출력에는 `to`, `data`, `value`, `chainId`가 포함된 `unsigned_tx`가 있습니다 — `from`을 추가하지 않고 서명자에게 직접 전달하세요.

## 사용하지 않을 경우

- 작업이 주소 조회, 화이트리스트 검증, 피싱 방지 검토만 해당하면 `mantle-address-registry-navigator`를 사용하세요.
- 작업이 `pass` / `warn` / `block` 사전 비행 판정만 반환하는 경우 `$mantle-risk-evaluator`를 사용하세요.
- 작업이 잔액 확인, 허용량 노출, 또는 지출자 위험 검토만 해당하면 `$mantle-portfolio-analyst`를 사용하세요.
- 사용자가 장소를 탐색 중이고 실행 준비 계획을 요청하지 않은 경우 `discovery_only` 모드에 머무르세요.

## 빠른 체크리스트

- `discovery_only`
  - 장소 제안, 근거, 발견 소스만 반환합니다.
  - 응답 어디에도 라우터 주소, hex 컨트랙트 주소(0x...), 승인 단계, calldata, 또는 시퀀싱을 반환하지 마세요.
  - `handoff_available`을 `no`로 설정합니다.
- `compare_only`
  - 검증된 장소를 비교하고 누락된 실행 입력을 지적합니다.
  - 검증된 레지스트리 키 또는 컨트랙트 역할(이름으로, hex 주소가 아님)은 허용하되, 승인 지시 또는 calldata는 제외합니다.
  - 응답 어디에도 hex 컨트랙트 주소(0x...)를 포함하지 마세요 — 근거에도, 필드에도 사용하지 마세요. 프로토콜 및 역할 이름만 사용하세요.
  - 증거가 없을 때 `risk_report_ref` / `portfolio_report_ref`를 비우거나 명시적으로 누락으로 표시합니다.
  - `handoff_available`을 `no`로 설정합니다.
- `execution_ready`
  - `address_resolution_ref`가 필요합니다.
  - 작업에 명시적으로 불필요한 경우가 아니면 `risk_report_ref`가 필요합니다.
  - 허용량 범위 또는 잔액 확인이 필요한 경우 `portfolio_report_ref`가 필요하며, 불필요한 이유를 설명합니다.
  - 그런 후에만 승인 계획, 시퀀싱, calldata, `handoff_available: yes`를 노출합니다.

## 워크플로우

1. 인텐트를 정규화합니다:
   - `swap`, `add_liquidity`, `remove_liquidity`, 또는 복합 흐름
   - 토큰 주소, 금액, 수신자, 데드라인, 슬리피지
2. `references/defi-execution-guardrails.md`에서 사전 검사를 실행합니다.
3. 요청된 액션에 필요한 레지스트리 키 또는 프로토콜 역할을 사용하여 `mantle-address-registry-navigator`에서 후보 프로토콜 컨트랙트를 확인합니다.
4. 계획 모드를 분류합니다:
   - `execution_ready`: 검증된 주소와 핸드오프를 생성하기에 충분한 견적/위험 증거
   - `compare_only`: 장소 비교는 가능하지만 실행 게이팅이 불완전; 사용자가 미검증 프로토콜을 지정한 경우에도 이 모드를 사용 — Protocol Selection의 `discovery_only` 아래에 나열하고, `readiness: blocked`로 설정하고, 검증된 큐레이션 대안을 권장
   - `discovery_only`: 실행 준비 없이 고수준 에코시스템 탐색 (특정 장소 비교 미요청)
5. `references/curated-defaults.yaml`에서 후보 세트를 구축합니다; 각 기본값의 신선도 메타데이터와 근거를 전달하고, 사용자가 다른 프로토콜을 지정하면 컨트랙트가 검증될 때까지 `compare_only`로 유지합니다.
6. `references/protocol-selection-policy.md`의 실시간 신호로 적격 후보만 순위를 매깁니다:
   - 스왑: 견적 품질, 최근 거래량, 풀 깊이, 슬리피지 위험
   - 유동성: TVL, 최근 거래량, 풀 적합성, 운영 복잡성
   - 대출: TVL, 활용도, 자산 지원, 출금 유동성
   - 실시간 메트릭이 오래되었거나 없으면 큐레이션 기본값으로 폴백하고 명시하세요
   - 요청된 액션에 적합한 큐레이션·검증된 후보가 하나뿐이면 최적화 후속 질문 전에 먼저 권장하세요
7. 실행 계획 전에 지원 증거를 수집합니다:
   - `mantle-address-registry-navigator`의 주소 신뢰
   - 상태 변경 경로가 준비될 때 `$mantle-risk-evaluator`의 사전 비행 판정
   - 승인 범위 또는 지갑 확인이 필요할 때 `$mantle-portfolio-analyst`의 허용량 및 지출 용량 컨텍스트
8. `planning_mode`에 따라 `Quick Checklist`를 사용하여 결과를 구조화합니다:
   - `recommended`
   - `also_viable`
   - `discovery_only`
   - 더 광범위한 에코시스템 발견을 위해서만 `DefiLlama`를 언급하고, 컨트랙트 진실 소스로는 절대 사용하지 마세요
9. `execution_ready` 계획에서만 작업 SOP를 로드합니다:
   - 스왑: `references/swap-sop.md`
   - 유동성: `references/liquidity-sop.md`
   - 대출 (공급/차입/상환/출금): `references/lending-sop.md`
10. 검증된 컨트랙트와 지원 증거로 프로토콜 선택이 게이팅된 후에만 견적, 풀, 또는 시장 경로를 확인합니다.
11. 승인이 필요한 경우 허용량 증거를 전달하고 최소 필요한 `approve` 단계를 준비합니다.
12. 계정이 배치(batching)를 지원하는 경우 (예: ERC-4337 스마트 계정), 외부 실행자가 approve+action을 안전하게 배치할 수 있는지 언급합니다.
13. 실행 핸드오프 계획을 생성합니다 (호출, 매개변수, 시퀀싱, 위험 노트). 서명, 브로드캐스트, 배포, 또는 실행을 주장하지 마세요.
14. 사용자가 외부 실행을 확인한 후 실행할 사후 실행 검증 검사(잔액, 허용량, 슬리피지)를 정의합니다.

## 가드레일

**⛔ 절대 금지 — 수동 트랜잭션 구성 ⛔**

어떠한 상황에서도 다음을 절대 하지 마세요:
- calldata, 함수 선택자, 또는 ABI 인코딩 매개변수를 직접 계산 (Python `encode_abi`, JS `encodeFunctionData`, 수동 hex, 또는 기타 방법)
- 토큰 금액 또는 wei 값을 수동으로 hex 인코딩
- `mantle-cli` 대신 `unsigned_tx` 객체를 수동으로 구성
- Python/JS 스크립트를 사용하여 트랜잭션 데이터 구축
- 수동으로 구성된 데이터로 `sign evm-transaction`, `eth_sendRawTransaction`, 또는 직접 브로드캐스트 도구 호출
- "CLI가 이 작업을 지원하지 않는다"는 이유로 수동 구성 정당화

**이 금지에는 예외가 없습니다.** CLI가 작업을 지원하지 않는다고 생각하면 틀린 것입니다 — 먼저 카탈로그를 확인하세요 (`mantle-cli catalog list --json`). 작업이 카탈로그에 정말로 없으면 안전한 인코딩 유틸리티를 사용하세요 (`mantle-cli utils encode-call`, `mantle-cli utils parse-units`). Python/JS를 사용하지 마세요.

**모든 전송에 사용 가능한 CLI 명령:**
```
mantle-cli transfer send-native --to <addr> --amount <n> --json        # 네이티브 MNT 전송
mantle-cli transfer send-token --token <sym> --to <addr> --amount <n> --json  # 모든 ERC-20 전송 (USDC, USDT, WMNT, BSB, ELSA 등)
```

**안전한 인코딩 유틸리티 (지원되지 않는 작업을 위한 탈출구):**
```
mantle-cli utils parse-units --amount <decimal> --decimals <n> --json   # 1단계: 소수 → raw/wei
mantle-cli utils encode-call --abi '<sig>' --function <name> --args '<json>' --json  # 2단계: ABI 인코딩 → calldata
mantle-cli utils build-tx --to <addr> --data <hex> [--value <mnt>] --json  # 3단계: Calldata → unsigned_tx
```

**실제 사례**: 에이전트가 USDC 전송에 `mantle-cli transfer send-token`을 우회하고, Python으로 calldata를 수동 계산하여 잘못된 인코딩을 생성했습니다. CLI 명령이 이를 올바르고 안전하게 처리했을 것입니다.

---

- **중복 트랜잭션 방지 (중요)**:
  - **인텐트당 하나의 빌드 호출**: 각 사용자 요청 액션(전송, 스왑, LP 등)에 대해 해당 빌드 도구를 정확히 한 번 호출하세요. 동일한 사용자 요청에 대해 동일하거나 유사한 매개변수로 같은 빌드 도구를 두 번 절대 호출하지 마세요. 이미 `unsigned_tx`를 받았으면 그 결과를 사용하세요 — "검증" 또는 "재시도"를 위해 빌더를 다시 호출하지 마세요.
  - **멱등성 키**: 모든 빌드 도구 응답에는 서명 지갑 범위의 결정론적 해시인 `idempotency_key`가 포함됩니다. 빌드 도구 호출 시 항상 `sender=<signing_wallet_address>`를 전달하세요 — 이렇게 하면 다른 지갑이 동일한 페이로드를 독립적으로 실행할 수 있습니다. 같은 지갑에서 빌더를 실수로 두 번 호출하고 동일한 `idempotency_key`를 받으면, 외부 서명자는 그 중 하나만 실행해야 합니다.
  - **투기적 빌드 금지**: "어떻게 보이는지 확인"하고 다시 실제로 빌드하지 마세요. 각 빌드 호출은 실행으로 이어질 수 있습니다.
  - **다음 단계 전에 대기**: 트랜잭션이 서명되고 브로드캐스트된 후, 다음 단계로 진행하기 전에 항상 영수증을 확인하세요 (`mantle-cli chain tx --hash <hash>`). 이전 트랜잭션이 온체인에서 확인되기 전에 절대 다음 트랜잭션을 제출하지 마세요.
  - **타임아웃 ≠ 실패**: 트랜잭션 제출이 타임아웃되거나 추적을 잃으면, 재구성 및 재제출하지 마세요. 대신, 지갑의 최근 트랜잭션을 확인하거나 트랜잭션 해시를 사용하여 상태를 확인하세요. 재구성은 다른 nonce를 가진 새 트랜잭션을 생성하여 중복 전송을 유발합니다.
- **CLI 우선 규칙**: 미서명 트랜잭션을 구축하는 데 항상 `--json`이 포함된 `mantle-cli` 명령어를 사용하세요. 표준 작업(전송, 스왑, LP, Aave)에는 전용 명령어를 사용하세요. 지원되지 않는 작업에는 utils 파이프라인을 사용하세요: `utils parse-units` → `utils encode-call` → `utils build-tx`. calldata를 구성하는 데 Python/JS/수동 hex를 절대 사용하지 마세요.
- **수동 Hex/Wei 구성 금지**: wei 값을 수동으로 계산하거나, 전송 금액을 hex 인코딩하거나, Python/JS를 사용하여 `amount * 10**decimals`를 계산하지 마세요. 소수→원시 변환에는 `mantle-cli utils parse-units`를 사용하세요. MNT는 `mantle-cli transfer send-native`, 모든 ERC-20은 `mantle-cli transfer send-token`을 사용하세요.
- **`from` 필드 금지**: `unsigned_tx` 객체에 `from` 필드를 절대 추가하지 마세요. 서명자는 서명 키에서 `from`을 결정합니다. `from` 추가는 Privy 및 기타 내장 지갑 서명자를 깨뜨립니다.
- **수동 라우팅 금지**: 중간 풀을 수동으로 발견하거나, 멀티홉 스왑을 별도 트랜잭션으로 분할하거나, 외부 애그리게이터/라우팅 서비스를 사용하지 마세요. CLI는 직접 쌍이 없을 때 브릿지 토큰(WMNT, USDC, USDT0, USDT, USDe, WETH)을 통해 2홉 경로를 자동으로 발견합니다. `--in`과 `--out`만 전달하면 CLI가 라우팅을 처리합니다.
- **USDT vs USDT0**: Mantle에는 두 가지 공식 USDT 변형이 있습니다 — USDT (브릿지된 Tether, `0x201E...`)와 USDT0 (LayerZero OFT, `0x779D...`). 둘 다 DEX에서 깊은 유동성을 보유합니다. **USDT0만 Aave V3에서 작동합니다.** 사용자가 USDT를 보유하고 Aave를 사용하려면, 먼저 Merchant Moe에서 USDT → USDT0로 스왑하도록 안내하세요 (USDT/USDT0 풀, bin_step=1).
- **팩토리 우선 풀 발견**: 특정 토큰 쌍의 LP 풀을 찾을 때 온체인 팩토리 컨트랙트를 쿼리하는 `mantle-cli lp find-pools --json`을 사용하세요. 사용자가 토큰을 지정하지 않고 최적 풀 또는 LP 권장사항을 요청할 때는 먼저 `mantle-cli lp top-pools --json`을 사용하세요.
- **LP 전 분석**: 유동성을 추가하기 전에 항상 `mantle-cli defi analyze-pool --json`을 실행하여 수수료 APR, 멀티 범위 비교, 위험 점수, 투자 예측을 받으세요. 어떤 범위나 금액에 투자할지 추측하지 마세요.
- **USD 금액 모드**: 사용자가 USD로 투자를 지정할 때 (예: "$1000 투자"), 토큰 금액을 수동으로 계산하는 대신 `--amount-usd`를 사용하세요.
- **비율 제거**: 사용자가 V3 포지션의 일부를 제거하려면 (예: "절반 제거"), 유동성 금액을 수동으로 읽고 계산하는 대신 `--percentage 50`을 사용하세요.
- **WETH는 Mantle에 존재합니다**: WETH (브릿지된 ETH)는 `0xdEAddEaDdeadDEadDEADDEAddEADDEAddead1111`에 있으며 약 125K ETH 공급량과 모든 DEX에서 풀이 있습니다. WETH가 없다고 주장하지 마세요.
- 서명, 브로드캐스트, 배포, 또는 트랜잭션 실행을 주장하지 마세요. "스왑을 실행했습니다", "트랜잭션이 제출되었습니다", "스왑 완료", "자금이 전송되었습니다" 같은 문구를 사용하지 마세요.
- 조율자 역할: 전문 주소, 위험, 포트폴리오 스킬이 적용될 때, 그 판단을 처음부터 다시 도출하는 대신 인용하거나 출력을 요청하세요.
- `discovery_only`에서 라우터 주소, 승인 단계, calldata, 실행 시퀀싱을 제공하지 마세요.
- `compare_only`에서 검증된 레지스트리 키 또는 컨트랙트 역할은 명명할 수 있지만, 실행 calldata와 승인 지시는 실행 증거가 완전해질 때까지 범위 밖으로 유지합니다.
- 명시적인 사용자 확인 없이 `warn`/`high-risk` 인텐트에 대한 외부 실행 계획으로 진행하지 마세요.
- 알 수 없거나 검증되지 않은 토큰/라우터/풀 주소를 거부하세요.
- 발견 데이터를 검증된 레지스트리 키의 대체물로 절대 취급하지 마세요.
- 공유 레지스트리에서 필요한 컨트랙트 역할을 확인할 수 없으면 계획을 `blocked`로 표시하세요.
- 실행 준비 옵션에서 명확히 분리된 후에만 발견 전용 프로토콜을 언급하세요.
- 외부 재시도를 위한 단계별 멱등성 노트를 유지하세요.
- 사용자가 온체인 실행을 요청하면 핸드오프 체크리스트를 제공하고 외부 서명자/지갑이 필요하다고 명시하세요.

## 출력 형식

**필수:** 모든 응답은 이 정확한 구조화된 템플릿을 사용해야 합니다. 이 템플릿 대신 산문이나 자유 형식 텍스트를 사용하지 마세요. 모든 필드를 채우고, 현재 모드에 적용되지 않는 필드에는 "현재 {planning_mode} 모드에 해당 없음"을 사용하세요. `discovery_only` 모드에서 Execution Handoff 및 Post-Execution Verification Plan 아래 필드는 모두 "discovery_only 모드에 해당 없음"으로 표시하세요.

```text
Mantle DeFi 사전 실행 보고서 (Mantle DeFi Pre-Execution Report)
- operation_type:
- planning_mode: discovery_only | compare_only | execution_ready
- environment:
- intent_summary:
- analyzed_at_utc:

준비 (Preparation)
- supporting_skills_used:
- address_resolution_ref:
- risk_report_ref:
- portfolio_report_ref:
- curated_defaults_considered:
- quote_source:
- expected_output_min:
- allowance_status:
- approval_plan:

프로토콜 선택 (Protocol Selection)
- recommended:
- also_viable:
- discovery_only:
- rationale:
- data_freshness:
- confidence: high | medium | low

실행 핸드오프 (Execution Handoff)
- recommended_calls:
- calldata_inputs:
- registry_key:
- sequencing_notes:
- batched_execution_possible: yes | no
- handoff_available: yes | no

사후 실행 검증 계획 (Post-Execution Verification Plan)
- balances_to_recheck:
- allowances_to_recheck:
- slippage_checks:
- anomalies_to_watch:

상태 (Status)
- preflight_verdict: pass | warn | block | unknown
- readiness: ready | blocked | needs_input
- blocking_issues:
- next_action:
```

## 참조

- `references/defi-execution-guardrails.md`
- `references/swap-sop.md`
- `references/liquidity-sop.md`
- `references/lending-sop.md`
- `references/curated-defaults.yaml`
- `references/protocol-selection-policy.md`
