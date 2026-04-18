# 주소 레지스트리 플레이북

`assets/registry.json`과 함께 이 파일을 사용하여 주소 조회를 결정론적이고 감사 가능하게 유지하세요.

## 해석 정책

- 주소 해석을 위한 CLI 명령 (MCP 서버는 사용하지 마세요):
  - 토큰 심볼/이름에는 `mantle-cli token resolve <symbol> --json`을 사용하세요.
  - 컨트랙트 키/별칭/레이블 조회에는 `mantle-cli registry resolve <identifier> --json`을 사용하세요.
  - 최종 주소를 반환하기 전에 `mantle-cli registry validate <address> --json`을 사용하세요.
- 자유 텍스트보다 기계 판독 가능한 소스를 선호하세요.
- 누락되거나 오래된 출처는 안전 실패로 처리하세요.
- 실패로 처리: 검증된 항목이 없으면 주소를 출력하지 않습니다.

## 레지스트리 필드

각 `contracts[]` 항목에는 다음이 포함되어야 합니다:

- `key`: 안정적인 조회 키 (`WETH`, `OFFICIAL_BRIDGE` 등)
- `label`: 사람이 읽을 수 있는 이름
- `environment`: `mainnet` 또는 `testnet` (런타임 `network=sepolia`는 `testnet`으로 매핑)
- `category`: `system`, `token`, `bridge`, 또는 `defi`
- `address`: EIP-55 체크섬 주소
- `status`: `active`, `deprecated`, `paused`, 또는 `unknown`
- `is_official`: 부울값
- `source.url`: 정규 소스 페이지
- `source.retrieved_at`: ISO-8601 타임스탬프
- `aliases`: 선택적 대체 이름/심볼
- `protocol_id`: DeFi 항목의 선택적 정규화된 프로토콜 슬러그
- `contract_role`: `router`, `quoter`, `position_manager`, `pool`, 또는 `pool_addresses_provider` 같은 선택적 정규화된 역할
- `supports`: `swap`, `add_liquidity`, `remove_liquidity`, `supply`, 또는 `withdraw` 같은 선택적 지원 작업 목록

## 조회 전략

1. CLI를 사용할 수 있으면 먼저 `mantle-cli token resolve`/`mantle-cli registry resolve`를 통해 해석합니다.
2. 로컬 폴백: `key`에 대한 정확한 일치.
3. `defi` 조회의 경우, 요청에 프로토콜과 역할이 포함되면 정확한 `protocol_id` + `contract_role`로 필터링합니다.
4. 로컬 폴백: 별칭/심볼에 대한 정확한 일치.
5. 로컬 폴백: `label`에 대한 대소문자 무감한 일치.
6. 동일한 프로토콜에 여러 활성 DeFi 항목이 남아 있으면 중단하고 역할 명확화를 요청합니다.

## 신선도 안내

- 지난 30일 이내에 검증된 항목을 선호하세요.
- 30일보다 오래되면 신뢰도를 `medium`으로 설정하세요.
- 타임스탬프가 없으면 신뢰도를 `low`로 설정하고 수동 확인을 요구하세요.
- `status`가 `active`가 아니면 실행 대상으로 취급하지 마세요.

## 업데이트 절차

1. 공식 Mantle 문서, 공식 프로토콜 문서, 공식 앱 설정 번들, 또는 공식 소스에서 링크된 검증된 익스플로러 컨트랙트 페이지에서 소스를 수집합니다.
2. `assets/registry.json`에서 항목을 업데이트하거나 추가합니다.
3. `source.retrieved_at` 및 최상위 `updated_at`을 현재 UTC 타임스탬프로 설정합니다.
4. 즉시 삭제하는 대신 `deprecated`로 표시하여 이전 항목을 보존합니다.
5. 실행 작업에 업데이트된 항목을 사용하기 전에 검증 검사를 다시 실행합니다.

## 권장 검증 검사

- 주소 형식 및 체크섬.
- 플레이스홀더/템플릿 값 없음 (예: `REPLACE_WITH_EIP55_CHECKSUM_ADDRESS`).
- 동일 환경 내 중복 키.
- 상충하는 레이블이 있는 중복 활성 주소.
- 소스 URL 또는 검색 타임스탬프 누락.
