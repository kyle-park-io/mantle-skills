# 대출 SOP

Mantle에서 Aave V3 대출 작업(공급, 차입, 상환, 출금)을 위한 이 표준 절차를 사용하세요.

## 중요: 트랜잭션 구축에 CLI 사용

**항상 `mantle-cli`를 사용하여 미서명 트랜잭션을 구축하세요.** calldata를 수동으로 구성하거나, 텍스트에서 주소를 추출하거나, approve 호출을 직접 구축하지 마세요. CLI는 주소 해석, ABI 인코딩, 화이트리스트 검증을 올바르게 처리합니다.

```bash
# 모든 명령은 구조화된 출력을 위해 --json을 지원합니다
mantle-cli aave supply          --asset USDC --amount 1.0 --on-behalf-of 0x... --json
mantle-cli aave set-collateral  --asset WMNT --user 0x... --json  # 담보 활성화 (진단)
mantle-cli aave borrow          --asset USDC --amount 0.5 --on-behalf-of 0x... --json
mantle-cli aave repay           --asset USDC --amount 0.5 --on-behalf-of 0x... --json
mantle-cli aave repay           --asset USDC --amount max --on-behalf-of 0x... --json
mantle-cli aave withdraw        --asset USDC --amount 1.0 --to 0x... --json
mantle-cli aave withdraw        --asset USDC --amount max --to 0x... --json
mantle-cli aave positions       --user 0x... --json  # 포지션 + 담보 플래그 확인
mantle-cli aave markets         --json  # 결정 전 APY/TVL 확인
```

승인(공급/상환 전 필요)의 경우:
```bash
mantle-cli swap approve --token USDC --spender 0x458F293454fE0d67EC0655f3672301301DD51422 --amount max --json
```

CLI는 `to`, `data`, `value`, `chainId`가 포함된 `unsigned_tx`를 출력합니다 — **`from` 필드 없음**. 수정 없이 서명자에게 직접 전달하세요.

## 1단계: 대출 시장 확인

```bash
mantle-cli aave markets --json
```

- 공급 APY, 차입 APY, TVL, LTV, 청산 임계값을 검토하세요.
- 대상 자산이 지원되는 Aave V3 준비금인지 확인하세요.
- 지원 자산: WETH, WMNT, USDT0, USDC, USDe, sUSDe, FBTC, syrupUSDT, wrsETH, GHO.
- **중요:** Aave V3에서는 USDT0만 지원되고 USDT는 지원되지 않습니다. 사용자가 USDT를 보유한 경우 먼저 Merchant Moe에서 USDT → USDT0으로 스왑해야 합니다 (USDT/USDT0 풀, bin_step=1).

## 2단계: 입력 정규화

- 자산 심볼 또는 주소
- 금액 (소수, 또는 상환/출금의 경우 `max`)
- 지갑 주소 (공급/차입/상환의 경우 on_behalf_of, 출금의 경우 to)

## 격리 모드

일부 Aave V3 준비금은 **격리 모드 자산**입니다 — 사용자의 유일한 담보가 격리 모드 자산인 경우, 사용자는 차입을 제한하는 격리 모드에 진입합니다.

### Mantle의 격리 모드 자산

| 자산 | 격리 모드 | 부채 한도 |
|-------|:-:|---:|
| WETH | 예 | $30,000,000 |
| WMNT | 예 | $2,000,000 |
| 기타 모두 | 아니오 | — |

### 격리 모드에서 차입 가능한 자산

| 자산 | 격리에서 차입 가능 |
|-------|:-:|
| USDC | 예 |
| USDT0 | 예 |
| USDe | 예 |
| GHO | 예 |
| sUSDe, FBTC, WETH, WMNT, syrupUSDT, wrsETH | **아니오** |

### 규칙

- 사용자가 WETH 또는 WMNT만 담보로 보유하는 경우 → 격리 모드에 있음
- 격리 모드에서는 USDC, USDT0, USDe, GHO만 차입 가능
- 모든 격리 모드 차입자의 총 부채는 부채 한도로 제한됨
- 오류 `UserInIsolationModeOrLtvZero` (0x5b263df7) = 격리 모드에서 화이트리스트되지 않은 자산 차입 시도

### 차입 트랜잭션 구축 전

1. **사용자가 공급한 담보 확인** (aToken 잔액 또는 `getUserAccountData`)
2. 담보가 격리 모드 자산만 있으면 → 격리 모드에 있음
3. 차입할 자산의 `borrowableInIsolation = true` 확인
4. 사용자가 비격리 자산(예: sUSDe)을 차입하려면 먼저 비격리 담보(예: USDC) 추가 필요

## 3단계: 잔액 및 허용량 확인

```bash
mantle-cli account balance <wallet> --tokens USDC,USDT0 --json
```

- 공급/상환을 위해 지갑에 충분한 토큰 잔액이 있는지 확인하세요.
- Aave Pool 주소는 `0x458F293454fE0d67EC0655f3672301301DD51422`입니다.

## 4단계: 필요한 경우 승인

공급 또는 상환에 허용량이 부족한 경우:

```bash
mantle-cli swap approve --token USDC \
  --spender 0x458F293454fE0d67EC0655f3672301301DD51422 \
  --amount <exact_or_max> --owner <wallet> --json
```

- CLI는 화이트리스트에 대해 지출자를 검증합니다.
- 기존 허용량을 확인하고 충분하면 건너뛰려면 `--owner`를 사용하세요.
- 공급/상환으로 진행하기 전에 approve `unsigned_tx`를 서명하고 브로드캐스트하세요.

## 5단계: 대출 트랜잭션 구축

적절한 CLI 명령을 사용하세요:

| 작업 | 명령 | 핵심 플래그 |
|-----------|---------|-----------|
| 예치 | `mantle-cli aave supply` | `--asset`, `--amount`, `--on-behalf-of` |
| 차입 | `mantle-cli aave borrow` | `--asset`, `--amount`, `--on-behalf-of`, `--interest-rate-mode` |
| 상환 | `mantle-cli aave repay` | `--asset`, `--amount` (또는 `max`), `--on-behalf-of` |
| 출금 | `mantle-cli aave withdraw` | `--asset`, `--amount` (또는 `max`), `--to` |

서명자를 위한 구조화된 출력을 받으려면 항상 `--json`을 사용하세요.

## 5b단계: 담보 확인 및 활성화 (차입 전)

자산을 공급한 후, Aave V3는 일반적으로 자동으로 담보를 활성화합니다. 그러나 이 자동 활성화가 실패할 수 있습니다 (특히 WMNT/WETH 같은 격리 모드 자산의 경우). **차입을 시도하기 전에 항상 담보 상태를 확인하세요.**

### 포지션 확인

```bash
mantle-cli aave positions --user <wallet> --json
```

- 공급된 자산의 `collateral_enabled` 필드 확인 (YES / NO)
- 계정 요약에서 `total_collateral_usd` > 0 확인
- `collateral_enabled`가 NO이고 `total_collateral_usd`가 0이면: 담보가 자동 활성화되지 않음

### set-collateral로 진단 및 수정

```bash
# 공급하고 담보 활성화가 필요한 실제 자산 사용 — 항상 WMNT가 아닙니다.
# positions 출력에서 식별: supplied > 0이고 collateral_enabled=NO인 준비금.
mantle-cli aave set-collateral --asset <supplied_asset> --user <wallet> --json
```

이 도구는 트랜잭션 구축 전 **사전 비행 진단**을 실행합니다:

| 확인 | 실패 | 의미 |
|-------|---------|---------|
| aToken 잔액 | `NO_SUPPLY_BALANCE` | 사용자가 이 자산을 공급하지 않음 — 먼저 공급 |
| 준비금 활성 | `RESERVE_NOT_ACTIVE` | 거버넌스에 의해 준비금 비활성화 — 사용 불가 |
| 준비금 LTV | `LTV_IS_ZERO` | 온체인 LTV=0 — 이 자산은 담보로 사용 불가 (근본 원인은 거버넌스 설정, 담보 플래그가 아님). set-collateral을 시도하지 마세요; 이것은 설계에 의한 것입니다. |
| 준비금 동결 | 경고 | 공급/차입 동결되었지만 담보 토글은 작동할 수 있음 |
| 담보 이미 활성화 | `NO-OP` 경고 | 플래그가 이미 설정됨 — 차입 실패는 다른 원인 (오라클 가격 등) |

**중요:** `--user` 플래그는 진단 전용입니다. 실제 트랜잭션은 `msg.sender` (서명 지갑)에서 작동합니다. 서명 지갑은 반드시 `<wallet>`과 동일한 주소여야 합니다 — 그렇지 않으면 잘못된 계정의 담보가 토글됩니다.

진단에서 담보가 활성화되지 않았고 LTV > 0인 경우:
1. `set-collateral` unsigned_tx에 서명하고 브로드캐스트 (서명자는 `<wallet>`이어야 함)
2. 포지션을 다시 확인하여 `collateral_enabled`가 이제 YES인지 확인
3. 차입으로 진행

## 6단계: 서명 및 브로드캐스트

- `unsigned_tx` 객체를 외부 서명자에게 직접 전달하세요.
- **`from` 필드를 추가하지 마세요** — 서명자는 서명 키에서 `from`을 결정합니다.
- **`to`, `data`, `value`, `chainId` 필드를 수정하지 마세요.**

## 7단계: 사후 실행 검증

- 토큰 잔액과 aToken 잔액을 다시 읽어 작업을 확인하세요.
- 공급의 경우: aToken 잔액이 증가했는지 **그리고 positions 출력에서 `collateral_enabled` 확인**.
- set-collateral의 경우: `collateral_enabled`가 변경되었고 `total_collateral_usd` > 0인지 확인.
- 차입의 경우: 토큰 잔액이 증가하고 debt 토큰이 나타났는지 확인.
- 상환의 경우: debt 토큰 잔액이 감소했는지 확인.
- 출금의 경우: aToken 잔액이 감소하고 토큰 잔액이 증가했는지 확인.
- 차입/출금 후 1.0 이상을 유지하는지 헬스 팩터를 확인하세요.

## 흔한 함정

- **승인 누락**: 공급 및 상환은 Aave Pool에 대한 사전 ERC-20 승인이 필요합니다.
- **담보 활성화 누락**: 공급 후 차입하기 전에 항상 `collateral_enabled=YES`를 확인하세요. NO이면 `set-collateral`을 사용하여 활성화하세요. `set-collateral`이 `LTV_IS_ZERO`를 발생시키면 거버넌스 설계에 의해 해당 자산은 담보로 사용할 수 없습니다.
- **unsigned_tx의 `from` 필드**: `from`을 절대 추가하지 마세요 — Privy 및 일부 서명자를 깨뜨립니다.
- **오래된 허용량**: approve에서 `--owner` 플래그를 사용하여 충분하면 자동으로 건너뜁니다.
- **헬스 팩터**: 차입 및 출금은 헬스 팩터를 낮춥니다 — 진행 전에 확인하세요.
- **`max` 의미론**: repay max는 전체 부채를 상환하고; withdraw max는 전체 잔액을 출금합니다.
- **격리 모드**: WETH와 WMNT는 격리 모드 자산입니다. 사용자의 유일한 담보가 이 중 하나이면 USDC, USDT0, USDe, 또는 GHO만 차입 가능합니다. 다른 자산 차입을 시도하면 `UserInIsolationModeOrLtvZero`로 되돌려집니다. 차입 트랜잭션 구축 전에 항상 담보 유형을 확인하세요.
