# OpenZeppelin MCP 핸드오프

Mantle 프로젝트의 실제 스마트 컨트랙트 작성 또는 구현 안내가 필요할 때는 언제든지 OpenZeppelin MCP를 사용하세요.

## 다음 경우 OpenZeppelin MCP로 라우팅하세요

- Solidity 컨트랙트 스캐폴딩 또는 코드 생성
- OpenZeppelin 상속 및 확장 선택
- `Ownable`, `AccessControl`, 토큰, 프록시, 업그레이드, 거버너, 또는 보안 모듈 사용
- 생성자, 초기화, 또는 스토리지 레이아웃 구현 세부사항
- OpenZeppelin 패턴에 연결된 컨트랙트 리팩터링

## 이 핸드오프 컨텍스트를 제공하세요

- Mantle 환경 (`mainnet` 또는 `testnet`)
- 컨트랙트 목적 및 사용자 흐름
- 자산 모델 및 관련 토큰 표준
- 관리자 및 업그레이드 가능성 요구사항
- 외부 Mantle 의존성 및 알려진 주소
- 이미 알려진 배포 제약 또는 검증 요구사항

## OpenZeppelin MCP에서 예상 반환값

- 권장 컨트랙트 모듈 및 상속 계획
- 구현 노트 또는 코드 안내
- 검토할 보안에 민감한 결정
- 테스트 및 배포에 필요한 입력

## 경계

- OpenZeppelin MCP는 컨트랙트 작성 안내를 처리합니다.
- `mantle-smart-contract-developer`는 Mantle 특화 프레이밍, 준비 검사, `$mantle-smart-contract-deployer`로의 핸드오프를 처리합니다.
