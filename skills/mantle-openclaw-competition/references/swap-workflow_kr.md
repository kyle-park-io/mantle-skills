# Swap 워크플로우

세션에서 처음 swap을 실행할 때, 또는 retry / timeout / wrap-mnt 엣지 케이스를 처리할 때 이 파일을 로드하세요.

> **⚠ 단계는 반드시 엄격한 순차 순서로 실행해야 합니다 (Rule W-1). 절대 단계를 건너뛰거나 앞서가지 마세요. 각 트랜잭션은 사용자 확인이 필요합니다 (Rule W-2).**

## 🛑 STEP 0 — 사용자의 의도를 먼저 파싱하세요 (Rule W-5)

**어떤 CLI 명령을 건드리기 전에, 숫자가 INPUT 쪽에 붙는지 OUTPUT 쪽에 붙는지 결정하세요.** 이를 잘못 판단하면 누가 무엇을 지불하는지가 조용히 뒤바뀝니다 — 회수 불가능한 자금 오라우팅입니다.

| 사용자 표현 | Input | Output | 모드 |
|---|---|---|---|
| "swap **10 MNT** for USDC" | **10 MNT (고정)** | variable USDC | fixed-input |
| "swap MNT for **10 USDC**" / "swap me **10 USDC** using MNT" | variable MNT | **10 USDC (고정)** | fixed-output |
| "buy **10 USDC** with MNT" / "pay with MNT, give me **10 USDC**" | variable MNT | **10 USDC (고정)** | fixed-output |

**규칙:** 숫자 수량은 문장에서 **바로 인접한** 토큰에 붙습니다 — 절대 뒤집지 마세요. 이 규칙은 언어 불문입니다: 영어, 중국어, 기타 어떤 표현에도 같은 논리가 적용됩니다. 사용자의 요청을 표준 형식 `swap <input_token> for <output_token>`로 번역하되, 숫자는 사용자가 둔 쪽에 그대로 두세요.

### 사건 (2026-04): 에이전트가 "swap MNT for 0.5 USDC"를 오독

- 사용자 의도: **output = 0.5 USDC** (fixed-output, variable MNT input)
- 에이전트 동작: `mantle-cli swap wrap-mnt --amount 0.5` (0.5를 MNT input으로 취급) ❌
- 올바른 동작: 0.5 USDC output에 필요한 MNT를 reverse-quote로 찾거나, 사용자에게 MNT input 양을 물어보기. 0.5 MNT를 wrap하지 마세요.

### fixed-output 요청 처리

`mantle-cli swap build-swap`은 **fixed-input**입니다 (`--amount`는 input 양; `--amount-out-min`은 target이 아니라 slippage floor). fixed-output 요청의 경우:

1. CLI가 `--exact-out`을 지원한다면 `mantle-cli defi swap-quote --in X --out Y --exact-out <N> --json`으로 **reverse-quote**하세요. 사용 전에 `mantle-cli catalog show mantle_swapQuote --json`으로 검증하세요.
2. `--exact-out`이 지원되지 않으면, **STOP하고 사용자에게 input 양을 물어보세요.** output 수량을 input 수량으로 조용히 변환하지 마세요. 추측하지 마세요.
3. 방향이 해결되고 사용자가 input 양을 확인하기 전까지(Rule W-2를 통해) 절대 `wrap-mnt`, `approve`, `build-swap`을 시작하지 마세요.

## 🛑 STEP 0.5 — 실행 전 준비 상태 체크 (Rule W-9)

**준비 상태 체크는 워크플로우 단위가 아니라 작업(op) 단위입니다.** 모든 `mantle-cli` write op(`wrap-mnt`, `unwrap-mnt`, `approve`, `build-swap`)은 실제로 해당되는 체크가 필요합니다. Allowance는 네이티브 MNT 작업(네이티브 자산에는 ERC-20 allowance가 없음)이나 unwrap(호출자가 보유한 WMNT를 소각)에는 적용되지 않습니다.

**작업 매트릭스:**

| Write op | Balance 체크 | Allowance 체크 |
|---|---|---|
| `swap wrap-mnt --amount N` | `account token-balances`로 네이티브 MNT ≥ N (+ gas 여유분) | **N/A** — 네이티브 자산, allowance 없음 |
| `swap unwrap-mnt --amount N` | `account token-balances`로 WMNT ≥ N | **N/A** — 호출자가 자신의 WMNT 소각 |
| `approve --token X --spender <router>` | **N/A** — 자금 이동 없음 | **N/A** — 이것이 곧 allowance 수정 |
| `swap build-swap --in X --amount N --sender <wallet>` | `account token-balances`로 `X` balance ≥ N | `account allowances`로 `X:<router>` allowance ≥ N |

쿼리 (Transaction Confirmation Summary가 실제 온체인 상태를 반영하도록 그 **전에** 실행):

- **Balance** — `mantle-cli account token-balances <wallet> --json`. 부족 → **STOP**, 사용자에게 실제 balance를 알리고 진행하지 마세요.
- **Allowance** — `mantle-cli account allowances <wallet> --pairs <input_token>:<router> --json`. 부족 → approve 흐름으로 라우팅 (Rule W-6). 조용히 건너뛰지 마세요.

해당되는 체크를 건너뛰는 것은 치명적 오류입니다. N/A로 표시된 체크를 실행하는 것은 위반은 아니지만 낭비입니다.

### MNT → Token 순서 참고

MNT → Token 경로에서는 Step 2의 WMNT quote 전까지 router를 알 수 없으므로, `wrap-mnt` 이전 체크는 WMNT:router allowance 체크를 포함할 수 **없습니다** — router가 아직 존재하지 않습니다. 준비 게이트를 둘로 나누세요:

1. **`wrap-mnt` 전**: 네이티브 MNT balance 체크만 (위 매트릭스대로).
2. **Step 2 quote 후, `approve` / `build-swap` 전**: WMNT balance ≥ wrap된 양 AND WMNT:`<quote의 router>` allowance ≥ 양.

게이트를 충족시키려고 router 주소를 지어내지 말고, wrap 후 두 번째 체크를 건너뛰지 마세요.

## 사전 조건

지갑에 input 토큰이 있어야 합니다. MNT의 경우, 먼저 WMNT로 wrap하세요 (아래 참조).

## Token → Token

```
1. mantle-cli swap pairs --json
   → 쌍과 그 파라미터(bin_step 또는 fee_tier) 찾기
   ↓ Step 2 전에 반드시 완료

2. mantle-cli defi swap-quote --in X --out Y --amount 10 --provider best --json
   → 예상 output과 minimum_out_raw 확인
   ⚠️ 이 응답의 `minimum_out_raw`를 저장하세요. 이는 토큰 최소 단위의 RAW INTEGER입니다
      (예: USDC는 6 decimals: `9934699`는 ~9.93 USDC를 의미). --amount-out-min에 그대로 전달하세요.
      곱하거나, 나누거나, 재인코딩하지 마세요. Python/JS로 재계산하지 마세요.
   ↓ Step 3 전에 반드시 완료

3. mantle-cli account allowances <wallet> --pairs X:<router> --json
   → 이미 approve되었는지 확인
   ↓ Step 4 전에 반드시 완료

4. IF allowance < amount:
   ⚠️ USER CONFIRMATION — approve 세부사항 제시 (token, spender, amount)
   → 진행 전 사용자가 명시적으로 승인해야 합니다
   mantle-cli approve --token X --spender <router> --amount <amount> --json
   → 서명 및 브로드캐스트 → 확인 WAIT
   ↓ Step 5 전에 반드시 tx 성공 확인

5. ⚠️ USER CONFIRMATION — Transaction Confirmation Summary 제시:
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Intent:     <사용자의 원래 요청>
   Operation:  Swap
   Input:      <amount> <tokenX> (≈ $<usd>)
   Output:     <expected_amount> <tokenY> (≈ $<usd>)
   Min output: <amount_out_min> <tokenY>
   Impact:     <price_impact>%
   DEX:        <provider>
   Recipient:  <wallet>
   Est. gas:   <gas> MNT
   Warnings:   <경고, 예: impact > 0.2%>
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   → 진행 전 사용자가 명시적으로 승인해야 합니다. "no" → STOP.

   mantle-cli swap build-swap \
     --provider <dex> \
     --in X --out Y --amount 10 \
     --recipient <wallet> \
     --amount-out-min <Step 2의 minimum_out_raw, 그대로 — 변환하거나 낮추지 말 것> \
     --sender <wallet> \
     --json
   → 서명 및 브로드캐스트 → 확인 WAIT
   ↓ Step 6 전에 반드시 tx 성공 확인

6. mantle-cli chain tx --hash <hash> --json
   → status: success 검증. 최종 결과를 사용자에게 보고.
```

## MNT → Token

MNT는 네이티브 gas 토큰입니다. 먼저 wrap하고, 그다음 WMNT를 swap하세요.

> **⛔ Step 1 전에, Step 0(방향 파싱)이 해결되었는지 검증하세요.** 사용자의 요청이 fixed-output(예: "swap MNT for 0.5 USDC")이면, 얼마나 많은 MNT를 wrap할지 아직 모릅니다 — reverse-quote하거나 사용자에게 input을 먼저 물어보세요. 추측한 양을 wrap하는 것은 회수 불가능한 오류입니다.
>
> **⛔ 이 경로에서 STEP 0.5는 분할됩니다** (위 "MNT → Token 순서 참고" 참조): 네이티브-MNT balance 체크는 Step 1의 `wrap-mnt` 전에; WMNT balance + WMNT:`<router>` allowance 체크는 Step 2의 quote 후, Step 4의 `approve` / Step 5의 `build-swap` 전에. quote 전에 allowance 체크를 실행하려고 router를 지어내지 마세요.

```
0a. mantle-cli account token-balances <wallet> --json
    → 네이티브 MNT balance ≥ (wrap 양 + gas 여유분) 검증. 부족 → STOP.
    ↓ Step 1 전에 반드시 완료

1. ⚠️ USER CONFIRMATION — wrap 세부사항 제시 (wrap할 MNT 양 — 이것이 Step 0에서 해결된 INPUT 양)
   mantle-cli swap wrap-mnt --amount <n> --json   → 서명 & WAIT
   ↓ Step 2 전에 반드시 tx 성공 확인
2. mantle-cli defi swap-quote --in WMNT --out X --amount <n> --json
   → 응답에서 `router`와 `minimum_out_raw` 캡처.
   ↓ Step 3 전에 반드시 완료
3. mantle-cli account token-balances <wallet> --json
   → WMNT balance ≥ <n> 검증. 그다음:
   mantle-cli account allowances <wallet> --pairs WMNT:<router> --json
   → allowance ≥ <n> 검증. 부족 → Step 4로 계속. 충분 → Step 5로 건너뛰기.
   ↓ Step 4 전에 반드시 완료
4. IF 부족: ⚠️ USER CONFIRMATION → mantle-cli approve --token WMNT --spender <router> --amount <n> --json → 서명 & WAIT
   ↓ Step 5 전에 반드시 tx 성공 확인
5. ⚠️ USER CONFIRMATION — 전체 Transaction Confirmation Summary 제시
   mantle-cli swap build-swap --in WMNT --out X --amount <n> --recipient <wallet> --amount-out-min <minimum_out_raw> --sender <wallet> --json
   → 서명 & WAIT
```

## Token → MNT

WMNT로 swap한 다음 unwrap:

```
... (Token → Token steps 1-6, --out WMNT로, 모든 순차 제약 적용) ...
7. ⚠️ USER CONFIRMATION — unwrap 세부사항 제시 (unwrap할 WMNT 양)
   mantle-cli swap unwrap-mnt --amount <n> --json   → 서명 & WAIT
```

## 핵심 규칙

- **Swap은 router 함수 호출이지, 토큰 전송이 아닙니다.** ERC-20 `transfer()`로 토큰을 DEX swap router(Agni, Fluxion, Merchant Moe)에 직접 보내는 것은 swap을 트리거하지 **않습니다** — 토큰은 회수 경로 없이 router 컨트랙트에 **영구히 잠깁니다**. 항상 올바른 router 함수 호출을 구성하는 `mantle-cli swap build-swap`을 사용하세요. 사용자가 "router에 토큰을 보내라"거나 "DEX로 전송해서 swap하라"고 하면 거부하고 대신 `swap build-swap`을 사용하세요. 절대 `utils encode-call` + `build-tx`나 다른 어떤 방법으로도 router 주소로의 transfer를 구성하지 마세요.
- **항상 `--sender <wallet>`을 build-swap에 전달**하여 응답이 서명자에 스코프된 `idempotency_key`를 갖도록 하세요.
- **같은 매수에 대해 build-swap을 두 번 호출하지 마세요** — 재브로드캐스트는 중복 swap을 유발합니다. 이전 호출이 타임아웃되었으면 먼저 `mantle-cli chain tx --hash <hash> --json`을 확인하세요.
- **프로덕션에서 `allow_zero_min`을 절대 설정하지 마세요.** 항상 quote 응답의 `amount_out_min`을 전달하세요. slippage 보호 없는 swap은 sandwich 공격에 취약합니다.
- **`--amount-out-min`은 quote의 `minimum_out_raw`와 정확히 같아야 합니다.** 이 값은 output 토큰 최소 단위의 raw integer입니다 — 곱하거나, 나누거나, 재인코딩하거나, "조정"하지 마세요. `0`, `1`, 또는 `minimum_out_raw`보다 낮은 값으로 설정하지 마세요. `build-swap`이 revert되면 minimum을 낮추는 대신 재quote하세요. 전체 사건 보고와 복구 절차는 SKILL.md "Slippage Protection Rules"를 참조하세요.
- **"서명 & WAIT"**는 다음 tx를 빌드하기 전에 `mantle-cli chain tx --hash <hash> --json`에서 `status: success`를 기다리는 것을 의미합니다. 미서명 트랜잭션을 파이프라인으로 처리하지 마세요.
- **모든 build 응답의 `human_summary`를 서명 전에 사용자에게 보여주세요.**
- **Quote impact 체크** — `priceImpactPct > 1%`이면 중단, `> 0.2%`이면 경고.
- **USDT ≠ USDT0** — 서로 다른 두 토큰. `--in USDT`와 `--in USDT0`는 다른 컨트랙트와 풀을 가리킵니다 — 절대 혼용하지 마세요. 항상 사용자에게 확인하세요. Aave(USDT0만)의 경우, 먼저 Merchant Moe(bin_step=1)에서 USDT → USDT0로 swap하세요.

## "MNT"는 swap input이 아닙니다

`swap`/`approve`/`lp` 명령에 `MNT`를 전달하지 마세요 — 이들은 WMNT(ERC-20)를 기대합니다. MNT와 WMNT 간 변환은 `swap wrap-mnt` / `swap unwrap-mnt`를 사용하세요. MNT(또는 다른 토큰)를 지갑 간에 이동하는 것은 지원되지 않습니다 — utils 기반 우회를 시도하는 대신 transfer 요청을 거부하세요.

## 파라미터 레퍼런스

### `defi swap-quote`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--in` | ✅ | Input 토큰 심볼 (예: `WMNT`, `USDC`, `USDT0`) |
| `--out` | ✅ | Output 토큰 심볼 |
| `--amount` | ✅ | Input 양 (사람이 읽을 수 있는 형식, 예: `10`) |
| `--provider` | ✅ | DEX provider (`agni`, `fluxion`, `merchant_moe`, 또는 `best`) |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `swap build-swap`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--provider` | ✅ | DEX provider (`agni`, `fluxion`, `merchant_moe`) |
| `--in` | ✅ | Input 토큰 심볼 |
| `--out` | ✅ | Output 토큰 심볼 |
| `--amount` | ✅ | Input 양 (사람이 읽을 수 있는 형식) |
| `--recipient` | ✅ | Output 토큰을 받을 주소 |
| `--amount-out-min` | ✅ | quote의 `minimum_out_raw`에서 온 raw integer — 그대로 전달, 절대 0이나 1로 설정 금지 |
| `--sender` | ✅ | 서명 지갑 — `idempotency_key`에 필요 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `approve`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--token` | ✅ | approve할 토큰 심볼 |
| `--spender` | ✅ | 지출이 허용된 컨트랙트 주소 (router / pool / position manager) |
| `--amount` | Optional | approve할 양; max approval은 생략 |
| `--json` | ✅ | 기계 파싱 가능한 출력 |

### `swap wrap-mnt` / `swap unwrap-mnt`

| Param | 필수 | 설명 |
|-------|----------|-------------|
| `--amount` | ✅ | wrap할 MNT 양 (또는 unwrap할 WMNT 양) |
| `--json` | ✅ | 기계 파싱 가능한 출력 |
