# Mantle 네트워크 기초

Mantle 온보딩, 차이점, 개발자 핸드오프 질문에 답할 때 사실적 근거를 위해 이 파일을 사용하세요. 실행 워크플로우가 아닌 참조/온보딩 스킬을 지원합니다.

## 소스 및 신선도

- 기본 소스: https://docs.mantle.xyz/network/for-developers/quick-access
- 추가 소스:
  - https://docs.mantle.xyz/network/system-information/architecture
  - https://docs.mantle.xyz/network/for-developers/resources-and-tooling/node-endpoints-and-providers
- 스냅샷 검증일: **2026년 3월 8일**
- 사용자가 "최신" 값이나 아키텍처 상태를 묻는 경우, 공식 문서를 다시 확인한 후 답변하세요.

## 이 참조 사용 방법

- `핵심 모델`, `Mantle 특유의 차이점`, `개발자 힌트`, `RPC 신뢰성 안내`를 안정적인 개념 레이어로 취급하세요.
- `네트워크 세부사항`, `온보딩 도구`, 컨트랙트 링크, 아키텍처 노트를 **2026년 3월 8일**에 검증된 날짜가 있는 스냅샷으로 취급하세요.
- "최신", "현재", 또는 롤아웃 상태 세부사항을 묻는 질문에는 이 파일로 답을 구성하고 결정적으로 답하기 전에 실시간 확인하세요.

## 핵심 모델

- Mantle은 이더리움 정렬 레이어 2 실행 네트워크입니다.
- 사용자는 L2에서 트랜잭션을 실행합니다.
- 정산/보안 보증은 이더리움 L1에 고정됩니다.
- Mantle은 EVM 호환이며 표준 이더리움 툴링을 사용합니다.
- Mantle의 가스는 `MNT`로 지불됩니다.

## Mantle 특유의 차이점

- `MNT`가 가스 토큰이므로 개발자는 `ETH` 충전된 지갑이 Mantle에서 트랜잭션할 수 있다고 가정하면 안 됩니다.
- Mantle은 이더리움 정렬이지만, 빠른 L2 트랜잭션 포함은 가장 강력한 L1 기반 정산/최종성과 동일하지 않습니다.
- Mantle 특유의 온보딩 상수가 실제로 중요합니다:
  - 메인넷 `chainId`: `5000`
  - Sepolia 테스트넷 `chainId`: `5003`
  - 익스플로러 도메인과 브릿지 링크는 Mantle 특유
- **2026년 3월 8일**에 검증된 Mantle 아키텍처 페이지 스냅샷에 따르면, Mantle v2 Skadi는 실행 레이어, ZK 유효성 증명 모듈, 블롭을 통한 이더리움 데이터 가용성을 사용하는 것으로 설명됩니다.
- 아키텍처 롤아웃 세부사항은 발전할 수 있으므로 수수료 동작, 처리량, "최신 아키텍처" 질문은 실시간 확인 항목으로 취급하세요.

## Mantle의 핵심 토큰

**중요: WETH (브릿지된 ETH)는 Mantle에 존재합니다.** "MNT가 가스 토큰"을 "ETH가 Mantle에 없다"와 혼동하지 마세요. ETH는 L1에서 브릿지되며 가장 활발하게 거래되는 자산 중 하나입니다.

| 토큰 | 주소 | 노트 |
|-------|---------|-------|
| MNT | 네이티브 (컨트랙트 없음) | 가스 토큰, 네이티브 통화 |
| WMNT | `0x78c1b0C915c4FAA5FffA6CAbf0219DA63d7f4cb8` | 래핑된 MNT (네이티브 MNT의 ERC-20 래퍼) |
| WETH | `0xdEAddEaDdeadDEadDEADDEAddEADDEAddead1111` | L1에서 브릿지된 ETH — **존재하며 깊은 유동성 보유** |
| mETH | `0xcDA86A272531e8640cD7F1a92c01839911B90bb0` | Mantle 스테이킹된 ETH (유동 스테이킹 파생상품) |
| cmETH | `0xE6829d9a7eE3040e1276Fa75293Bde931859e8fA` | 리스테이킹된 mETH |
| USDC | `0x09Bc4E0D864854c6aFB6eB9A9cdF58aC190D0dF9` | USD Coin (브릿지됨) |
| USDT | `0x201EBa5CC46D216Ce6DC03F6a759e8E766e956aE` | 브릿지된 Tether — DEX에서 활성 |
| USDT0 | `0x779Ded0c9e1022225f8E0630b35a9b54bE713736` | LayerZero OFT Tether — DEX 및 Aave V3에서 활성 |
| USDe | `0x5d3a1Ff2b6BAb83b63cd9AD0787074081a52ef34` | Ethena USDe |

> **USDT vs USDT0:** Mantle에는 두 가지 공식 USDT 변형이 있습니다. 둘 다 DEX에서 깊은 유동성이 있습니다 (Merchant Moe, Fluxion). USDT0만 Aave V3에서 지원됩니다. Merchant Moe의 USDT/USDT0 풀 (bin_step=1)로 직접 변환이 가능합니다.

**피해야 할 흔한 실수:** WETH가 Mantle에 없다고 주장하는 것. 있습니다 — `0xdEAD...1111`에 브릿지된 ETH이며, ~125K ETH 총 공급량과 모든 주요 DEX (Agni, Fluxion, Merchant Moe)에서 풀이 있습니다.

## 네트워크 세부사항 (날짜가 있는 Quick Access 스냅샷)

### 메인넷

- RPC URL: `https://rpc.mantle.xyz`
- WebSocket URL: `wss://rpc.mantle.xyz`
- 체인 ID: `5000`
- 토큰 심볼: `MNT`
- 익스플로러: `https://mantlescan.xyz/`

### 테스트넷 (Sepolia)

- RPC URL: `https://rpc.sepolia.mantle.xyz`
- WebSocket URL: `N/A` (Quick Access 기준)
- 체인 ID: `5003`
- 토큰 심볼: `MNT`
- 익스플로러: `https://sepolia.mantlescan.xyz/`

## 온보딩 도구 (날짜가 있는 Quick Access 스냅샷)

### 메인넷

- 브릿지: `https://app.mantle.xyz/bridge`
- 권장 Solidity 컴파일러: `v0.8.23 이하`
- 래핑된 MNT: `0x78c1b0C915c4FAA5FffA6CAbf0219DA63d7f4cb8`

### 테스트넷 (Sepolia)

- 수도꼭지(Faucet): `https://faucet.sepolia.mantle.xyz/`
- 서드파티 수도꼭지:
  - `https://faucet.quicknode.com/mantle/sepolia`
  - `https://thirdweb.com/mantle-sepolia-testnet/faucet`
- 브릿지: `https://app.mantle.xyz/bridge?network=sepolia`
- 권장 Solidity 컴파일러: `v0.8.23 이하`
- 래핑된 MNT: `0x19f5557E23e9914A18239990f6C70D68FDF0deD5`
- 참고: Mantle 문서에 따르면 Sepolia MNT는 수도꼭지에서 직접 요청할 수 있습니다 (한도 있음).

## 개발자 힌트

- Mantle에서 트랜잭션을 시도하기 전에 개발자 및 테스트 지갑에 `ETH`가 아닌 `MNT`를 충전하세요.
- 일반 "커스텀 EVM" 가정 대신 위의 공식 체인 설정을 사용하세요.
- Mantle 문서는 현재 Solidity `v0.8.23 이하`를 권장합니다.
- 공식 공개 RPC 엔드포인트는 온보딩 및 가벼운 사용에 적합하지만, 프로덕션 또는 고빈도 워크로드에는 전용 공급자를 사용해야 합니다.
- 정확한 컨트랙트 주소, 토큰 매핑, 브릿지 관련 메타데이터는 메모리 대신 아래의 공식 주소 페이지 및 토큰 목록 링크를 선호하세요.
- UX 문제를 디버깅할 때 두 가지를 모두 설명하세요:
  - `포함(inclusion)`: 트랜잭션이 L2 블록에 보임
  - `L1 기반 정산 최종성`: L1 측 조건이 충족되면 가장 강력한 정산 보증

## 컨트랙트 및 토큰 진실 소스

- L1 시스템 컨트랙트: `https://docs.mantle.xyz/network/system-information/on-chain-system/key-l1-contract-address`
- L2 시스템 컨트랙트: `https://docs.mantle.xyz/network/system-information/off-chain-system/key-l2-contract-address`
- 토큰 목록 진실 소스: `https://token-list.mantle.xyz`
- 브릿지 참조: `https://bridge.mantle.xyz`
- 토큰 목록 PR 저장소 (토큰 추가): `https://github.com/mantlenetworkio/mantle-token-lists`

## RPC 신뢰성 안내

- Mantle 문서에 따르면 공식 RPC 엔드포인트는 안정성을 위해 속도 제한이 있습니다.
- 고빈도 또는 프로덕션 워크로드에는 전용 공급자 엔드포인트를 선호하세요.
- 공급자 디렉토리: `https://docs.mantle.xyz/network/for-developers/resources-and-tooling/node-endpoints-and-providers`

## 이 스킬의 응답 규칙

- 이 파일에서 값을 인용할 때 절대 날짜를 사용하세요.
- 처리량, 수수료 수준, 에코시스템 수, 지연/최종성 창을 변동성 있는 것으로 취급하세요.
- 구분하세요:
  - `포함(inclusion)`: 트랜잭션이 L2 블록에 나타남.
  - `L1 기반 정산 최종성`: L1 조건이 충족되면 가장 강력한 정산 보증.
- 실행 컨텍스트에서 정확한 컨트랙트 주소 조회는 다음을 교차 확인하세요:
  - 위의 Mantle 컨트랙트 주소 페이지, 또는
  - 런타임에서 사용 가능한 경우 `$mantle-address-registry-navigator`.

## 차이점 체크리스트

"Mantle이 다른 점은 무엇인가"라는 질문을 받으면 다음을 다루세요:

1. 가스 토큰 및 지갑 충전 기대사항
2. 포함(inclusion) 대 L1 기반 정산/최종성(finality)
3. 현재 아키텍처 스냅샷 대 실시간 확인 항목
4. 체인 ID, 브릿지, 익스플로러, 수도꼭지 같은 온보딩 상수
5. RPC 속도 제한 및 공급자 선택 같은 운영 안내
