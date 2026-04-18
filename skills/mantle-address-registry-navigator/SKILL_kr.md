---
name: mantle-address-registry-navigator
version: 0.1.10
description: "Mantle 작업에서 검증된 컨트랙트 또는 토큰 주소가 필요하거나, 화이트리스트 검증, 피싱 방지 확인, 온체인 상호작용 전 안전한 레지스트리 기반 조회가 필요할 때 사용합니다."
---

# Mantle 주소 레지스트리 내비게이터

## 개요

신뢰할 수 있는 소스에서만 주소를 확인하고, 데이터가 없거나 오래된 경우 실패로 처리하세요. 기억에서 컨트랙트 주소를 합성하지 마세요.

## 소스 우선순위

1. 토큰 심볼/이름 요청의 경우 `mantle-cli token resolve <symbol> --json` 사용.
2. 컨트랙트 키, 별칭, 레이블 요청의 경우 `mantle-cli registry resolve <identifier> --json` 사용.
3. 로컬 레지스트리 파일: `assets/registry.json` (폴백/수동 교차 확인 전용).
4. 어떤 소스도 검증된 일치 항목을 제공하지 않으면 중단하고 차단된 결과를 반환하세요.
5. **CLI 명령이 실패하면** (연결 오류, 도구 없음, 오프라인 모드), 로컬 레지스트리 폴백(3단계)으로 건너뛰고 신뢰도를 `medium`으로 제한하세요. CLI 사용 불가 사실을 응답의 `notes` 필드에 기록하세요.

## 조회 워크플로우

1. 요청을 정규화합니다:
   - `network` (`mainnet` 또는 `sepolia`; 로컬 레지스트리 폴백에서 `sepolia`는 `testnet`으로 매핑)
   - `identifier` (컨트랙트 키, 심볼, 또는 별칭)
   - `category` (system, token, bridge, `defi`, 또는 `any`)
   - DeFi 요청에서 프로토콜을 지정했지만 레지스트리 키를 지정하지 않은 경우 선택적 `protocol_id` + `contract_role`
2. 소스 우선순위에 따라 후보를 확인합니다. 로컬 DeFi 폴백의 경우, 먼저 정확한 `key` 일치를 선호하고, 다음으로 정확한 `protocol_id` + `contract_role`을 확인합니다.
3. `mantle-cli registry validate <address> --json` 및 레지스트리 메타데이터로 선택된 후보를 검증합니다:
   - `valid_format`이 `true`임.
   - 주소가 zero 주소가 아님.
   - 항목 환경이 요청과 일치함.
   - 실행을 위해 항목 상태가 사용 가능(`active`)함.
   - 항목에 출처(`source.url` 및 `source.retrieved_at`)가 있음.
   - 요청이 의도된 액션을 명시한 경우, 항목이 해당 액션을 `supports`함.
4. 출처 메타데이터와 함께 하나의 정규 결과를 반환합니다.
5. 여러 후보가 모호하게 남아 있으면 임의로 선택하는 대신 명확화 질문을 합니다.
6. **검증 요청** (사용자가 주소를 제공하고 공식 주소인지 묻는 경우): 먼저 동일한 소스 우선순위를 사용하여 명명된 컨트랙트/토큰의 정규 주소를 확인합니다. 그런 다음 사용자가 제공한 주소를 확인된 정규 주소와 비교합니다. 검증 불일치 형식으로 반환합니다.

## 안전 규칙

- 추측된 주소를 절대 출력하지 마세요.
- 레지스트리/도구 검증 없이 사용자 제공 주소를 신뢰하지 마세요.
- 더 이상 사용되지 않거나 일시 중지된 컨트랙트를 실행 불가로 표시하세요.
- `assets/registry.json`의 플레이스홀더/템플릿 값을 반환하지 마세요 (예: `REPLACE_WITH_EIP55_CHECKSUM_ADDRESS`).
- 레지스트리 최신성이 불명확한 경우 신뢰도를 `low`로 표시하고 수동 확인을 요청하세요.
- DeFi 요청에서 프로토콜을 명시했고 해당 프로토콜에 대해 동일한 `contract_role`을 공유하더라도 여러 활성 항목이 존재하면, 중단하고 사용자에게 필요한 구체적인 컨트랙트를 물어보세요. 명확화 필요 형식을 사용하고 모든 일치하는 후보를 나열하세요.

## 응답 형식

### 성공적인 조회

```text
주소 조회 결과 (Address Resolution Result)
- identifier:
- network:
- address:
- category:
- status:
- source_url:
- source_retrieved_at:
- confidence: high | medium | low
- notes:
```

### 차단됨 / 찾을 수 없음

검증된 항목이 없는 경우, 추측하는 대신 이 구조를 반환하세요:

```text
주소 조회 결과 (Address Resolution Result)
- identifier: [요청된 식별자]
- network: [요청된 네트워크]
- address: BLOCKED
- category: [요청되거나 추론된 카테고리]
- status: not_found
- source_url: N/A
- source_retrieved_at: N/A
- confidence: none
- notes: [조회 실패 이유 — 예: "이 식별자에 대한 도구 또는 로컬 레지스트리의 검증된 항목이 없습니다."]
```

### 검증 불일치 (피싱 방지)

사용자가 주소를 제공하고 공식 여부를 묻는 경우, 레지스트리와 비교하여 반환하세요:

```text
주소 검증 결과 (Address Verification Result)
- queried_address: [사용자 제공 주소]
- identifier: [레지스트리 키 일치 항목, 있는 경우]
- network: [네트워크]
- registry_address: [레지스트리의 주소, 또는 NONE]
- match: true | false
- status: [레지스트리 항목 상태, 또는 not_found]
- source_url: [레지스트리 소스 URL, 또는 N/A]
- source_retrieved_at: [레지스트리 타임스탬프, 또는 N/A]
- confidence: high | medium | low
- notes: [설명 — 예: "주소가 검증된 Aave v3 Pool 항목과 일치하지 않습니다."]
```

### 명확화 필요

여러 후보가 남아 있고 요청이 모호한 경우, 임의로 선택하지 마세요. 대신 다음으로 응답하세요:

```text
명확화 필요 (Clarification Needed)
- identifier: [요청된 식별자]
- network: [네트워크]
- candidates:
  1. [키] — [레이블] ([contract_role]) — [주소]
  2. [키] — [레이블] ([contract_role]) — [주소]
- question: [사용자에게 필요한 구체적인 컨트랙트 질문]
```

## 리소스

- `assets/registry.json`
- `references/address-registry-playbook.md`
