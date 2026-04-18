---
name: mantle-tx-simulator
version: 0.1.10
description: "Mantle 트랜잭션에서 서명 전 시뮬레이션 증거, 상태 변경 검토, 되돌리기(revert) 분석, 또는 실행 전 WYSIWYS 설명이 필요할 때 사용합니다."
---

# Mantle 트랜잭션 시뮬레이터

## 개요

외부 백엔드를 위한 시뮬레이션 핸드오프 패키지를 준비하고 반환된 기술적 변경사항을 서명 또는 실행 전에 사용자가 읽을 수 있는 예상 결과로 변환합니다.

## 워크플로우

1. 시뮬레이션 요청을 정규화합니다:
   - 네트워크
   - from/to/value/data
   - 선택적 번들/멀티콜 컨텍스트
2. `references/simulation-backends.md`를 사용하여 외부 시뮬레이션 백엔드를 선택합니다.
3. 사전 상태를 캡처합니다:
   - 관련 토큰 잔액
   - 허용량 값
   - nonce 및 가스 컨텍스트
4. 외부 시뮬레이션 핸드오프 패키지를 생성합니다:
   - 트랜잭션 페이로드 및 네트워크
   - 필요한 사전 상태 컨텍스트
   - 요청된 출력 (상태/가스/로그/변경사항)
5. 외부 실행 증거가 제공된 후 수집합니다:
   - 성공/되돌리기
   - 가스 추정
   - 로그/이벤트
   - 상태 변경사항
6. `references/wysiwys-template.md`로 기술적 결과를 WYSIWYS 요약으로 변환합니다.
   - **항상 WYSIWYS 섹션을 생성하세요**, 외부 시뮬레이션 결과가 도착하기 전에도.
   - 외부 증거가 아직 없으면, 요청 매개변수를 기반으로 **예비 WYSIWYS**를 생성하고 명확히 레이블을 붙이세요: "예비 — 시뮬레이션으로 아직 검증되지 않음."
   - 시뮬레이션 전 WYSIWYS는 여전히 `what_user_gives`, `what_user_gets_min` ("시뮬레이션 보류 중"으로 표시), `plain_language_outcome`을 채워야 합니다.
7. 시뮬레이션 증거가 없거나, 실패하거나, 신뢰도가 낮으면 `do_not_execute`를 반환합니다.
   - 구체적인 설명과 함께 `do_not_execute_reason`을 설정하세요 (예: "외부 시뮬레이션 증거가 제공되지 않음", "시뮬레이션 되돌려짐", "불완전한 calldata").
   - `do_not_execute`를 반환할 때도 **전체 시뮬레이션 보고서 출력 형식을 계속 생성하세요**.

## 가드레일

- 이 스킬은 읽기 전용입니다: 트랜잭션 시뮬레이션을 직접 실행할 수 없습니다. 온체인 읽기에는 `mantle-cli` 명령을 사용하세요. MCP 서버를 활성화하거나 연결하지 마세요.
- 이 스킬에서 실제 트랜잭션을 절대 브로드캐스트하지 마세요.
- 외부 백엔드 출력이 제공되지 않는 한 시뮬레이션이 실행되었다고 주장하지 마세요.
- 시뮬레이션 추정과 보장된 실행 결과를 구분하세요.
- 토큰 소수점/가격 컨텍스트가 불완전하면 불확실성을 명시적으로 명시하세요.
- 번들 흐름의 경우 각 단계와 순 효과를 설명하세요.
- CLI 도구를 통해 시뮬레이션 실행을 요청받으면, `mantle-cli`에 트랜잭션 시뮬레이션 실행 도구가 없다고 설명하고 외부 핸드오프 지시를 제공하세요.
- **시뮬레이션 증거가 있는지, 요청이 불완전한지, 실행이 거부되었는지에 관계없이 모든 응답에 전체 시뮬레이션 보고서 형식을 항상 출력하세요.** 구조화된 보고서 없이 산문만으로 응답하지 마세요.

## 출력 형식

```text
Mantle 시뮬레이션 보고서 (Mantle Simulation Report)
- execution_mode: external_simulator_handoff
- backend:
- environment:
- simulated_at_utc: <외부 타임스탬프, 있는 경우>
- status: pending_external_simulation | success | revert | inconclusive
- simulation_artifact: <외부 링크/ID, 있는 경우>

상태 변경 요약 (State Diff Summary)
- assets_debited:
- assets_credited:
- approvals_changed:
- contract_state_changes:

실행 추정 (시뮬레이션 — 보장되지 않음) (Execution Estimates)
- gas_estimate:
- estimated_fee_native:
- slippage_or_price_impact_note:

WYSIWYS (서명하면 이렇게 됩니다)
- plain_language_outcome:
- what_user_gives:
- what_user_gets_min:
- do_not_execute_reason: <안전한 경우 비워둠>
- disclaimer: 시뮬레이션 결과는 추정치이며 온체인 실행 결과를 보장하지 않습니다.
```

## 참조

- `references/simulation-backends.md`
- `references/wysiwys-template.md`
