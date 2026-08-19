**

Cross-Currency Transfer (FX) — Implementation Plan

# 1. Objective

Enable cross-currency transfers in the existing MFS transfer flow while reusing the current TransferType, transfer processing, and transaction infrastructure as much as possible.

For example:

Sender:   100 USD

Receiver: SOS wallet

  

The system will convert the source amount using the latest available FX rate, apply the configured platform spread, and settle the transfer through Treasury wallets.

---

# 2. Confirmed Design Decisions

|   |   |
|---|---|
|Area|Decision|
|Currency Source|Read localCurrency and foreignCurrency from active Domain entities. No additional currency mapping table.|
|FX Rate Storage|ExchangeRate for daily market tracking and ExchangeRateSnapshot for immutable transfer-level rates.|
|TransferType|Reuse existing transfer types.|
|Transaction ID|Append FX to the existing prefix for FX transactions, e.g. P2PFX....|
|FX Identification|Add fx_rate_snapshot_id to AccountTransfer.|
|Treasury Wallet|Use aggregateId = "AMT01" and resolve the wallet by currency.|
|P2P FX Resolution|Handle FX detection and conversion in TransferController.subscriberP2p().|
|Same-Currency Transfer|Continue using the existing flow without FX processing.|

---

# 3. Database Changes

## 3.1 exchange_rate

Stores daily market rates for each currency pair.

Main fields:

- id
    
- base_currency
    
- target_currency
    
- opening_rate
    
- closing_rate
    
- high_rate
    
- low_rate
    
- current_rate
    
- source
    
- rate_date
    
- created_at
    
- updated_at
    

The FX poller will update this record every 5 minutes.

For the first rate of the day:

opening = current = high = low = fetched rate

  

For subsequent updates:

current = latest rate

high    = highest rate of the day

low     = lowest rate of the day

  

Unique lookup:

base_currency + target_currency + rate_date

  

---

## 3.2 exchange_rate_snapshot

Stores the exact rate used for an FX transfer.

Main fields:

- id
    
- exchange_rate_id
    
- base_currency
    
- target_currency
    
- market_rate
    
- spread_rate
    
- effective_rate
    
- inverse_rate
    
- created_at
    

A snapshot is created once per FX transfer and is never updated.

This ensures that historical transfers always retain the exact rate used during execution.

---

## 3.3 account_transfer

Add:

fx_rate_snapshot_id

  

Behaviour:

- NULL → Same-currency transfer
    
- Value present → FX transfer
    

---

# 4. FX Rate Management

## 4.1 FX Poller

A Quartz job will run every 5 minutes.

Responsibilities:

1. Read active domains.
    
2. Collect supported currencies.
    
3. Determine required currency pairs.
    
4. Fetch rates from the external FX provider.
    
5. Create or update daily ExchangeRate records.
    

---

## 4.2 FX Conversion Service

A new FxRateService will:

1. Retrieve the current market rate.
    
2. Apply the configured spread.
    
3. Calculate the effective customer rate.
    
4. Calculate the target amount.
    
5. Create an immutable ExchangeRateSnapshot.
    
6. Return the converted amount and snapshot ID.
    

Example:

Market Rate:    1 USD = 571.50 SOS

Spread:         2%

Effective Rate: 1 USD = 559.87 SOS

  

100 USD → 55,987 SOS

  

The transfer will reference the generated snapshot ID.

---

# 5. Transfer Flow Changes

## 5.1 FX Detection

For P2P transfers, compare:

Source Currency vs Target Currency

  

If both currencies are the same:

Use existing transfer flow

  

If currencies differ:

Execute FX conversion flow

  

P2P FX detection will be implemented in:

TransferController.subscriberP2p()

  

because the existing request advice does not cover P2PRequestDto.

---

## 5.2 Transaction ID

Existing TransferType values will remain unchanged.

An overloaded transaction ID method will accept an FX flag.

Example:

Normal P2P: P2P...

FX P2P:     P2PFX...

  

This avoids creating duplicate transfer types and changing existing validators and listeners.

---

## 5.3 DTO Changes

### P2PRequestDto / TransferDto

Add:

targetCurrency

  

### TransferEventDto

Add:

fxTargetAmount

fxRateSnapshotId

  

These fields will carry FX information through the existing transfer flow.

---

# 6. FX Wallet Settlement

FX transfers will use the following settlement flow. The current implementation scope includes the first four legs. FX commission processing (Leg 5) is reserved for a future phase. 

Example:

Sender sends: 100 USD

Receiver gets: 55,987 SOS

  

Flow:

1. Sender Wallet       → Debit 100 USD

2. Treasury AMT01 USD  → Credit 100 USD

3. Treasury AMT01 SOS  → Debit 55,987 SOS

4. Receiver Wallet     → Credit 55,987 SOS

  

Treasury wallets will use the same aggregate ID:

AMT01

  

The correct Treasury wallet will be identified by:

aggregateId + currency

  

All four operations must be transactional. If the Treasury does not have sufficient target-currency liquidity, the complete transfer must fail and roll back.

---

# 7. Account Transfer Processing

For an FX transfer:

- AccountTransfer stores fx_rate_snapshot_id.
    
- Four AccountTransferOperation records are created.
    
- Wallet balance processing uses the FX four-leg bridge.
    

For a same-currency transfer:

- fx_rate_snapshot_id remains NULL.
    
- Existing processing remains unchanged.
    

A transient isFxTransfer() method can determine FX status using the snapshot ID and, where required, validate based on operation currencies.

---

# 8. Components Affected

### New

- ExchangeRate
    
- ExchangeRateRepository
    
- ExchangeRateSnapshot
    
- ExchangeRateSnapshotRepository
    
- FxRateService
    
- FxRatePersistenceService
    
- FxRatePollerJob
    

### Modified

- MfsUtils
    
- TransferController
    
- P2PRequestDto
    
- TransferDto
    
- TransferEventDto
    
- NewWalletService
    
- AccountTransferListener
    
- AccountTransfer
    

---

# 9. End-to-End Flow

FX Provider

    ↓

FxRatePollerJob (Every 5 Minutes)

    ↓

ExchangeRate

    ↓

User Initiates Transfer

    ↓

Compare Source and Target Currency

    ↓

Same Currency ─────────────> Existing Transfer Flow

  

Different Currency

    ↓

FxRateService

    ↓

Calculate Effective Rate + Create Snapshot

    ↓

Generate FX Transaction ID

    ↓

4-Leg Treasury Settlement

    ↓

Persist AccountTransfer + 4 Operations

  

---

# 10. Expected Outcome

The implementation will provide:

- External FX rate integration.
    
- Periodic market rate updates.
    
- Immutable rate snapshots per transfer.
    
- Configurable platform spread.
    
- FX transaction identification using existing transfer types.
    
- Treasury-based cross-currency settlement.
    
- Full transaction auditability.
    
- Backward compatibility with existing same-currency transfers.
    

  
  
**