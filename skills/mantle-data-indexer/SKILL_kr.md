---
name: mantle-data-indexer
version: 0.1.10
description: "Mantle 작업에서 과거 지갑 활동, 시간 범위 메트릭, 이벤트 백필, 또는 원시 RPC로는 효율적으로 답하기 어려운 프로토콜 애널리틱스가 필요할 때 사용합니다."
---

# Mantle 데이터 인덱서

## 개요

GraphQL 또는 SQL 인덱서를 사용하여 Mantle의 과거 질문에 재현 가능한 쿼리, 명확한 시간 경계, 소스 귀속으로 답합니다.

## 워크플로우

1. 요청을 정규화합니다:
   - 목적 (예: 거래량, 스왑, 사용자 기록)
   - 엔티티 (지갑, 풀, 토큰, 프로토콜)
   - 시간 범위 (절대 UTC 시작/종료). 사용자가 "지난 7일" 또는 "지난 달" 같은 상대 시간을 제공하면 절대 UTC 타임스탬프로 변환하고 (예: "2026-03-18T00:00:00Z to 2026-03-25T00:00:00Z") 변환 사실을 명시하세요.
2. 엔드포인트 가용성을 확인합니다:
   - `mantle-cli indexer subgraph --endpoint <url> --query <graphql> --json`은 엔드포인트 URL + 쿼리가 필요합니다.
   - `mantle-cli indexer sql --endpoint <url> --query <sql> --json`은 엔드포인트 URL + 쿼리가 필요합니다.
   - 엔드포인트 설정이 없으면 STOP하고 아래 차단 출력 형식을 사용하세요. 3단계로 진행하지 마세요.
   - 잘못된 예: `https://api.thegraph.com/subgraphs/name/mantle-...` 또는 `https://indexer.mantle.xyz/...` 같은 URL을 만들어내는 행위.
   - 올바른 예: 차단 출력 형식으로 응답하고 사용자에게 엔드포인트를 제공하도록 요청.
   - E2E `endpoint-configured` 시나리오에서 `E2E_SUBGRAPH_ENDPOINT` 또는 `E2E_SQL_ENDPOINT`가 설정되지 않은 경우 건너뛰세요.
3. 가용성 및 지연 목표에 따라 소스를 선택합니다:
   - GraphQL 인덱서 → `mantle-cli indexer subgraph --endpoint <url> --query <graphql> --json`
   - SQL 인덱서 → `mantle-cli indexer sql --endpoint <url> --query <sql> --json`
4. `references/query-templates.md`에서 쿼리를 작성합니다.
5. 페이지네이션과 결정론적 정렬로 실행합니다.
6. 집계 전에 단위와 소수를 정규화합니다.
7. 쿼리 출처, 도구 경고, 주의사항과 함께 출력을 생성합니다.

## 가드레일

- 쿼리 전에 체인 범위가 Mantle인지 확인하세요.
- 절대 타임스탬프를 사용하고 시간대(`UTC`)를 포함하세요.
- 엔드포인트 URL을 발명하지 마세요. 없으면 차단 입력을 보고하고 엔드포인트를 요청하세요.
- SQL은 읽기 전용으로 유지하세요; 변형 문을 피하세요.
- 레이블 없이 다른 세분성의 데이터셋을 병합하지 마세요.
- `데이터 없음`과 `쿼리 실패`를 구분하세요.
- 도구 경고를 전파하세요 (예: `hasNextPage=true` 또는 SQL 절단).
- 인덱서 지연이 알려져 있거나 의심되면 공개하세요.

## 출력 형식

```text
Mantle 과거 데이터 보고서 (Mantle Historical Data Report)
- objective:
- source_type: graphql | sql
- source_endpoint:
- queried_at_utc:
- time_range_utc:
- entity_scope:

쿼리 요약 (Query Summary)
- dataset_or_subgraph:
- filters_applied:
- pagination_strategy:
- records_scanned:

결과 (Results)
- metric_1:
- metric_2:
- sample_rows:

품질 노트 (Quality Notes)
- indexer_lag_status:
- tool_warnings:
- assumptions:
- caveats:
- confidence:
```

## 차단 출력 형식

워크플로우가 진행될 수 없는 경우 (예: 엔드포인트 URL이 제공되지 않음), 반드시 구조화된 출력을 사용하세요. 엔드포인트 URL을 절대 조작하지 마세요. 대신 이 형식을 사용하세요:

```text
Mantle 과거 데이터 보고서 -- 차단됨 (Mantle Historical Data Report -- BLOCKED)
- objective:
- status: blocked
- blocked_reason: [정확히 무엇이 누락되었는지 설명, 예: "GraphQL 또는 SQL 엔드포인트가 제공되지 않았습니다"]
- action_required: [사용자가 제공해야 할 것, 예: "서브그래프 엔드포인트 URL 또는 SQL 인덱서 엔드포인트 URL을 제공하세요"]
- time_range_utc: [상대 시간을 절대 UTC로 변환, 예: "2026-02-23T00:00:00Z to 2026-03-25T00:00:00Z"]
- entity_scope:

품질 노트 (Quality Notes)
- tool_warnings: 엔드포인트 누락 -- 쿼리를 실행할 수 없습니다. 이것은 "데이터 없음" 결과가 아니라 설정 누락입니다.
- assumptions: none
- caveats: 쿼리가 실행되지 않았습니다. 엔드포인트가 제공된 후 결과를 사용할 수 있습니다.
- confidence: n/a
```

중요: 사용자가 엔드포인트 URL을 제공하지 않았고 설정된 URL이 없으면 반드시 위의 차단 형식을 사용하세요. 엔드포인트 URL을 추측, 조작, 또는 가정하지 마세요.

## 참조

- `references/indexer-selection-and-quality.md`
- `references/query-templates.md`
