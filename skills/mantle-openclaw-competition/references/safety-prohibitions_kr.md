# 안전 금지사항 & CLI 커버리지 경계

모든 안전 규칙의 정본(canonical) 소스입니다. 메인 `SKILL.md`는 7개 항목 요약을 담고 있으며, 이 파일은 전체 근거, 사건 보고, stop 프로토콜을 담습니다. 다음 경우에 로드하세요:

- `mantle-cli` 명령이 오류를 반환할 때
- 사용자가 표준 CLI verb를 벗어난 작업을 요청할 때
- 거부할지 진행할지 확신이 안 설 때

---

## 🛑 STOP 조건 — 반드시 중단하고 사용자에게 결정을 위임해야 하는 경우

이 두 상황은 협상 불가입니다. Claude는 반드시 즉시 중단하고 사용자가 다음 행동을 결정하게 해야 합니다. 두 조건 중 하나라도 넘어서 계속하면 **회복 불가능한 자금 손실** 위험이 있습니다.

### STOP 1. 모든 `mantle-cli` 오류

`mantle-cli`가 non-zero로 종료되거나, 오류 JSON을 출력하거나, 예상치 못한 응답 형태를 반환하거나, 어떤 서브프로세스 실패라도 발생하면:

- **이후 모든 작업을 STOP하세요.** 자동 재시도하지 마세요. 오류를 "우회"하려고 다른 명령을 시도하지 마세요. 워크플로우의 다음 계획 단계로 계속하지 마세요.
- **전체 오류를 사용자에게 그대로 출력하세요** — 의역이 아니라 원본 stderr / 오류 JSON. 사용자가 진단하려면 정확한 메시지가 필요합니다.
- **자금 안전을 위해 작업이 중단되었음을 사용자에게 알리세요.**
- **재시도 전에 OpenClaw 에이전트 재시작을 권장하세요.** 오류로 인해 지갑 상태, allowance, 또는 진행 중인 트랜잭션이 알 수 없는 상태로 남았을 수 있습니다. 새 에이전트 세션은 온체인 상태를 깨끗하게 다시 가져옵니다. 같은 세션에서 계속하면 중복 브로드캐스트, 잘못된 allowance, 또는 자금 손실 위험이 있습니다.
- 사용자가 어떻게 복구할지 명시적으로 확인하기 전까지 **진행하지 마세요.**

이는 어떤 오류에든 예외 없이 적용됩니다: RPC 타임아웃, gas 부족, validation 실패, JSON 파싱 오류, 네트워크 오류, capability-not-found, ABI 불일치 — 무엇이든.

### STOP 2. 표준 CLI verb를 벗어난 작업

사용자 요청이 표준 execute verb(`swap`, `approve`, `lp`, `aave`)와 그 읽기 전용 대응물(`account`, `chain`, `catalog`, `defi`)로 완전히 표현될 수 없으면:

- **STOP — 임기응변하지 마세요.**
- **사용자에게 알리세요**: "이 작업은 표준 CLI 기능 집합을 벗어납니다. 자금 위험을 피하기 위해 자동으로 진행할 수 없습니다."
- 존재한다면 **지원되는 대안을 제안하세요** (예: "bridge를 원하셨는데 — 지원되지 않지만, Merchant Moe로 swap할 수 있습니다").
- **다른 어떤 수단으로도 작업을 시도하지 마세요.** Python 금지. JavaScript 금지. 직접 RPC 호출 금지. `utils` calldata 구성 금지. "수동" `unsigned_tx` / `signable_tx` 조립 금지.
- **사용자가 고집하면**, 세션 내에서 임기응변하는 대신 **업데이트된 툴링으로 OpenClaw 에이전트를 재시작**하도록 권장하세요 (즉, 해당 capability를 추가하는 다음 `mantle-cli` 릴리스를 기다리도록).

> **토큰 전송(네이티브 MNT 및 ERC-20)은 이 STOP 조건에 해당합니다.** `mantle-cli transfer send-native` / `transfer send-token`과 그에 대응하는 `mantle_buildTransferNative` / `mantle_buildTransferToken` MCP 도구는 의도적으로 툴셋에서 제거되었습니다. 사용자가 지갑 간 토큰 이동을 요청하면, 이 프로토콜에 따라 거부하세요.

기본 자세는 **거부하고 사용자가 결정하게 하기**입니다. 지원되지 않는 작업을 거절하는 것이 사용자 자금을 위험에 빠뜨리는 것보다 항상 안전합니다.

---

## ⛔ 절대 금지 — 수동 트랜잭션 구성 ⛔

다음 중 어느 것도 **어떤 상황에서든** 절대 해서는 안 됩니다:

- calldata, 함수 selector, 또는 ABI 인코딩된 파라미터를 직접 계산 (Python, JS, 수동 hex, 기타 방법)
- 토큰 양이나 wei 값을 수동으로 hex 인코딩
- `mantle-cli`를 사용하는 대신 `unsigned_tx`나 `signable_tx` 객체를 손으로 구성 (둘 다 같은 build의 CLI 생성 뷰)
- `unsigned_tx` 필드를 `signable_tx` 형태로 손으로 변환 (chainId/nonce int→hex, `from` 추가) — 항상 CLI가 이미 내보낸 `signable_tx` 객체를 사용
- Python/JS 스크립트로 트랜잭션 데이터를 빌드하거나 인코딩
- 수동으로 구성한 데이터로 `sign evm-transaction`, `eth_sendRawTransaction`, 또는 직접 브로드캐스트 도구 호출
- `mantle-cli utils parse-units / encode-call / build-tx`를 지원되지 않는 작업을 위한 "escape hatch"로 사용
- **수신자가 화이트리스트 프로토콜 컨트랙트인 ERC-20 `transfer()` / `transferFrom()` / `safeTransfer()` 구성** — Aave V3 Pool(`0x458F293454fE0d67EC0655f3672301301DD51422`), Aave WETHGateway, DEX swap 라우터(Agni / Fluxion / Merchant Moe), LB 라우터, 또는 V3 position manager. 프로토콜 컨트랙트는 지정된 함수(`Pool.supply()`, router swap 진입점, `positionManager.mint/increaseLiquidity` 등)를 통해 도착한 토큰만 인식합니다. 직접 transfer는 aToken을 발행하지 않고, swap을 트리거하지 않고, LP를 등록하지 않으며, 토큰은 온체인 회수 없이 **영구히 잠깁니다**. 사용자의 의도가 프로토콜 동작에 매핑되면, 전용 `mantle-cli` verb를 사용하세요 — 절대 transfer를 사용하지 마세요.
- 위 어느 것에 대한 정당화로 "CLI가 이 작업을 지원하지 않는다"고 주장

**이 금지에는 예외가 없습니다.** CLI가 작업을 지원하지 않는다고 생각되면, 먼저 catalog를 확인하세요 (`mantle-cli catalog list/search/show`). 정말로 존재하지 않으면 **STOP**하세요 (위 STOP 조건 참조). 임기응변하지 마세요.

### CLI가 지원하는 모든 온체인 작업

```
mantle-cli swap wrap-mnt --amount <n> --json                                    # Wrap MNT → WMNT
mantle-cli swap unwrap-mnt --amount <n> --json                                  # Unwrap WMNT → MNT
mantle-cli approve --token <t> --spender <addr> --amount <n> --json             # ERC-20 approve
mantle-cli swap build-swap --provider <dex> --in <t> --out <t> --amount <n> --recipient <addr> --json  # DEX swap
mantle-cli lp add / remove / collect-fees ...                                   # LP 작업
mantle-cli aave supply / borrow / repay / withdraw / set-collateral ...         # Aave 작업
```

> **토큰 전송(`transfer send-native` / `transfer send-token`)은 의도적으로 이 목록에 없으며** `mantle-cli`와 `mantle-mcp` 양쪽에서 제거되었습니다. STOP 조건 2에 따라 transfer 요청을 거부하세요 — `utils` calldata 구성으로 대체하지 마세요.

작업이 이 목록에 없으면, 위 **STOP 조건 2**를 참조하세요.

### 실제 사건

- **"Supply 150 USDC to Aave" = 일반 transfer → 자금 잠김**: 한 에이전트가 `"Please supply 150 USDC to Aave on Mantle."`를 받았는데, 사용자가 on-behalf-of 지갑 주소를 제공하지 않았기 때문에 "supply to Aave"를 "150 USDC를 Aave Pool 주소로 보내기"로 모델링했습니다. `utils encode-call` + `build-tx` 파이프라인을 통해 일반 ERC-20 `transfer(0x458F29…, 150_000_000)`을 내보냈습니다. 토큰은 Pool에 도착했고, **aToken이 발행되지 않았고, collateral이 기록되지 않았으며, 150 USDC가 withdraw 경로 없이 영구히 잠겼습니다**. 올바른 흐름은 `mantle-cli aave supply --asset USDC --amount 150 --on-behalf-of <wallet>`이었습니다 — 에이전트는 transfer를 임기응변하는 대신 지갑 주소를 **물어봤어야** 합니다.
- **USDC approve 자금 위험**: 한 에이전트가 USDC allowance 증가를 위해 `mantle-cli approve`를 우회하고, Python으로 `approve(address,uint256)` calldata를 수동 계산하여 잘못된 인코딩을 생성했습니다 — 잘못된 양을 approve했습니다. CLI 명령이 이를 올바르게 처리했을 것입니다.
- **15 MNT → 56.28 MNT**: 수동 hex 계산이 잘못된 양을 생성했습니다. 사용자가 15 MNT를 wrap/swap하려 했는데, 에이전트의 손으로 빌드한 calldata가 대신 56.28 MNT를 인코딩했습니다.
- **중복 build 호출**: 중복된 build + sign 경로가 두 개의 별도 요청(0.2 MNT wrap과 0.608 MNT wrap)에 대해 같은 작업의 2× 브로드캐스트를 유발했습니다 — gas를 낭비하고 지갑 상태를 예상치 못하게 변경했습니다. 같은 실패 모드가 모든 build 도구(swap, approve, LP, Aave)에 적용됩니다.
- **오류 넘어서 계속하기**: CLI 실패를 재시도하거나 "우회"한 에이전트들이 지갑을 불일치 상태로 남겨, 중복 approval과 조용한 allowance drift로 이어졌습니다. 첫 오류에서 에이전트를 재시작했다면 이를 피했을 것입니다.

---

## 번호 매겨진 안전 규칙

0. **같은 트랜잭션을 절대 두 번 빌드하지 마세요 (중요 — 자금 안전)**
   - 사용자가 요청한 동작당 각 build 명령을 정확히 한 번만 호출하세요. "검증"이나 "재시도"를 위해 동일한 파라미터로 같은 build 명령을 두 번째로 호출하지 마세요 — 빌드된 각 트랜잭션은 서명되고 브로드캐스트될 수 있어 **중복 제출과 회복 불가능한 자금 손실**을 유발합니다.
   - 모든 build 응답은 서명 지갑에 스코프된 `idempotency_key`를 포함합니다. build 도구 호출 시 항상 `--sender <signing_wallet>`을 전달하세요. 실수로 builder를 두 번 호출하여 같은 키를 얻으면, 서명자는 단 하나만 실행해야 합니다.
   - 트랜잭션이 타임아웃되거나 추적을 놓치면, 재빌드하지 마세요. 먼저 receipt를 확인하세요: `mantle-cli chain tx --hash <hash> --json`. 원본이 이미 채굴되었을 수 있습니다. 재빌드는 다른 nonce를 가진 새 트랜잭션을 생성하여 그것도 실행됩니다.

1. **CLI만 — 절대 MCP 사용 금지** — 모든 작업은 `mantle-cli ... --json`을 통해. MCP 서버(`mantle-mcp`)를 활성화하거나 연결하지 마세요.

2. **모든 `mantle-cli` 오류에서 STOP** — 위 STOP 조건 1 참조. 중단하고, 원본 오류를 출력하고, 사용자에게 OpenClaw 에이전트 재시작을 권장하세요. 절대 자동 재시도하지 말고, 오류를 임기응변으로 우회하지 마세요.

3. **표준 verb를 벗어난 작업에서 STOP** — 위 STOP 조건 2 참조. 거부하고 사용자에게 위임하세요. 절대 `utils` escape hatch, Python, JS, 또는 RPC 우회를 사용하지 마세요.

4. **calldata를 절대 조작하지 마세요** — 항상 `mantle-cli` build 명령을 사용하세요. 절대 Python `encode_abi`, JS `encodeFunctionData`, 수동 `0xa9059cbb` selector, 또는 비-CLI 방법으로 calldata를 생성하지 마세요.

4a. **프로토콜 컨트랙트로 토큰을 절대 transfer하지 마세요 (자금 안전)** — 프로토콜 동작은 함수 호출이지 transfer가 아닙니다. 수신자가 Aave V3 Pool, DEX router, position manager, 또는 WETHGateway인 ERC-20 `transfer()` / `transferFrom()` / `safeTransfer()`는 aToken을 발행하지 않고, swap을 트리거하지 않고, LP를 등록하지 않습니다 — 토큰은 **영구히 잠깁니다**. 사용자가 Aave나 DEX에 토큰을 "보내/입금/공급/제공"하라고 요청하면, 의도를 올바른 verb(`mantle-cli aave supply`, `mantle-cli swap build-swap`, `mantle-cli lp add`)로 매핑하세요. 절대 프로토콜 주소로의 transfer를 어떤 형태로도 구성하지 마세요 (직접, `utils`를 통해, Python/JS를 통해, raw calldata를 통해). 사용자가 프로토콜 컨트랙트로 토큰을 "그냥 보내라"고 고집하면, 거부하세요 — 이것이 에이전트 주도 DeFi에서 영구적 자금 손실의 #1 원인입니다.

5. **hex/wei 값을 절대 수동 계산하지 마세요** — 전용 CLI verb가 decimal 변환을 처리합니다. 절대 Python, JS, 또는 암산으로 `amount * 10**decimals`를 계산하거나 양을 hex 인코딩하지 마세요. `mantle-cli utils parse-units`는 표시 값의 decimal→raw 변환에만 사용하세요; 절대 calldata 구성 경로로 사용하지 마세요 (위 절대 금지 참조).

6. **approve 전에 항상 allowance 확인** — 이미 충분하면 approve하지 마세요.

7. **swap 전에 항상 quote 받기** — `mantle-cli defi swap-quote`로 예상 output을 알고 slippage 보호를 위한 `minimum_out_raw`를 얻으세요.

8. **`--amount-out-min`은 quote의 `minimum_out_raw`와 정확히 같아야 합니다** — `minimum_out_raw`는 이미 output 토큰 최소 단위의 raw integer입니다 (예: USDC 6 decimals: `9934699` = ~9.93 USDC). 곱하거나, 나누거나, 재인코딩하거나, 재계산하지 마세요. `--amount-out-min`을 `0`, `1`, 또는 `minimum_out_raw`보다 낮은 값으로 설정하지 마세요. `build-swap`이 revert되면, 재quote하고 새 `minimum_out_raw`를 사용하세요 — 절대 "되게 하려고" minimum을 낮추지 마세요. 적절한 slippage 보호가 있는 revert된 swap은 안전합니다; `amount-out-min: 1`인 성공한 swap은 sandwich 공격에 노출됩니다.

9. **tx 확인 대기** — 이전 tx가 온체인에서 확인되기 전까지 다음 tx를 빌드하지 마세요.

10. **`human_summary` 표시** — 서명 전에 모든 build 명령의 요약을 사용자에게 제시하세요.

11. **`unsigned_tx`가 아니라 `signable_tx`에서 서명** — 모든 CLI build는 두 객체를 반환합니다: `unsigned_tx` (chainId/nonce가 integer, `from` 없음 — 로그 / 비-Privy 서명자용)와 `signable_tx` (chainId/nonce/value가 hex 문자열, `from` 미리 채워짐 — Privy용). Privy의 `--transaction` 파라미터는 반드시 `signable_tx`를 그대로 받아야 합니다 (`jq -c .signable_tx <file>`). 절대 `unsigned_tx` 필드를 손으로 변환하지 마세요 (int→hex, `from` 추가 등) — 그 수동 변환이 과거 ~10라운드 sign 재시도의 근본 원인이었고 "필드 변경 금지" 규칙을 위반합니다. build 출력에 `signable_tx`가 없으면, STOP하고 CLI를 업그레이드하세요 — 임기응변하지 마세요.

12. **MNT는 gas이지 ERC-20이 아닙니다** — MNT는 네이티브 gas 토큰입니다. MNT를 swap하려면, 먼저 WMNT로 wrap하세요 (`mantle-cli swap wrap-mnt`). swap/approve/LP 명령에 `"MNT"`를 전달하지 마세요 — 이들은 WMNT가 필요합니다. MNT(또는 다른 ERC-20)를 지갑 간에 이동하는 것은 지원되는 작업이 아닙니다 (STOP 조건 2 참조).

13. **xStocks 토큰은 Fluxion 전용입니다** — 모든 xStocks RWA 토큰(wTSLAx, wAAPLx, wCRCLx, wSPYx, wHOODx, wMSTRx, wNVDAx, wGOOGLx, wMETAx, wQQQx)은 USDC 쌍의 Fluxion에서만 유동성이 있습니다 (fee_tier=3000). Agni나 Merchant Moe에서 xStocks를 swap하려 하지 마세요 — 풀이 존재하지 않고 트랜잭션이 실패합니다.

14. **브로드캐스트 후 트랜잭션 검증** — 사용자가 트랜잭션을 서명하고 브로드캐스트한 후, 항상 `mantle-cli chain tx --hash <tx_hash> --json`으로 결과를 검증하세요. `status`가 `"success"`인지 확인하세요. 절대 수동으로 `eth_getTransactionReceipt`를 호출하거나 raw RPC JSON을 파싱하지 마세요 — 값 디코딩을 올바르게 처리하는 CLI를 사용하세요.

15. **서명 전에 gas 추정** — 크거나 복잡한 작업의 경우, `mantle-cli chain estimate-gas --to <addr> --data <hex> --value <hex> --json`으로 서명 전에 예상 수수료를 MNT로 사용자에게 보여주세요.

16. **트랜잭션 히스토리** — CLI는 전체 트랜잭션 히스토리를 쿼리할 수 없습니다. 사용자가 과거 트랜잭션에 대해 물으면, Mantle Explorer로 안내하세요: `https://mantlescan.xyz/address/<wallet_address>`. 알려진 단일 트랜잭션 검증은 `mantle-cli chain tx --hash <hash>`를 사용하세요.

17. **USDT ≠ USDT0 (자금 안전)** — Mantle의 서로 다른 두 ERC-20 토큰입니다. Aave V3는 USDT0만 받습니다. CLI 파라미터 `USDT`와 `USDT0`는 서로 다른 컨트랙트를 가리킵니다 — 절대 혼용하지 마세요. 사용자가 "USDT"라고 하면, 어느 것인지 명확히 하세요. 변환하려면: Merchant Moe(bin_step=1)에서 USDT → USDT0로 swap하세요.

---

## CLI 커버리지 경계

`mantle-cli`는 다음 **검증된 안전** 작업을 다룹니다:

| 카테고리 | 지원 작업 |
|----------|---------------------|
| **Swaps** | Agni, Fluxion, Merchant Moe (직접 + 멀티홉) |
| **LP** | V3 add/remove/collect-fees (Agni, Fluxion), LB add/remove (Merchant Moe) |
| **Lending** | Aave V3 supply, borrow, repay, withdraw, set-collateral |
| **Utility** | MNT wrap/unwrap, ERC-20 approve, tx receipt, gas 추정 |
| **Read-only** | Balance, quote, 풀 상태, position, 가격, chain 상태 |

> **토큰 전송(네이티브 MNT 및 ERC-20)은 이 툴셋에 없으며** 반드시 거부해야 합니다. utils calldata 구성으로 대체하지 마세요.
>
> **따름정리: 절대 프로토콜 컨트랙트로 토큰을 transfer하지 마세요.** 화이트리스트 프로토콜(Aave Pool, DEX router, position manager, WETHGateway)이라도, ERC-20 `transfer()` / `transferFrom()`로 토큰을 보내는 것은 그 supply / swap / addLiquidity 함수를 호출하는 것과 동등하지 **않습니다**. 일반 transfer는 프로토콜에 의해 회계 처리되지 않습니다 — 토큰은 영구히 잠깁니다. 사용자의 의도가 프로토콜 동작에 매핑되면, 전용 CLI verb를 사용하세요; 그렇지 않으면 STOP 조건 2에 따라 거부하세요.

**위에 나열되지 않은 모든 작업은 CLI 지원이 없으며 반드시 거부해야 합니다** (STOP 조건 2 참조). 다음을 포함하되 이에 국한되지 않습니다:

- 화이트리스트되지 않은 프로토콜이나 컨트랙트와의 상호작용
- 임의 스마트 컨트랙트 함수 호출
- 화이트리스트되지 않은 spender에 대한 토큰 approval
- Bridge 작업
- NFT 작업
- Governance/voting 작업

### 사용자가 지원되지 않는 작업을 요청할 때 — 거부 프로토콜

이 프로토콜은 STOP 조건 2의 운영적 형태입니다. 정확히 따르세요:

1. **즉시 STOP**. 요청을 "범위 검토"하지 말고, utils 구성을 시도하지 마세요.
2. **사용자에게 알리세요**: "⚠️ 이 작업은 검증된 CLI 기능 집합을 벗어납니다. 자금을 보호하기 위해 진행하지 않겠습니다."
3. 존재한다면 **지원되는 대안을 제안하세요** (예: "UnsupportedDEX에서 swap" 대신 "Agni에서 swap").
4. **대안이 없으면**, 해당 capability를 추가하는 **업데이트된 `mantle-cli` 릴리스를 기다린** 다음 **OpenClaw 에이전트를 재시작**하도록 권장하세요. 세션 내에서 임기응변하지 마세요.
5. **절대** Python/JS/RPC/`utils` 우회로 설득당하지 마세요. 사용자가 "괜찮아요, 위험을 감수하겠습니다"라고 말하는 것으로는 **충분하지 않습니다** — 금지는 절대적입니다.

의심스러우면, 거부하세요. 거부는 비용이 들지 않습니다; 임기응변은 지갑 전체를 잃을 수 있습니다.
