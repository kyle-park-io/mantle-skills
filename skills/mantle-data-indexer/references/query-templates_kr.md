# 쿼리 템플릿

이 템플릿을 시작점으로 사용하세요. 플레이스홀더를 명시적으로 교체하세요.

## 도구 인수 매핑 (CLI)

- GraphQL 템플릿 → `mantle-cli indexer subgraph --json` 사용:
  - `--endpoint <url>` (필수)
  - `--query <graphql>` (필수)
  - `--variables <json>` (선택)
- SQL 템플릿 → `mantle-cli indexer sql --json` 사용:
  - `--endpoint <url>` (필수)
  - `--query <sql>` (필수, 읽기 전용 SQL만)
  - `--params <json>` (선택)
- E2E `endpoint-configured` 시나리오에서 엔드포인트 플레이스홀더는 `E2E_SUBGRAPH_ENDPOINT` / `E2E_SQL_ENDPOINT`에서 옵니다; 설정되지 않으면 해당 시나리오는 건너뜁니다.

## GraphQL: 지갑 스왑 기록 (템플릿)

```graphql
query WalletSwaps(
  $wallet: String!,
  $startTs: Int!,
  $endTs: Int!,
  $first: Int!,
  $skip: Int!
) {
  swaps(
    where: {
      trader: $wallet
      timestamp_gte: $startTs
      timestamp_lte: $endTs
    }
    orderBy: timestamp
    orderDirection: asc
    first: $first
    skip: $skip
  ) {
    id
    txHash
    timestamp
    tokenIn
    tokenOut
    amountIn
    amountOut
  }
}
```

## GraphQL: 풀 일별 거래량 (템플릿)

```graphql
query PoolDailyVolume(
  $poolId: String!,
  $startDay: Int!,
  $endDay: Int!,
  $first: Int!,
  $skip: Int!
) {
  poolDayDatas(
    where: {
      pool: $poolId
      date_gte: $startDay
      date_lte: $endDay
    }
    orderBy: date
    orderDirection: asc
    first: $first
    skip: $skip
  ) {
    date
    volumeUSD
    txCount
  }
}
```

## SQL: 지갑 활동 롤업 (템플릿)

```sql
SELECT
  DATE_TRUNC('day', block_time) AS day_utc,
  COUNT(*) AS tx_count,
  SUM(amount_usd) AS volume_usd
FROM swaps
WHERE chain_id = 5000
  AND LOWER(wallet_address) = LOWER(:wallet)
  AND block_time >= :start_utc
  AND block_time < :end_utc
GROUP BY 1
ORDER BY 1 ASC;
```

## SQL: 24시간 거래량 상위 풀 (템플릿)

```sql
SELECT
  pool_address,
  SUM(amount_usd) AS volume_24h_usd,
  COUNT(*) AS swap_count
FROM swaps
WHERE chain_id = 5000
  AND block_time >= :window_start_utc
  AND block_time < :window_end_utc
GROUP BY pool_address
ORDER BY volume_24h_usd DESC
LIMIT :limit_n;
```

## 템플릿 사용 규칙

- 매개변수와 출력에서 항상 타임스탬프 시간대를 UTC로 선언하세요.
- 대형 결과셋에는 결정론적 정렬 + 페이지네이션을 사용하세요.
- SQL은 읽기 전용으로 유지하세요; INSERT/UPDATE/DELETE/DDL 문을 피하세요.
- 금액이 원시 토큰 단위인지 USD 정규화인지 명시하세요.
- 보고서 부록에 교체된 매개변수 값을 포함하세요.
- 최종 보고서에 도구 경고(`hasNextPage=true`, `truncated`)를 전달하세요.
