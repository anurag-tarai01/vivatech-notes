```
================================================================================================
                                   FX ENGINE (ASYNCHRONOUS)
================================================================================================
 
 [ External Provider ] (e.g. ExchangeRate-API)
          ^
          | (HTTP GET /latest every 5 mins)
          v
 +-----------------------+
 |   FxRatePollerJob     | (Quartz Scheduled Job)
 |   (Cluster-Safe)      |
 +-----------+-----------+
             | 1. Fetches latest market rates
             | 2. Applies Treasury Spread (e.g. 2%)
             | 3. Deactivates old rates
             v
 +-----------------------+    << TABLE: ExchangeRate >>
 |                       |    id: "01J_RATE_ULID" (Immutable Snapshot ID)
 | ExchangeRateRepository|    base_currency: "USD", target_currency: "SOS"
 |        (DB)           |    rate: 571.50 (Market), spread: 0.02
 |                       |    effective_rate: 582.93 (What user gets)
 +-----------------------+    is_active: true, fetched_at: <timestamp>


================================================================================================
                               SYNCHRONOUS TRANSFER FLOW
================================================================================================

[ User App / Client ]  -- Request Transfer (100 USD to TILL-002) --> [ API Controller ]
                                                                             |
                                                                             v
+-----------------------------------------------------------------------------------------+
| TransferRequestBodyAdvice (or Pre-Dispatch Validator)                                   |
|-----------------------------------------------------------------------------------------|
| 1. Detects cross-currency (Sender Wallet is USD, Receiver Wallet is SOS)                |
| 2. Calls FxRateService.convert(100 USD, "SOS")                                          |
|      -> Queries ExchangeRateRepository for active USD/SOS rate                          |
|      <- Returns FxConversionResult (Amount: 58,293.00 SOS, Snapshot ID: 01J_RATE_ULID)  |
| 3. Injects FX data into TransferEventDto                                                |
+-------------------------------------------+---------------------------------------------+
                                            |
                                            v
                                 [ TransferEventDto ]
                                 - amount: 100 USD (Source Amount)
                                 - fxTargetAmount: 58,293.00 SOS
                                 - fxEffectiveRate: 582.93
                                 - fxRateSnapshotId: "01J_RATE_ULID"
                                 - isFxTransfer: true
                                            |
                                            v
+-----------------------------------------------------------------------------------------+
| GPayAccountTransferService (Main Transfer Engine)                                       |
|-----------------------------------------------------------------------------------------|
| -> executeInterceptorsAndInitiate(dto)                                                  |
|      - Calculates commission                                                            |
|      - Initiates DB transaction (AccountTransfer table)                                 |
|                                                                                         |
| -> executeTransactionCore(dto)                                                          |
|      - newWalletService.updateWalletBalances(dto)                                       |
|            -> DEBITS 100 USD from Sender's USD Wallet                                   |
|            -> CREDITS 58,293.00 SOS to Receiver's SOS Wallet                            |
|                                                                                         |
| -> finishTransaction(dto)                                                               |
|      - Completes transaction in DB                                                      |
+-------------------------------------------+---------------------------------------------+
                                            |
                                            v
+-----------------------------------------------------------------------------------------+
| AccountTransfer Entity (Final DB Record)                                                |
|-----------------------------------------------------------------------------------------|
| - id: "TRANS-999"                                                                       |
| - amount: 100 USD                <- Official requested amount                           |
| - is_fx_transfer: true                                                                  |
| - fx_source_currency: "USD", fx_source_amount: 100.00                                   |
| - fx_target_currency: "SOS", fx_target_amount: 58,293.00                                |
| - fx_effective_rate: 582.93                                                             |
| - fx_rate_snapshot_id: "01J_RATE_ULID"  <- Link back to exact rate used                 |
+-----------------------------------------------------------------------------------------+
```

### Key Takeaways from this Flow:

1. **Decoupled Caching**: The transfer engine NEVER calls the external API. The Quartz job updates the database asynchronously. This guarantees high throughput and 100% uptime for transfers even if the external FX API goes down.
2. **Immutable Snapshots**: The `ExchangeRate` row has an ID (e.g., `01J_RATE_ULID`). This ID is fetched during validation and permanently saved on the `AccountTransfer` record. If a rate updates 2 seconds after the transfer, the transfer's historical record safely points to the exact rate snapshot it used.
3. **Pre-Dispatch Resolution**: The FX conversion happens in the `TransferRequestBodyAdvice` (or a similar validation layer) _before_ `GPayAccountTransferService.processTransaction` is called. This keeps the core transfer engine clean and focused purely on moving money according to the DTO it receives.