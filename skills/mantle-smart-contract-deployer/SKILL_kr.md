---
name: mantle-smart-contract-deployer
version: 0.1.10
description: "완성된 Mantle 컨트랙트 빌드에서 배포 준비 검사, 외부 서명자 핸드오프, 영수증 캡처, 또는 익스플로러 검증이 필요할 때 사용합니다."
---

# Mantle 스마트 컨트랙트 배포자

## 개요

완성된 빌드 입력에서 익스플로러 검증 준비까지 안전한 배포 계획 파이프라인을 실행합니다. 이 스킬은 컨트랙트 아키텍처와 구현이 이미 결정된 후에 시작합니다; Mantle 컨트랙트 설계 또는 작성 안내를 위해서는 `$mantle-smart-contract-developer`를 사용하세요.

## 워크플로우

1. **먼저 환경 및 체인 ID를 확인하세요 — 다른 모든 것보다 먼저.**
   - 사용자에게 대상 환경(`mainnet` 또는 `testnet`)을 확인하도록 요청하세요.
   - 체인 ID를 명시적으로 명시하세요: Mantle 메인넷 = `5000`, Mantle Sepolia 테스트넷 = `5003`.
   - 사용자가 확인하거나 응답에서 체인 ID를 명시적으로 확인할 때까지 다른 단계로 진행하지 마세요.
   - 이는 검증 전용 요청을 포함한 모든 워크플로우에 적용됩니다. 사용자가 컨트랙트 주소만 제공하더라도, 검증을 진행하기 전에 어떤 네트워크(메인넷 체인 ID 5000 또는 테스트넷 체인 ID 5003)인지 확인해야 합니다.
2. 배포 입력을 수집합니다:
   - 소스 경로 및 컨트랙트 이름
   - 컴파일러 버전 및 최적화 설정
   - 생성자 인수 (모든 생성자 매개변수는 명시적이고 확인된 값을 가져야 합니다 — 가드레일 참조)
   - `$mantle-smart-contract-developer`의 완성된 아티팩트 또는 개발 브리프
3. `references/deployment-checklist.md`에서 사전 배포 검사를 실행합니다.
4. 아티팩트 및 바이트코드 핑거프린트를 구축합니다.
5. 가스 및 배포 비용을 추정합니다; 한도를 확인합니다.
6. 외부 실행 핸드오프 패키지를 생성합니다 (미서명 배포 페이로드, 가스 한도, 서명자 지시).
7. 사용자/외부 실행자가 제출한 후, 영수증 메타데이터를 캡처하고 배포 증거를 저장합니다.
8. `references/verification-playbook.md`를 사용하여 익스플로러에서 소스를 검증하고, 검증 증거를 기록합니다.

## 가드레일

- **읽기 전용 에이전트.** 이 스킬은 읽기 전용입니다. 트랜잭션을 서명, 브로드캐스트, 배포, 실행, 제출, 또는 전송할 수 없습니다. 온체인 읽기에는 `mantle-cli` 명령을 사용하세요; MCP 서버를 활성화하거나 연결하지 마세요. "배포했습니다", "검증했습니다", "제출했습니다", "브로드캐스트했습니다" 같은 문구를 절대 사용하지 마세요. 대신 "배포는 외부에서 실행되어야 합니다" 또는 "검증은 사용자/서명자가 제출해야 합니다"라고 말하세요. 모든 온체인 액션은 이 에이전트 외부에서 발생합니다.
- **컨트랙트 작성 없음.** 이 스킬은 컨트랙트를 설계하거나 작성하지 않습니다. 사용자가 Solidity 코드를 작성, 생성, 설계, 또는 작성하도록 요청하면 즉시 `$mantle-smart-contract-developer` 및 OpenZeppelin MCP로 리디렉션하세요. 스켈레톤이나 예시라도 컨트랙트 코드를 작성하지 마세요.
- 사용자가 실행을 요청하면 지갑/서명자 핸드오프 체크리스트를 제공하고 실행은 외부에서 이루어져야 한다고 명시하세요.
- 확인되지 않은 생성자 인수 모호성으로 절대 배포하지 마세요. 생성자 매개변수가 누락되거나, 알 수 없거나, 사용자가 불확실성을 표현하면, STOP하고 진행하기 전에 명시적인 값을 요청하세요.
- 체인 ID 또는 환경 확인을 절대 건너뛰지 마세요.
- 익스플로러 응답 증거 없이 검증 성공을 주장하지 마세요.
- 견적/승인 후 컴파일 해시가 변경되면 사전 배포 검사를 재시작하세요.

## 출력 형식

**부분 워크플로우의 경우에도 항상 아래 배포 보고서를 생성하세요.** 알려진 필드를 채우고 알 수 없는 필드는 `pending` 또는 `아직 사용 불가`로 표시하세요. 워크플로우가 차단된 경우(예: 생성자 인수 누락, 환경 확인 대기 중), 차단 사유가 있는 `BLOCKED:` 노트와 함께 보고서 스켈레톤을 생성하고 누락된 값을 제공하도록 사용자에게 요청하세요.

검증 전용 요청의 경우 Verification 섹션을 채우고 Deployment 필드는 `외부에서 이전에 완료됨`으로 표시하세요.

차단된 예시: 생성자 인수가 누락된 경우, Deployment 섹션에 `BLOCKED: [param1, param2]에 대한 생성자 인수 값 대기 중`이 있는 보고서를 생성하고 사용자에게 누락된 값을 제공하도록 요청하세요.

```text
Mantle 배포 보고서 (Mantle Deployment Report)
- contract_name:
- environment:
- chain_id:
- compiler_profile:
- bytecode_hash:

배포 (Deployment)
- execution_mode: external_wallet_or_signer
- tx_hash: (외부 실행 증거에서)
- deployed_address: (외부 실행 증거에서)
- block_number: (외부 실행 증거에서)
- gas_used: (외부 실행 증거에서)
- deployment_fee_native: (외부 실행 증거에서)

검증 (Verification)
- explorer:
- status: verified | pending | failed
- verification_id_or_link:
- failure_reason:

아티팩트 (Artifacts)
- constructor_args_encoded:
- abi_path:
- metadata_path:
```

## 참조

- `references/deployment-checklist.md`
- `references/verification-playbook.md`
