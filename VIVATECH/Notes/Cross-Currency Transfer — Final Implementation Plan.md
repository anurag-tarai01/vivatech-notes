# Cross-Currency Transfer — Final Implementation Plan

---

## Decisions & Constraints (All Confirmed)

| Topic                                 | Decision                                                                                                                                   |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Currency discovery**                | Read `localCurrency` + `foreignCurrency` from active `Domain` entities. No `domain_currency` table.                                        |
| **FX Rate Tracking**                  | Split into two tables: `ExchangeRate` (Daily OHLC market tracking) and `ExchangeRateSnapshot` (Immutable lock-in for a specific transfer). |
| **TransferType**                      | Reuse existing types (zero code duplication). Dynamically append `"FX"` to the transaction ID prefix (e.g., `P2PFX...`) via `MfsUtils`.    |
| **FX detection on `AccountTransfer`** | `@Transient isFxTransfer()` — derived from operation currencies. One new persisted column: `fx_rate_snapshot_id`.                          |
| **Treasury wallet lookup**            | Always `aggregateId = "AMT01"`, differentiated by `currency` from `BigMoney.amount`.                                                       |
| **P2P FX resolution**                 | Inside `TransferController.subscriberP2p()`. Advice does NOT cover `P2PRequestDto`.                                                        |

---

## Rate Field Reference

```
Market rate from API:   1 USD = 571.50 SOS

spread_rate    = 0.02       → the 2% the platform keeps as profit
effective_rate = 571.50 × (1 − 0.02) = 559.87  → what the user actually gets per USD
inverse_rate   = 1 ÷ 571.50 = 0.00174958       → for reverse direction (SOS → USD)
```

User sends 100 USD → receiver gets `100 × 559.87 = 55,987 SOS`.  
Treasury profit = `57,150 − 55,987 = 1,163 SOS` (≈ 2%).

---

## Phase 1: Schema Changes

### 1.1 New Table: `exchange_rate` (Daily Market Tracker)
Tracks market data throughout the day (OHLC). Updated every 5 minutes by the Quartz poller.

```sql
CREATE TABLE exchange_rate (
    id               VARCHAR(50)   NOT NULL PRIMARY KEY,   -- ULID
    base_currency    VARCHAR(10)   NOT NULL,               -- "USD"
    target_currency  VARCHAR(10)   NOT NULL,               -- "SOS"
    opening_rate     DECIMAL(19,8) NOT NULL,               -- first rate of the day
    closing_rate     DECIMAL(19,8) NULL,                   -- last rate of the day
    high_rate        DECIMAL(19,8) NOT NULL,               -- highest rate today
    low_rate         DECIMAL(19,8) NOT NULL,               -- lowest rate today
    current_rate     DECIMAL(19,8) NOT NULL,               -- latest fetched rate
    source           VARCHAR(50)   NOT NULL,               -- 'EXCHANGE_RATE_API'
    rate_date        DATE          NOT NULL,               -- '2026-08-19'
    created_at       DATETIME      NOT NULL,
    updated_at       DATETIME      NOT NULL,

    INDEX idx_fx_daily (base_currency, target_currency, rate_date)
);
```

### 1.2 New Table: `exchange_rate_snapshot` (Immutable Transfer Record)
Created **once** per FX transfer. Never updated.

```sql
CREATE TABLE exchange_rate_snapshot (
    id                 VARCHAR(50)   NOT NULL PRIMARY KEY,   -- ULID
    exchange_rate_id   VARCHAR(50)   NOT NULL,               -- FK to exchange_rate
    base_currency      VARCHAR(10)   NOT NULL,
    target_currency    VARCHAR(10)   NOT NULL,
    market_rate        DECIMAL(19,8) NOT NULL,               -- 571.50
    spread_rate        DECIMAL(10,6) NOT NULL DEFAULT 0.02,
    effective_rate     DECIMAL(19,8) NOT NULL,               -- 559.87
    inverse_rate       DECIMAL(19,8) NOT NULL,               -- 0.00174958
    created_at         DATETIME      NOT NULL,

    CONSTRAINT fk_snapshot_exchange_rate FOREIGN KEY (exchange_rate_id) REFERENCES exchange_rate(id)
);
```

### 1.3 One New Column on `account_transfer`

```sql
ALTER TABLE account_transfer
    ADD COLUMN fx_rate_snapshot_id VARCHAR(50) NULL;
    -- NULL for all existing same-currency transfers
```

---

## Phase 2: New Classes to Create

### 2.1 Entities & Repositories (`wallet-query` module)

- **`ExchangeRate.java`** & **`ExchangeRateRepository.java`**
- **`ExchangeRateSnapshot.java`** & **`ExchangeRateSnapshotRepository.java`**

### 2.2 `FxRateService` (`application/service/fx/`)
Reads the daily `ExchangeRate` and creates the `ExchangeRateSnapshot`.

```java
@Service
public class FxRateService {
    @Autowired ExchangeRateRepository exchangeRateRepository;
    @Autowired ExchangeRateSnapshotRepository snapshotRepository;
    @Value("${fx.spread.default:0.02}") private BigDecimal defaultSpread;

    @Transactional
    public FxConversionResult convert(BigMoney sourceAmount, String targetCurrency) {
        String baseCurrency = sourceAmount.getCurrencyUnit().getCode();

        ExchangeRate todayRate = exchangeRateRepository
            .findFirstByBaseCurrencyAndTargetCurrencyAndRateDate(baseCurrency, targetCurrency, LocalDate.now())
            .orElseThrow(() -> new MFSTransferException("No active FX rate for today."));

        BigDecimal marketRate = todayRate.getCurrentRate();
        BigDecimal spread = defaultSpread;
        BigDecimal effectiveRate = marketRate.multiply(BigDecimal.ONE.subtract(spread))
                                             .setScale(8, RoundingMode.HALF_UP);
        BigDecimal inverseRate = BigDecimal.ONE.divide(marketRate, 8, RoundingMode.HALF_UP);

        // Create immutable snapshot
        ExchangeRateSnapshot snapshot = ExchangeRateSnapshot.builder()
            .id(MfsUtils.getNextRandomUlid())
            .exchangeRateId(todayRate.getId())
            .baseCurrency(baseCurrency)
            .targetCurrency(targetCurrency)
            .marketRate(marketRate)
            .spreadRate(spread)
            .effectiveRate(effectiveRate)
            .inverseRate(inverseRate)
            .createdAt(Instant.now())
            .build();
        snapshotRepository.save(snapshot);

        BigDecimal convertedAmount = sourceAmount.getAmount().multiply(effectiveRate)
                                                 .setScale(4, RoundingMode.HALF_UP);

        return FxConversionResult.builder()
            .sourceAmount(sourceAmount)
            .targetAmount(BigMoney.of(CurrencyUnit.of(targetCurrency), convertedAmount))
            .rateSnapshotId(snapshot.getId())
            .build();
    }
}
```

### 2.3 `FxRatePersistenceService` (`application/service/fx/`)
Handles the OHLC logic on every poller tick.

```java
@Service @Transactional
public class FxRatePersistenceService {
    @Autowired ExchangeRateRepository repository;

    public void persistRates(String baseCurrency, FxProviderResponse response) {
        LocalDate today = LocalDate.now();

        response.getRates().forEach((targetCurrency, marketRate) -> {
            Optional<ExchangeRate> opt = repository
                .findFirstByBaseCurrencyAndTargetCurrencyAndRateDate(baseCurrency, targetCurrency, today);

            if (opt.isPresent()) {
                // Update existing day's row
                ExchangeRate e = opt.get();
                e.setCurrentRate(marketRate);
                if (marketRate.compareTo(e.getHighRate()) > 0) e.setHighRate(marketRate);
                if (marketRate.compareTo(e.getLowRate()) < 0) e.setLowRate(marketRate);
                e.setUpdatedAt(Instant.now());
                repository.save(e);
            } else {
                // First fetch of the day
                repository.save(ExchangeRate.builder()
                    .id(MfsUtils.getNextRandomUlid())
                    .baseCurrency(baseCurrency).targetCurrency(targetCurrency)
                    .openingRate(marketRate).currentRate(marketRate)
                    .highRate(marketRate).lowRate(marketRate)
                    .rateDate(today).createdAt(Instant.now()).updatedAt(Instant.now())
                    .source("EXCHANGE_RATE_API").build());
            }
        });
    }
}
```

### 2.4 `FxRatePollerJob` (`application/job/`)
Discovers currencies from `Domain` and calls provider every 5 minutes.

---

## Phase 3: Changes to Existing Classes

### 3.1 Dynamic Prefixing in `MfsUtils` (`core-api-kotlin`)
We keep `TransferType` identical, avoiding code duplication in validators and listeners. We just visually inject "FX" into the ID.

```java
// Keep existing method for backward compatibility
public String getAccountTransferId(TransferType transferType) {
    return getAccountTransferId(transferType, false);
}

// Add overloaded method
public String getAccountTransferId(TransferType transferType, boolean isFx) {
    String accountTransferPrefix = getAccountTransferPrefix(transferType);
    
    if (isFx) {
        accountTransferPrefix += "FX";  // e.g., "P2PFX" instead of "P2P"
    }

    String date = new SimpleDateFormat(TRANSFER_DATE_FORMAT).format(getLocalDateTime(new Date()));
    String random = RandomStringUtils.random(6, 0, 0, true, true, null, new SecureRandom());
    return String.format("%s%s.%s", accountTransferPrefix, date, random);
}
```

### 3.2 Add FX Fields to DTOs
- `TransferEventDto`: Add `fxTargetAmount` (BigMoney) and `fxRateSnapshotId` (String).
- `P2PRequestDto` & `TransferDto`: Add `targetCurrency` (String, optional).

### 3.3 `TransferController.subscriberP2p`
Call the overloaded `getAccountTransferId`.

```java
boolean isFx = !sourceCurrency.equals(targetCurrency);

// Do the conversion if FX
if (isFx) {
    FxConversionResult fxResult = fxRateService.convert(requestDto.getAmount(), targetCurrency);
    dto.setFxRateSnapshotId(fxResult.getRateSnapshotId());
    dto.setFxTargetAmount(fxResult.getTargetAmount());
}

// Generate ID with FX prefix flag
dto.setTransferAggregateId(utils.getAccountTransferId(dto.getTransferType(), isFx));
```

### 3.4 `NewWalletService.updateWalletBalances` (The 4-Leg Bridge)

```java
@Transactional
public TransferEventDto updateWalletBalances(TransferEventDto dto) {
    if (dto.getFxRateSnapshotId() != null) {
        // 4-Leg Bridge:
        // Leg 1: Debit Sender (USD)
        // Leg 2: Credit AMT01 (USD)
        // Leg 3: Debit AMT01 (SOS) -> (Rolls back if treasury lacks SOS liquidity)
        // Leg 4: Credit Receiver (SOS)
        return updateWalletBalancesFxBridge(dto);
    }
    return updateWalletBalancesSameCurrency(dto);
}
```

### 3.5 `AccountTransferListener.processTransaction`
Save 4 operations into the DB. The DB `(aggregate_id, currency)` constraint uniquely identifies the correct Treasury wallets.

```java
if (dto.getFxRateSnapshotId() != null) {
    // Leg 1: Sender WITHDRAW (USD)
    saveOperation(a, dto, dto.getFromAccountAggregateId(), WITHDRAW, sourceAmount);

    // Leg 2: Treasury DEPOSIT (USD) → aggregateId="AMT01"
    saveOperation(a, dto, "AMT01", DEPOSIT, sourceAmount);

    // Leg 3: Treasury WITHDRAW (SOS) → aggregateId="AMT01"
    saveOperation(a, dto, "AMT01", WITHDRAW, targetAmount);

    // Leg 4: Receiver DEPOSIT (SOS)
    saveOperation(a, dto, dto.getToAccountAggregateId(), DEPOSIT, targetAmount);
}
```

### 3.6 `@Transient isFxTransfer()` on `AccountTransfer`

```java
@Transient
public boolean isFxTransfer() {
    if (fxRateSnapshotId != null) return true;
    if (transferOperations == null || transferOperations.isEmpty()) return false;
    return transferOperations.stream()
        .map(op -> op.getAmount().getCurrencyUnit().getCode())
        .collect(Collectors.toSet())
        .size() > 1;
}
```

---

## End-to-End Flow Recap

1. **Every 5 mins:** `FxRatePollerJob` fetches 1 USD = 571.50 SOS. Updates `ExchangeRate` row for today (updates `current`, `high`, `low`).
2. **User Requests P2P:** Sender 100 USD -> Receiver SOS.
3. **Controller detects FX:** Calls `FxRateService.convert()`.
4. **Snapshot Created:** `FxRateService` calculates `effective_rate = 559.87`, saves `ExchangeRateSnapshot`, returns snapshot ID.
5. **ID Gen:** Controller calls `utils.getAccountTransferId(TransferType.SUBSCRIBER_TO_SUBSCRIBER_P2P, true)` → outputs `P2PFX260819.1023.xYzAbC`.
6. **Execution:** 4-leg bridge transfers 100 USD to Treasury, takes 55,987 SOS from Treasury, gives to Receiver.
7. **Ledger:** 4 `AccountTransferOperation` rows saved. Treasury takes the spread profit.
