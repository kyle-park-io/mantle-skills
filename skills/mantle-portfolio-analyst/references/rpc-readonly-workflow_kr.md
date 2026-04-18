# RPC 읽기 전용 워크플로우

`mantle-cli` 읽기 전용 명령만을 통해 지갑 잔액과 허용량을 수집하려면 이 가이드를 사용하세요. MCP 서버를 활성화하거나 연결하지 마세요.

## 필수 입력

- 지갑 주소
- 네트워크 (`mainnet` 또는 `sepolia`)
- 토큰 세트 및 지출자 세트 (사용자 제공 또는 발견)

## 호출 시퀀스

1. 지갑 형식 확인을 위해 `mantle-cli registry validate <wallet> --json`.
2. 네트워크 컨텍스트 확인을 위해 `mantle-cli chain info --json` (가능한 경우 `mantle-cli chain status --json`).
3. `mantle-cli account balance <wallet> --json`
4. `mantle-cli account token-balances <wallet> --tokens <token1>,<token2>,... --json`
5. `mantle-cli account allowances <wallet> --pairs <token1>:<spender1>,<token2>:<spender2> --json`
6. 선택적 메타데이터 백필: 심볼/소수점이 누락된 토큰에 `mantle-cli token info <token> --json`.

## 토큰 및 지출자 발견

- 먼저 명시적 사용자 범위를 선호하세요.
- 토큰 범위가 누락된 경우 `mantle://registry/tokens`를 읽고 범위를 위해 제한된 세트를 선택하세요.
- 지출자 범위가 누락된 경우 `mantle://registry/protocols`를 읽고 알려진 라우터/풀을 추출하세요.
- 잔액/허용량 호출 전에 현재 범위 목록 외의 심볼에 `mantle-cli token resolve <symbol> --json`을 사용하세요.
- 범위가 여전히 알 수 없으면 대상을 발명하는 대신 범위가 부분적이라고 보고하세요.

## 정규화 규칙

- CLI JSON 출력에서 이미 반환된 정규화된 값을 선호하세요.
- 소수점이 명시적으로 알려진 경우에만 원시 값을 수동으로 변환하세요.
- 출력에 `raw`와 `normalized` 값 모두 유지하세요.
- 소수점을 사용할 수 없으면 원시만 유지하고 신뢰도를 낮추세요.

## 신뢰성 검사

- 응답 체인/네트워크가 요청된 입력(`mainnet` 또는 `sepolia`)과 일치하는지 확인하세요.
- 제한된 시도로 일시적 읽기 실패를 재시도하세요; 추측된 토큰/지출자로 전환하지 마세요.
- 도구 수준의 `partial` 플래그와 항목별 `error` 필드를 통해 부분 실패를 감지하고 보고하세요.
- 최종 보고서에 CLI 출력의 `collected_at_utc` 값을 포함하세요.
