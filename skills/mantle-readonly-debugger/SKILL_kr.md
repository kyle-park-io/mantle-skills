---
name: mantle-readonly-debugger
version: 0.1.10
description: "Mantle 읽기 경로 요청이 실패하거나, 견적이 되돌아가거나, 잔액이 불일치하거나, RPC 동작이 일관성이 없어 구조적 근본 원인 분류가 필요할 때 사용합니다."
---

# Mantle 읽기 전용 디버거

## 개요

사전 실행 실패에 대한 결정론적 트러블슈팅을 실행하고 신뢰할 수 있는 읽기 경로 동작을 복원하기 위한 최소한의 다음 조치를 제공합니다.

## 워크플로우

1. 실패 컨텍스트를 캡처합니다:
   - 메서드/도구 이름
   - 엔드포인트
   - 매개변수
   - 타임스탬프 (UTC)
   - 전체 오류 텍스트/코드
2. `references/error-signature-map.md`로 오류를 분류합니다:
   - RPC 전송/연결
   - 온체인 되돌리기(revert) 또는 호출 예외
   - 견적/유동성 실패
   - 잔액/nonce 불일치
   - 기타/미분류 (오류가 위 카테고리 중 어느 것과도 일치하지 않을 때 사용; `issue_type`에 오류의 성격을 설명)
3. `references/troubleshooting-playbook.md`에서 대상 검사를 실행합니다.
4. 신뢰도 및 즉각적인 다음 단계와 함께 근본 원인 가설을 반환합니다.
5. 해결되지 않으면 제한된 에스컬레이션 체크리스트를 반환합니다.

## 가드레일

- 증거 없이 확정적 근본 원인을 주장하지 마세요.
- `raw_error` 필드에 원래 오류 문자열을 그대로 보존하세요 — 정확한 텍스트를 복사/붙여넣기하고 바꿔 말하거나 요약하지 마세요.
- 사용자가 명시적으로 실행을 요청하지 않는 한 수정을 읽기 전용으로 유지하세요.
- 가장 작고 되돌릴 수 있는 진단 단계를 먼저 선호하세요.

## 출력 형식

**이 템플릿을 정확히 사용하여 응답해야 합니다. 전후에 산문을 추가하지 마세요. 모든 필드를 채우세요. 값을 알 수 없으면 "unknown"이라고 쓰세요.**

```text
Mantle 읽기 전용 디버그 보고서 (Mantle Read-Only Debug Report)
- issue_type:
- observed_at_utc:
- endpoint:
- method_or_tool:
- raw_error:

진단 (Diagnosis)
- likely_causes:
- confidence: low | medium | high
- evidence:

다음 조치 (Next Actions)
- step_1:
- step_2:
- step_3:
- escalation_if_unresolved:
```

## 참조

- `references/error-signature-map.md`
- `references/troubleshooting-playbook.md`
