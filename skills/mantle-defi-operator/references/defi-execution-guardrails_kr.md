# DeFi 사전 실행 가드레일

잠재적인 상태 변경 DeFi 액션 전에 이 컨트롤을 적용하세요.

## 기능 카탈로그를 통한 도구 발견

- 어떤 계획도 구성하기 전에 `mantle://registry/capabilities`를 읽어 사용 가능한 도구를 발견하세요.
- `category`로 필터링하세요: 읽기는 `query`, 인사이트는 `analyze`, 트랜잭션 구축은 `execute`.
- `auth`로 지갑 요구사항을 확인하세요: `required` 도구는 지갑 주소가 필요하고, `none` 도구는 필요하지 않습니다.
- `workflow_before`로 호출 순서를 이해하세요 (예: `buildSwap` 전에 `getSwapQuote`).
- 간단한 읽기 전용 작업(query/analyze)에는 기능 카탈로그로 충분합니다 — 스킬 로딩이 필요 없습니다.
- 실행 계획을 위해 아래 가드레일로 계속하세요.

## 기능 경계 (CLI 전용)

- 모든 온체인 작업은 `--json`이 포함된 `mantle-cli` 명령을 사용합니다. MCP 서버를 활성화하거나 연결하지 마세요.
- CLI는 쿼리를 위한 읽기 중심이고 쓰기를 위한 미서명 트랜잭션을 구축합니다 — 서명, 브로드캐스트, 배포, 또는 실행하지 않습니다.
- 이 스킬은 분석 + 계획 생성에서 중단해야 합니다.
- 트랜잭션 해시, 영수증, 또는 정산 결과를 절대 조작하지 마세요.

## ⛔ 절대 금지 — 수동 트랜잭션 구성 없음

어떠한 상황에서도 **절대** 하지 마세요:
- calldata, 함수 선택자, 또는 ABI 인코딩 매개변수 계산 (Python `encode_abi`, JS `encodeFunctionData`, 수동 `0xa9059cbb` 선택자, 또는 기타 방법)
- 토큰 금액 또는 wei 값을 수동으로 hex 인코딩
- `unsigned_tx` JSON 객체를 수동으로 구성
- 트랜잭션 데이터를 구축하는 Python/JS 스크립트 사용
- 수동으로 구성된 데이터로 `sign evm-transaction` 또는 `eth_sendRawTransaction` 호출
- "CLI가 이를 지원하지 않는다"는 이유로 수동 구성을 정당화 — 먼저 카탈로그를 확인하세요

**CLI는 모든 일반 작업을 지원합니다. 다음을 사용하세요:**
```bash
mantle-cli transfer send-native --to <addr> --amount <n> --json        # 네이티브 MNT
mantle-cli transfer send-token --token <sym> --to <addr> --amount <n> --json  # 모든 ERC-20 (USDC, USDT, WMNT 등)
mantle-cli swap build-swap ...                                         # DEX 스왑
mantle-cli swap approve ...                                            # ERC-20 승인
mantle-cli swap wrap-mnt / unwrap-mnt ...                              # 래핑/언래핑
mantle-cli lp add / remove / collect-fees ...                          # LP
mantle-cli aave supply / borrow / repay / withdraw / set-collateral ...  # Aave
```

**실제 사례**: 에이전트가 `mantle-cli`는 잔액만 확인하고 ERC-20 전송을 지원하지 않는다고 주장했습니다. 그런 다음 USDC 전송에 Python으로 calldata를 수동 계산하여 모든 안전 검사를 우회했습니다. 이것은 거짓입니다 — `mantle-cli transfer send-token --token USDC --to <addr> --amount <n>`이 결정론적인 소수점-wei 변환으로 모든 ERC-20 전송을 처리합니다.

정말로 지원되지 않는 작업이 필요한 경우, Python/JS 대신 안전한 인코딩 유틸리티를 사용하세요:
```bash
mantle-cli utils parse-units --amount <decimal> --decimals <n> --json   # 1단계: 소수 → raw/wei
mantle-cli utils encode-call --abi '<sig>' --function <name> --args '<json>' --json  # 2단계: ABI 인코딩 → calldata
mantle-cli utils build-tx --to <addr> --data <hex> [--value <mnt>] --json  # 3단계: Calldata → unsigned_tx
```
`build-tx` 출력에는 `⚠ UNVERIFIED MANUAL CONSTRUCTION` 경고가 포함됩니다. 결과 트랜잭션을 핸드오프에서 **UNVERIFIED**로 표시하세요.

## 조율 경계

- 이 스킬을 전문 주소, 위험, 포트폴리오 스킬을 대체하는 것이 아닌 최종 계획을 조립하는 데 사용하세요.
- 주소 신뢰는 `mantle-address-registry-navigator`로 라우팅하세요.
- pass/warn/block 판정은 `$mantle-risk-evaluator`로 라우팅하세요.
- 승인 범위 또는 잔액 확인이 필요할 때 허용량 및 잔액 증거는 `$mantle-portfolio-analyst`로 라우팅하세요.

## 주소 신뢰

- 공유 `mantle-address-registry-navigator` 레지스트리에서 실행 준비 토큰/라우터/풀/포지션 매니저 주소를 해석하세요.
- 검증되지 않거나 잘못된 형식의 주소에 대해 계획을 차단으로 표시하세요.
- 최종 핸드오프에서 선택된 레지스트리 키를 언급하세요.
- 발견 전용 프로토콜은 비교를 위해 언급될 수 있지만, 컨트랙트가 검증될 때까지 실행 대상이 아닙니다.
- 실시간 메트릭은 순위에 영향을 줄 수 있지만 주소 신뢰를 절대 확립하지 않습니다.

## 인텐트 완전성

- 작업 유형, 토큰 금액, 수신자, 슬리피지 한도, 데드라인이 있는지 확인하세요.
- 필수 필드가 누락된 경우 계획을 차단으로 표시하세요.

## 위험 결합

- 가능한 경우 `$mantle-risk-evaluator`의 최신 사전 비행 판정을 요구하세요.
- `warn`/`high-risk` 결과의 경우 명시적 사용자 확인을 요구하세요.
- `block` 결과의 경우 실행 준비 계획을 생성하지 마세요.

## 허용량 컨트롤

- 무제한 승인보다 필요한 최소 승인을 선호하세요.
- 허용량 범위, 지출자 노출, 또는 잔액 확인에 읽기 전용 증거가 필요할 때 `$mantle-portfolio-analyst`를 사용하세요.
- 무제한 승인이 요청되면 명시적 사용자 확인을 요구하세요.
- 외부 실행 체크리스트에 명시적 허용량 재확인을 포함하세요.

## 실행 핸드오프 무결성

- 선택된 견적/유동성 컨텍스트에서 결정론적 경로 및 calldata 입력을 사용하세요.
- 외부 실행자를 위한 필수 호출 시퀀스 및 매개변수 값을 기록하세요.
- 사용자가 확인한 실행 후 실행할 사후 실행 조정 검사(잔액/허용량/슬리피지)를 정의하세요.

## 트랜잭션 중복 제거 (중요)

모든 빌드 도구 응답에는 서명 지갑 범위의 결정론적 keccak256 해시인 `idempotency_key`가 포함됩니다. 키에는 `sender` (지갑 주소), `request_id` (호출자가 제공한 인텐트 ID), `unsigned_tx` 필드 (to, data, value, chainId)가 포함됩니다.

**범위 규칙:**
- **동일 지갑, 동일 calldata** → 동일 키 → 중복 제거 ✓
- **다른 지갑, 동일 calldata** → 다른 키 → 둘 다 실행 가능 ✓
- **동일 지갑, 동일 calldata, 다른 request_id** → 다른 키 → 둘 다 실행 가능 ✓

**에이전트 규칙:**
1. 사용자 인텐트당 각 빌드 도구를 정확히 한 번 호출하세요. "재시도" 또는 "확인"을 위해 다시 호출하지 마세요.
2. 빌드 도구 호출 시 항상 `sender=<signing_wallet_address>`를 전달하세요.
3. 사용자가 두 개의 동일한 개별 전송을 명시적으로 요청하면 각각에 별개의 `request_id`를 전달하세요.
4. 실수로 빌더를 두 번 호출했으면 `idempotency_key` 값을 비교하세요. 일치하면 중복을 버리세요.

**외부 서명자/실행자 규칙:**
1. 서명 전에 이 `idempotency_key`가 지난 5분 이내에 이미 서명되었는지 확인하세요.
2. 그렇다면 중복을 거부하고 원래 트랜잭션 해시를 반환하세요.
3. 브로드캐스트 후 중복 제거 조회를 위해 `(idempotency_key → tx_hash)`를 저장하세요.
4. `idempotency_scope.sender`가 `"unscoped"`이면 중복 제거 전에 서명 지갑 주소를 키에 주입하세요.

**타임아웃 후 재시도 규칙:**
1. 타임아웃 시 트랜잭션을 절대 재구성하지 마세요. 원본이 이미 채굴되었을 수 있습니다.
2. 대신: `mantle-cli chain tx --hash <original_hash>`로 영수증을 확인하세요.
3. 원본 해시가 삭제된 것으로 확인된 경우에만 재구성하세요 (어떤 블록에도 없고 mempool에도 없음).

## CLI 범위 경계

`mantle-cli`는 검증된 안전 작업(전송, 화이트리스트된 DEX 스왑, Aave V3, V3/LB LP)과 범용 인코딩 유틸리티를 다룹니다. 요청된 작업에 해당하는 전용 CLI 명령이 없는 경우:

1. **Python/JS/수동 hex를 사용하지 마세요.** 대신 CLI 유틸리티 파이프라인을 사용하세요:
   - `mantle-cli utils parse-units` — 소수 금액을 원시 정수로 변환
   - `mantle-cli utils encode-call` — 함수 호출을 ABI 인코딩
   - `mantle-cli utils build-tx` — calldata를 검증된 unsigned_tx로 래핑
2. **사용자에게 경고하세요** — 결과 트랜잭션은 `UNVERIFIED`입니다 — 전용 프로토콜별 CLI 명령으로 구축되지 않았습니다.
3. **명시적 사용자 확인 요구** — `⚠ UNVERIFIED` unsigned_tx에 서명하기 전에.
4. **의심스러우면 계획을 `blocked`로 표시하세요** — 사용자 자금 위험을 감수하는 것보다.
