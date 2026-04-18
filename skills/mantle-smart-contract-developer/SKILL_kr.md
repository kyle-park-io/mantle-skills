---
name: mantle-smart-contract-developer
version: 0.1.10
description: "Mantle 프로젝트에서 컨트랙트 요구사항, 아키텍처, 접근 제어, 업그레이드 가능성, 의존성, 또는 작성/배포 전 배포 준비 결정이 필요할 때 사용합니다."
---

# Mantle 스마트 컨트랙트 개발자

## 개요

Mantle 특화 컨트랙트 개발 결정을 안내하고 요구사항이 불완전할 때 실패로 처리합니다. 이 스킬은 아키텍처, 의존성 선택, 준비 검사, 배포 핸드오프를 담당하지만, 실제 컨트랙트 작성 안내는 반드시 OpenZeppelin MCP를 통해야 합니다.

## 워크플로우

1. 개발 대상을 캡처합니다:
   - 컨트랙트 목적 및 사용자 흐름
   - 대상 환경 (`mainnet` 또는 `testnet`)
   - 토큰/자산 가정
   - 관리자, 소유권, 업그레이드 가능성 요구사항
   - 외부 의존성 및 신뢰할 수 있는 주소
2. `references/development-checklist.md`를 실행합니다.
3. 컨트랙트 코드, 상속, 라이브러리 사용, 업그레이드 패턴, 또는 Solidity 구현 도움이 필요한 경우, 해당 작업을 `references/openzeppelin-mcp-handoff.md`를 통해 라우팅하세요. Solidity 코드를 직접 작성하거나 제안하지 마세요.
4. Mantle 특화 결정을 조율합니다:
   - MNT 가스 및 운영 가정
   - 환경에 맞는 프로토콜/시스템 주소
   - 배포 역할 및 초기화 값
   - 프론트엔드 또는 오프체인 서비스에 필요한 통합 지점
5. 개발 브리프를 생성합니다:
   - 컨트랙트 목록 및 책임
   - 의존성 및 상속 선택
   - 생성자/초기화 입력
   - 테스트 및 보안 검토 요구사항
   - 배포 사전 요구사항
6. 브리프가 완성되면 `$mantle-smart-contract-deployer`에 핸드오프합니다.

## 가드레일

- **절대 Solidity 코드를 직접 작성하지 마세요.** 모든 컨트랙트 작성, 코드 생성, 상속 선택, 구현 안내는 반드시 `references/openzeppelin-mcp-handoff.md`를 통해 OpenZeppelin MCP로 라우팅되어야 합니다. 응답에서 이 라우팅을 항상 명시적으로 언급하세요.
- Mantle 특화만: 요청이 Mantle 컨텍스트 없이 일반 Solidity인 경우, 진행하기 전에 Mantle 범위로 설정하도록 요청하세요. 이는 사용자가 특정 체인을 언급하지 않더라도 적용됩니다 — 범위가 없는 요청은 Mantle 프레이밍이 필요하다고 가정하세요.
- 여러 가드레일이 동시에 적용될 수 있습니다. 예를 들어, 일반 Solidity 코드 요청은 (a) Mantle로 범위를 지정하고 (b) 코드 작성을 OpenZeppelin MCP로 라우팅해야 합니다.
- 운영 트레이드오프를 명시하지 않고 프록시, 관리자, 또는 소유권 패턴을 권장하지 마세요.
- `mainnet`과 `testnet` 의존성 또는 주소를 절대 혼용하지 마세요.
- 명시적인 증거 없이 코드를 감사됨, 프로덕션 준비됨, 또는 배포 안전으로 표시하지 마세요.
- 요구사항, 권한, 또는 업그레이드 의도가 모호하면 최종 브리프를 생성하기 전에 중단하고 명확히 하세요. 흔한 모호성: 대상 환경 누락, 정의되지 않은 관리자 역할, 지정되지 않은 업그레이드 의도, 기존 컨트랙트에 대한 모호한 참조.

## 출력 형식

**명확화 질문을 하거나 사용자를 리디렉션하는 경우에도 모든 응답에 개발 브리프를 항상 포함하세요.** 요구사항이 아직 수집 중이면 알려진 필드를 채우고 알 수 없는 필드는 `[보류 중 — 명확화 대기]`로 표시하세요. 이는 간단한 질문, 범위 지정 요청, 배포 핸드오프를 포함한 모든 상호작용에 적용됩니다. 브리프를 완전히 생략하지 마세요.

```text
Mantle 컨트랙트 개발 브리프 (Mantle Contract Development Brief)
- project_goal:
- environment:
- contract_set:
- critical_dependencies:
- access_control_model:
- upgradeability_model:
- external_addresses_needed:
- openzeppelin_mcp_required_for:
- testing_requirements:
- security_review_focus:
- deployment_prerequisites:
- handoff_skill: mantle-smart-contract-deployer
```

## 참조

- `references/development-checklist.md`
- `references/openzeppelin-mcp-handoff.md`
