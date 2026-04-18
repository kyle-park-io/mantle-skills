---
name: mantle-portfolio-analyst
version: 0.1.10
description: "Mantle 작업에서 DeFi 또는 보안 결정 전에 지갑 잔액, 토큰 보유량, DeFi 포지션(Aave, LP), 허용량 노출, 또는 무제한 승인 검토가 필요할 때 사용합니다."
---

# Mantle 포트폴리오 분석가

## 개요

Mantle에서 결정론적이고 읽기 전용 지갑 분석을 구축합니다. 잔액, DeFi 포지션(Aave V3 대출, V3 LP, Merchant Moe LB LP), 허용량을 열거하고, 구조화된 보고서에서 승인 위험을 강조합니다.

## 워크플로우

**중요: 4단계부터 9단계까지는 모두 필수이며 정확한 순서대로 실행해야 합니다. 어떤 단계도 건너뛰거나 순서를 바꾸거나 병합하지 마세요. N단계가 결과를 반환할 때까지 N+1단계로 진행하지 마세요. 모든 단계는 도구 호출을 생성해야 합니다 — 모든 단계를 실행하지 않고 보고서를 작성하려고 하면 STOP하고 놓친 첫 번째 단계로 돌아가세요.**

1. 입력을 확인합니다:
   - `wallet_address`
   - `network` (`mainnet` 또는 `sepolia`)
   - 선택적 토큰/지출자 범위
2. 요청된 지갑 및 체인 컨텍스트를 검증합니다:
   - `mantle-cli registry validate <address> --json`
   - `mantle-cli chain info --json`
   - `mantle-cli chain status --json`
3. 분석 범위를 결정합니다:
   - 사용자 입력 또는 `mantle://registry/tokens`의 토큰 목록
   - 사용자 입력 또는 `mantle://registry/protocols`의 지출자 목록
4. **[필수]** `mantle-cli account balance <address> --json`으로 네이티브 잔액을 가져옵니다.
5. **[필수]** `mantle-cli account token-balances <address> --tokens <list> --json`으로 ERC-20 잔액을 가져옵니다.
   - 이 단계는 3단계에서 결정된 범위의 모든 토큰을 쿼리해야 합니다. 네이티브 잔액이 적거나 0이더라도 이 단계를 건너뛰지 마세요.
   - 도구 결과를 기다린 후 진행하세요. 결과에 `partial: true`가 포함되면 오류가 있는 토큰을 기록하지만 성공한 결과로 계속 진행하세요.
6. **[필수]** `mantle-cli aave positions --user <address> --json`으로 Aave V3 포지션을 가져옵니다.
   - 집계 계정 데이터(총 담보/부채 USD, 헬스 팩터)와 준비금별 공급/차입 금액을 반환합니다.
   - aToken 잔액은 **양의 자산**(담보)이고, debtToken 잔액은 **부채**입니다.
   - 보고서에 health_status를 포함하세요. `at_risk` 또는 `liquidatable` 포지션을 눈에 띄게 표시하세요.
7. **[필수]** `mantle-cli lp positions --owner <address> --json`으로 V3 LP 포지션을 가져옵니다.
   - 틱 범위, 유동성, 범위 내 상태가 포함된 Agni 및 Fluxion DEX의 포지션을 반환합니다.
   - 포트폴리오에 LP 자산으로 포함하세요.
8. **[필수]** `mantle-cli lp lb-positions --owner <address> --json`으로 Merchant Moe LB 포지션을 가져옵니다.
   - 사용자 지분 비율과 예상 토큰 금액이 포함된 LB 빈 포지션을 반환합니다.
   - 포트폴리오에 LP 자산으로 포함하세요.
9. **[필수]** `mantle-cli account allowances <owner> --pairs <token:spender,...> --json`으로 토큰-지출자 허용량을 가져옵니다.
10. 토큰의 메타데이터가 누락된 경우, 해당 토큰에 `mantle-cli token info <token> --json`을 사용하고 확인되지 않으면 누락된 필드를 `unknown`으로 유지하세요.
11. 다음 규칙을 사용하여 승인 위험을 분류하세요:
   - `low`: 허용량이 0이거나, 엄격하게 제한되어 있고 지갑 잔액/예상 사용량보다 명확히 낮습니다.
   - `medium`: 허용량이 0이 아니고 즉각적인 예상 사용량보다 크지만 여전히 제한됩니다.
   - `high`: 허용량이 예상 사용량에 비해 매우 크거나, 의도적으로 광범위하고 사용자 의도가 불분명합니다.
   - `critical`: 도구 출력의 `is_unlimited=true`, 또는 허용량이 최대 정수와 같거나 근접합니다(값 >= 2^255).
   - 항상 각 위험 레이블에 근거 문장을 포함하세요.
   - `mantle://registry/protocols`에서 확인되거나 사용자가 확인한 경우가 아니면 지출자 신뢰를 `unknown`으로 표시하세요.
   - 요약 상단에 모든 `high` 및 `critical` 승인을 강조하세요.
   - 토큰 소수점이 누락된 경우 원시 값을 사용하여 분류하고 신뢰도를 낮추세요.
12. DeFi 포지션 도구에서 부분 결과를 처리하세요:
   - `mantle-cli aave positions`가 `partial: true`를 반환하면, 오류가 있는 준비금을 기록하고 준비금별 세부사항이 불완전할 수 있지만 집계 USD 총계(`getUserAccountData`에서)는 정확하다고 명시하세요.
   - `mantle-cli aave positions`가 `possible_missing_reserves: true`를 반환하면, Aave 거버넌스가 이 도구에서 아직 추적하지 않는 새 준비금을 추가했을 수 있다고 경고하세요. 집계 USD 총계는 여전히 정확합니다.
   - 각 포지션의 `collateral_enabled` 필드를 확인하세요. 포지션에 `supplied > 0`이지만 `collateral_enabled: false`가 있는 경우:
     - `mantle-cli aave markets --json`을 실행하여 준비금의 `ltv_bps` 필드를 확인하세요.
     - `ltv_bps`가 0이면 (알려진 예: Mantle의 sUSDe, FBTC, syrupUSDT, wrsETH), **정보성**으로 분류 — 이 자산은 거버넌스 설계에 의해 담보로 사용할 수 없습니다, 사용자 오류가 아닙니다. set-collateral을 제안하지 마세요.
     - `ltv_bps` > 0이면, **담보 경고**로 표시 — 사용자가 토큰을 예치했지만 차입 용량에 포함되지 않고 있습니다. 이는 차입 실패를 유발할 수 있습니다. `set-collateral` 확인을 제안하세요.
   - `mantle-cli lp lb-positions`가 포지션을 반환하면, `coverage: "known_pairs_only"` 및 `scan_radius` 제한을 기록하세요. 원거리 빈 또는 비등록 쌍의 포지션이 확인되지 않았다고 명시하세요.
   - `mantle-cli lp lb-positions`가 `total_positions: 0`을 반환하면, 지갑에 LB 노출이 없다고 결론 내리지 마세요 — 스캔 범위 내에서 포지션이 발견되지 않았다고만 명시하세요.
13. **보고서 전 자가 점검**: 보고서를 작성하기 전에 다음 모든 도구 호출이 이루어졌고 결과를 반환했는지 확인하세요. 누락된 것이 있으면 지금 바로 실행하세요:
   - [ ] `mantle-cli account balance` (4단계)
   - [ ] `mantle-cli account token-balances` (5단계)
   - [ ] `mantle-cli aave positions` (6단계)
   - [ ] `mantle-cli lp positions` (7단계)
   - [ ] `mantle-cli lp lb-positions` (8단계)
   - [ ] `mantle-cli account allowances` (9단계)
14. 발견사항, 신뢰도, 명시적 범위/부분 결과 격차가 포함된 형식화된 보고서를 반환합니다.

## 가드레일

- **ERC-20 토큰 잔액 쿼리(5단계)를 절대 건너뛰지 마세요.** 이것은 가장 흔한 실행 실패입니다. token-balances 호출은 Aave/LP 쿼리 전에 이루어져야 하고 범위의 모든 토큰을 다루어야 합니다. Token Balances 섹션이 누락된 보고서는 불완전하며 반환해서는 안 됩니다.
- 이 스킬에는 `mantle-cli` 읽기 전용 명령만 사용하세요. MCP 서버를 활성화하거나 연결하지 마세요.
- 읽기 전용으로 유지하세요; 트랜잭션을 구성하거나 전송하지 마세요.
- 이 워크플로우에서 직접 JSON-RPC 호출(`eth_*`)을 호출 가능한 도구인 것처럼 참조하지 마세요.
- 호출이 실패하면 토큰 소수점이나 심볼을 추측하지 마세요.
- 지갑, 토큰, 지출자에 대한 체크섬 주소를 검증하세요. 주소가 체크섬 검증에 실패하거나 zero 주소(`0x0000...0000`)이면, 중단하고 오류 메시지를 반환하세요 — 잘못된 주소에 대한 쿼리를 진행하지 마세요.
- 누락된 토큰 메타데이터를 `unknown`으로 표시하고 계속하세요.
- RPC 응답이 일치하지 않으면 부분 범위를 명시적으로 보고하세요.
- 출력에 `raw`와 `normalized` 값 모두 유지하세요. 도구 응답의 정규화된 값을 선호하고, 소수점이 명시적으로 알려진 경우에만 수동으로 변환하세요. 소수점을 사용할 수 없으면 원시만 유지하고 신뢰도를 낮추세요.
- 응답 체인/네트워크가 요청된 입력과 일치하는지 확인하세요. 도구 수준의 `partial` 플래그와 항목별 `error` 필드를 통해 부분 실패를 감지하고 보고하세요.
- 범위를 알 수 없고 발견할 수 없으면, 토큰 또는 지출자 대상을 발명하는 대신 범위가 부분적이라고 보고하세요.

## 보고서 형식

사용자 쿼리가 특정 토큰, 지출자, 또는 부분 집합으로 범위가 지정된 경우에도 항상 이 정확한 보고서 구조를 사용하세요. 섹션이 정말 비어 있는 경우에만 생략하되(예: 허용량 없음), 모든 섹션 헤더는 유지하세요. 범위가 지정된 쿼리의 경우 각 섹션 내의 관련 항목만 채우고 요약에 적용된 필터를 기록하세요.

```text
Mantle 포트폴리오 보고서 (Mantle Portfolio Report)
- wallet:
- network:
- chain_id:
- collected_at_utc:

네이티브 잔액 (Native Balance)
- MNT:

토큰 잔액 (Token Balances)
- token: <symbol_or_label>
  address:
  balance_raw:
  decimals:
  balance_normalized:

Aave V3 포지션 (Aave V3 Positions)
- health_factor:
- health_status: no_debt | safe | moderate | at_risk | liquidatable
- total_collateral_usd:
- total_debt_usd:
- available_borrows_usd:
- positions:
  - symbol:
    supplied:
    borrowed:
    collateral_enabled: true | false | null (null = 읽을 수 없음)
- collateral_warnings:  # supplied > 0이지만 collateral_enabled = false인 포지션 표시

V3 LP 포지션 (Agni / Fluxion) (V3 LP Positions)
- total_positions:
- positions:
  - provider:
    pool:
    token0/token1:
    liquidity:
    in_range:

Merchant Moe LB 포지션 (Merchant Moe LB Positions)
- total_positions:
- positions:
  - pair:
    token_x/token_y:
    bins_with_liquidity:

허용량 노출 (Allowance Exposure)
- token:
  spender:
  allowance_raw:
  allowance_normalized:
  risk_level: low | medium | high | critical
  rationale:

요약 (Summary)
- tokens_with_balance:
- aave_health_status:
- v3_lp_positions:
- lb_positions:
- allowances_checked:
- unlimited_or_near_unlimited_count:
- key_risks:
- confidence:
```

## 참조

- `references/rpc-readonly-workflow.md`
- `references/allowance-risk-rules.md`
