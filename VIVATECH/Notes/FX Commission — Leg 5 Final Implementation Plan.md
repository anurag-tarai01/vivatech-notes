
## Confirmed Decisions

|#|Decision|
|---|---|
|Commission wallet|**AMT03** (all currencies already exist from setup)|
|Commission type|**Percentage** of source amount, in source currency|
|Commission config|New `AdminConfig` entry: `fx_commission_rate` (e.g., `0.01` = 1%)|
|Same `@Transactional`|Yes — Leg 5 is atomic with Legs 1–4. All or nothing|
|Transfer type|Unchanged — reuses existing type; FX identified by non-null `fxRateSnapshotId`|
|CommissionInfo in AMT03|Yes — same `commission_info` table row saved and visible in AMT03 commission report|
|Reversal|Out of scope for now|
|Zero commission|If `fx_commission_rate` not configured or is zero, Leg 5 is skipped — transfer proceeds normally|

---

## How the Existing Commission Pattern Works (Reference)

TransferRequestBodyAdvice / ServiceAndCommissionChargeService

    → builds CommissionInfo { accountAggregateId, amount, transferType, paidStatus=false }

    → adds to dto.commissionInfos[]

NewWalletService.updateWalletBalances()

    → handleCommissions() deposits each CommissionInfo.amount into its accountAggregateId wallet

    → calls dto.setCommissionDepositData() to record the operation row

GPayAccountTransferService.finishTransaction()

    → commissionInfoListener.saveCommissionData(dto)   ← saves CommissionInfo rows to DB

**We follow this exact pattern.** No new listeners, no new events, no new DB tables.

---

## Files to Change

|#|File|Change|
|---|---|---|
|1|`FxConversionResult.java`|Add `fxCommissionAmount` field|
|2|`FxRateService.java`|Read `fx_commission_rate`, calculate commission, return it|
|3|`TransferEventDto.java`|Add `fxCommissionAmount` field|
|4|`TransferRequestBodyAdvice.java`|Set `fxCommissionAmount` on dto + build `CommissionInfo` for AMT03|
|5|`NewWalletService.java`|Leg 1: debit `amount + commission`. Add Leg 5: credit AMT03|
|6|DB: `admin_config`|Insert row: `fx_commission_rate = 0.01`|

---

## Step-by-Step Changes

---

### Step 1 — `FxConversionResult.java`

**Add `fxCommissionAmount` field.** This carries the calculated commission amount (in source currency) back to the caller.

**Current file (lines 9–18):**

java

@Data

@Builder

public class FxConversionResult {

    private BigMoney sourceAmount;

    private BigMoney targetAmount;

    private String rateSnapshotId;

    private BigDecimal effectiveRate;

    private BigDecimal marketRate;

    private BigDecimal spreadRate;

}

**After change:**

java

@Data

@Builder

public class FxConversionResult {

    private BigMoney sourceAmount;

    private BigMoney targetAmount;

    private String rateSnapshotId;

    private BigDecimal effectiveRate;

    private BigDecimal marketRate;

    private BigDecimal spreadRate;

    private BigMoney fxCommissionAmount; // NEW — in source currency, e.g. 1 USD

}

---

### Step 2 — `FxRateService.java`

**Read `fx_commission_rate` config and calculate commission.** Insert after the existing spread block (after line 75), and add to the `return` builder.

**Add after the spread config block (after line 75):**

java

// FX Commission Rate (explicit service charge, separate from spread)

BigDecimal commissionRate = BigDecimal.ZERO;

AdminConfig commissionConfig = adminConfigQueryRepository.findByConfigName("fx_commission_rate");

if (commissionConfig != null && commissionConfig.getConfigValue() != null) {

    commissionRate = new BigDecimal(commissionConfig.getConfigValue());

}

// Commission is a % of source amount, always in source currency

BigMoney fxCommissionAmount = sourceAmount.multipliedBy(commissionRate, RoundingMode.HALF_UP);

**Modify the `return` builder (lines 102–109) to include the new field:**

java

return FxConversionResult.builder()

        .sourceAmount(sourceAmount)

        .targetAmount(BigMoney.of(CurrencyUnit.of(targetCurrency), convertedAmount))

        .rateSnapshotId(snapshot.getId())

        .marketRate(marketRate)

        .effectiveRate(effectiveRate)

        .spreadRate(spread)

        .fxCommissionAmount(fxCommissionAmount)  // NEW

        .build();

---

### Step 3 — `TransferEventDto.java`

**Add `fxCommissionAmount` field alongside the existing FX fields.**

**After line 125** (after `fxTargetAmount`), add:

java

// FX Commission (Leg 5) — amount in source currency, credited to AMT03

@JsonDeserialize(using = BigMoneyDeserializer.class)

@JsonSerialize(using = BigMoneySerializer.class)

private BigMoney fxCommissionAmount;

NOTE

`TransferDto.java` does **not** need this field — the commission is calculated server-side in `TransferRequestBodyAdvice` and never comes from the client.

---

### Step 4 — `TransferRequestBodyAdvice.java`

**Replace the existing FX block (lines 139–143)** to also set `fxCommissionAmount` and build the `CommissionInfo`.

**Current code (lines 139–143):**

java

if (isFx) {

    FxConversionResult fxResult = fxRateService.convert(dto.getAmount(), targetCurrency);

    dto.setFxRateSnapshotId(fxResult.getRateSnapshotId());

    dto.setFxTargetAmount(fxResult.getTargetAmount());

}

**Replace with:**

java

if (isFx) {

    FxConversionResult fxResult = fxRateService.convert(dto.getAmount(), targetCurrency);

    dto.setFxRateSnapshotId(fxResult.getRateSnapshotId());

    dto.setFxTargetAmount(fxResult.getTargetAmount());

    // Leg 5 — FX Commission: set amount and register CommissionInfo for AMT03

    BigMoney fxCommission = fxResult.getFxCommissionAmount();

    if (fxCommission != null && !fxCommission.isZero()) {

        dto.setFxCommissionAmount(fxCommission);

        CommissionInfo commissionInfo = new CommissionInfo();

        commissionInfo.setTransferType(dto.getTransferType());

        commissionInfo.setAmount(fxCommission);

        commissionInfo.setPaidStatus(false);

        commissionInfo.setTransferId(dto.getTransferAggregateId());

        commissionInfo.setAccountAggregateId(Constants.AMT03_WALLET_AGGREGATE_ID);

        List<CommissionInfo> commissionInfos = dto.getCommissionInfos() != null

                ? dto.getCommissionInfos() : new ArrayList<>();

        commissionInfos.add(commissionInfo);

        dto.setCommissionInfos(commissionInfos);

    }

}

**Add imports** at the top of `TransferRequestBodyAdvice.java`:

java

import com.vivacom.mfs.common.dto.CommissionInfo;

import com.vivacom.mfs.common.Constants;

import java.util.ArrayList;

import java.util.List;

NOTE

`dto` here is `TransferDto`. The `CommissionInfo` (DTO from `core-api-kotlin`) gets mapped to `TransferEventDto.commissionInfos` in the controller layer, exactly as other commission flows do. Check if `TransferDto` needs `commissionInfos` field — if not already present, it may need to be added. **Verify this.**

---

### Step 5 — `NewWalletService.java`

Two changes inside `updateWalletBalancesFxBridge()`:

#### 5a — Leg 1: Include commission in the sender debit

**Current Leg 1 (lines 64–67):**

java

BigMoney totalWithdrawAmount = dto.getAmount();

BigMoney withdrawWalletCurrentBalance = getCurrentWalletBalance(dto.getFromAccountId());

dto.setWithDrawBalanceData(withdrawWalletCurrentBalance, totalWithdrawAmount);

**Replace with:**

java

// Include FX commission in sender debit (Leg 1 + Leg 5 deducted from sender together)

BigMoney fxCommission = dto.getFxCommissionAmount() != null

        ? dto.getFxCommissionAmount()

        : BigMoney.zero(dto.getAmount().getCurrencyUnit());

BigMoney totalWithdrawAmount = MfsUtils.getScaledMoney(dto.getAmount().plus(fxCommission));

BigMoney withdrawWalletCurrentBalance = getCurrentWalletBalance(dto.getFromAccountId());

dto.setWithDrawBalanceData(withdrawWalletCurrentBalance, totalWithdrawAmount);

#### 5b — Leg 5: Credit AMT03 (source currency)

**After line 123** (after the existing `return dto;` — but **before** the return), add Leg 5:

java

        /* ---------------------------------------------------------

         Leg 5: Credit AMT03 FX Commission Wallet (Source Currency)

         --------------------------------------------------------- */

        if (fxCommission != null && !fxCommission.isZero()) {

            WalletInfo amt03Wallet = walletService.getByAggregateIdAndCurrencyAndStatusFromDb(

                    Constants.AMT03_WALLET_AGGREGATE_ID, sourceCurrency, WalletStatus.ACTIVE);

            BigMoney amt03CurrentBalance = getCurrentWalletBalance(amt03Wallet.getWalletId());

            dto.setCommissionDepositData(amt03Wallet.getWalletId(), amt03CurrentBalance, fxCommission);

            walletQueryRepository.depositToWallet(amt03Wallet.getWalletId(), fxCommission.getAmount(), now);

            sendMessageToRabbitMQ(amt03Wallet.getAggregateId(), fxCommission,

                    amt03CurrentBalance, dto.getTransferAggregateId(), "MoneyAddedEvent");

            log.info("FX Leg 5: FX commission {} credited to AMT03 ({}) for transfer id: {}",

                    fxCommission, sourceCurrency, dto.getTransferAggregateId());

        }

        return dto;

IMPORTANT

`dto.setCommissionDepositData()` feeds `dto.commimssionWalletTransferDtos` which the `AccountTransferListener` uses to write the **5th `AccountTransferOperation` row** automatically. No additional listener code is needed.

---

### Step 6 — Database

Insert a new row into `admin_config` table:

sql

INSERT INTO admin_config (config_name, config_value, description)

VALUES ('fx_commission_rate', '0.01', 'FX transfer commission rate as a decimal (e.g. 0.01 = 1%)');

> Adjust the value with your senior before inserting.

---

## Complete Data Flow (End to End)

1. Client sends P2P transfer request with targetCurrency != sourceCurrency

2. TransferRequestBodyAdvice.afterBodyRead()

   → FxRateService.convert(100 USD, "SOS")

       reads fx_spread_rate   → adjusts effective rate

       reads fx_commission_rate = 0.01

       calculates fxCommission = 100 USD × 0.01 = 1 USD

       returns FxConversionResult { targetAmount=55,987 SOS, fxCommissionAmount=1 USD }

   → dto.fxRateSnapshotId = snapshot.id

   → dto.fxTargetAmount   = 55,987 SOS

   → dto.fxCommissionAmount = 1 USD                ← NEW

   → dto.commissionInfos.add({ AMT03, 1 USD })      ← NEW

3. GPayAccountTransferService.executeInterceptorsAndInitiate()

   → calculateServiceAndCommissionCharge() — skips FX type or adds no extra charge

   → accountTransferListener.initiateTransaction() — saves AccountTransfer to DB

4. GPayAccountTransferService.executeTransactionCore()

   → newWalletService.updateWalletBalances()

       → updateWalletBalancesFxBridge()

           Leg 1: Debit sender    101 USD  (100 + 1 commission)   ← MODIFIED

           Leg 2: Credit AMT01 USD  100 USD

           Leg 3: Debit  AMT01 SOS  55,987 SOS

           Leg 4: Credit receiver   55,987 SOS

           Leg 5: Credit AMT03 USD    1 USD                       ← NEW

   → finishTransaction()

       → commissionInfoListener.saveCommissionData(dto)

           saves CommissionInfo row: { AMT03, 1 USD, paidStatus=false, transferId=P2PFX... }

---

## DB Side-Effects After a Successful FX Transfer

|Table|What gets written|
|---|---|
|`account_transfer`|1 row with `fx_rate_snapshot_id` = set|
|`account_transfer_operation`|**5 rows** — Legs 1–5, all with same `transfer_id`|
|`exchange_rate_snapshot`|1 row (already existing from Legs 1–4)|
|`commission_info`|**1 new row** — `{ accountAggregateId=AMT03, amount=1 USD, paidStatus=false }`|

---

## Verification Checklist

|#|Test|Expected|
|---|---|---|
|1|FX transfer 100 USD → SOS, commission 1%|Sender debited **101 USD**|
|2|Same transfer|Receiver gets correct SOS amount (unaffected)|
|3|Same transfer|AMT03 USD wallet balance increases by **1 USD**|
|4|Same transfer|`commission_info` has 1 new row: `amount=1 USD`, `accountAggregateId=AMT03`|
|5|Same transfer|`account_transfer_operation` has exactly **5 rows**|
|6|FX with `fx_commission_rate = 0`|Leg 5 skipped, sender debited **100 USD** only, 4 operation rows|
|7|Same-currency P2P (USD → USD)|No FX logic triggered — zero regression|
|8|AMT03 USD not found (setup error)|Transfer fails with clear exception, rolls back all legs|