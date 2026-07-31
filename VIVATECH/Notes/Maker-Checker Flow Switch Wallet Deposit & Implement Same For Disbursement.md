# Switch-Wallet Deposit Flow → Commission Disbursement Approval Pattern

> A precise trace of the switch-wallet approval/rejection mechanism, mapped to the minimum changes needed to add PAY → PENDING → APPROVE → EXECUTE for commission disbursement. No new entities. No system redesign.

---

## PART 1 — Switch-Wallet Deposit: Exact End-to-End Flow

### The Full State Machine

```
Admin UI (webclient)
     │
     │  POST /transfer/deposit-switch-wallet
     │  { fromAccount: AMT01, toAccount: SWITCH_WALLET, amount: X }
     ▼
SwitchWalletController.switchWalletDeposit()
     │
     │  Maps TransferDto → TransferEventDto
     │  Generates transferAggregateId = "DSW-YYYYMMDD-XXXX"
     │  Calls newAccountTransferService.beginTransaction(dto)
     ▼
NewAccountTransferService.beginTransaction()
     │
     ├─ Scale amount
     ├─ transferValidationService.checkTransactionValidations(dto)
     │    └─ Validate: user status, balance, limits, pincode
     ├─ serviceAndCommissionChargeService.calculateServiceAndCommissionCharge(dto)
     │    └─ (SWITCH_WALLET_DEPOSIT has no service charge, returns 0)
     ├─ accountTransferListener.initiateTransaction(dto)
     │    └─ INSERT account_transfer row:
     │         transferAggregateId = "DSW-YYYYMMDD-XXXX"
     │         transferStatus      = STARTED         ← *** THE PENDING STATE ***
     │         transferType        = SWITCH_WALLET_DEPOSIT
     │         amount              = X
     │         fromAccountId       = AMT01 wallet
     │         toAccountId         = SWITCH_WALLET
     │         transferApprovedById = 0 (null/not yet approved)
     │
     ├─ [NO third-party call for SWITCH_WALLET_DEPOSIT at this stage]
     │    Note: beginTransaction only calls external API for
     │          SWITCH_WALLET_DEPOSIT_MOBILE_MONEY and DEPOSIT_BANK,
     │          not for the base SWITCH_WALLET_DEPOSIT
     │
     ├─ RabbitMQ: publish "AccountTransferCreateEvent"
     │
     └─ Return SUCCESS { "Transaction Processed, Transfer Id: DSW-..." }

══════════════════════════════════════════════════
DB STATE: account_transfer.transfer_status = STARTED
Money has NOT moved yet. Wallets unchanged.
══════════════════════════════════════════════════

Admin UI polls GET /transfer/list-pending-deposit-switch-wallet
     │
     │  Queries: WHERE transfer_type = 'SWITCH_WALLET_DEPOSIT'
     │           AND   transfer_status = 'STARTED'
     ▼
     Admin sees the pending record, clicks Approve or Reject.
```

---

### APPROVE Path

```
Admin clicks Approve on the pending transfer "DSW-YYYYMMDD-XXXX"
     │
     │  POST /transfer/approve-deposit-switch-wallet
     │  { transferAggregateId: "DSW-YYYYMMDD-XXXX", transferInitiedById: adminId }
     ▼
SwitchWalletController.approveSwitchWalletDeposit()
     │
     ├─ Maps TransferDto → TransferEventDto
     ├─ userService.getUserInfoFromIdStatusAndType(adminId, ACTIVE, ADMIN)
     │    └─ Validates approving admin is real + active
     ├─ dto.setTransferApprovedByAggregateId(admin.getAggregateId())
     ├─ dto.setUpdatedAt(now)
     ├─ validateTransferRequestNew(dto)             ← KEY GATE (see below)
     │    └─ if dto.amount == null → reads amount from DB (the original stored amount)
     │    └─ re-validates from/to wallet status and balance
     └─ newAccountTransferService.processTransaction(dto)
     ▼
NewAccountTransferService.processTransaction()
     │
     │  SWITCH_WALLET_DEPOSIT is in the ALREADY_INITIATED list:
     │  → SKIP re-initiation (already in DB as STARTED)
     │
     ├─ checkMetadata(dto)  [no-op for SWITCH_WALLET_DEPOSIT]
     │
     ├─ newWalletService.updateWalletBalances(dto)
     │    └─ AMT01 wallet balance  -= amount     ← MONEY MOVES HERE
     │    └─ SWITCH wallet balance += amount
     │
     ├─ newThirdPartyService.handleThirdPartyService(dto)
     │    └─ [External call if applicable, no-op for basic deposit]
     │
     ├─ finishTransaction(dto):
     │    ├─ accountTransferListener.processTransaction(dto)
     │    │    └─ UPDATE account_transfer: metadata, narration details
     │    ├─ accountTransferListener.updateApprovalDetail(dto)
     │    │    └─ UPDATE account_transfer:
     │    │         transfer_approved_by_id        = adminId
     │    │         transfer_approved_by_aggregate_id = adminAggId
     │    ├─ accountTransferListener.updateTransactionStatus(dto)
     │    │    └─ UPDATE account_transfer:
     │    │         transfer_status = SUCCESS      ← FINAL STATE
     │    ├─ accountTransferListener.completeTransaction(dto)
     │    │    └─ Type-specific post-processing
     │    ├─ commissionInfoListener.saveCommissionData(dto)
     │    └─ promotionListener.handlePromotionalOffer(dto)
     │
     ├─ RabbitMQ: publish "AccountTransferDoneEvent"
     ├─ RabbitMQ: publish "AccountTransferCompletedEvent"
     │
     └─ Return SUCCESS { "Transaction Success, Transfer Id: DSW-..." }

══════════════════════════════════════════════════
FINAL DB STATE:
  account_transfer.transfer_status          = SUCCESS
  account_transfer.transfer_approved_by_id  = adminId
  wallet[AMT01].balance                    -= amount
  wallet[SWITCH_WALLET].balance            += amount
══════════════════════════════════════════════════
```

---

### REJECT Path

```
Admin clicks Reject on the pending transfer "DSW-YYYYMMDD-XXXX"
     │
     │  POST /transfer/reject-deposit-switch-wallet
     │  { transferAggregateId: "DSW-YYYYMMDD-XXXX", transferInitiedById: adminId }
     ▼
SwitchWalletController.rejectSwitchWalletDeposit()
     │
     ├─ Maps TransferDto → TransferEventDto
     ├─ userService.getUserInfoFromIdStatusAndType(adminId, ACTIVE, ADMIN)
     ├─ dto.setTransferApprovedByAggregateId(admin.getAggregateId())
     ├─ dto.setUpdatedAt(now)
     │
     │  NOTE: NO validateTransferRequestNew() call here (unlike approve)
     │
     └─ newAccountTransferService.rejectTransaction(dto)
     ▼
NewAccountTransferService.rejectTransaction()
     │
     ├─ storeFailedTransaction(dto):
     │    ├─ accountTransferListener.updateApprovalDetail(dto)
     │    │    └─ UPDATE account_transfer:
     │    │         transfer_approved_by_id = adminId  (who rejected it)
     │    ├─ newWalletService.setPreviousBalances(dto)
     │    │    └─ reads current wallet balances (no money ever moved)
     │    ├─ fixTransferMetadataProcessor.handleMetadataForInCompleteTransaction(dto)
     │    ├─ accountTransferListener.handleFailedTransaction(dto)
     │    │    └─ UPDATE account_transfer:
     │    │         transfer_status = FAILED          ← FINAL STATE
     │    │         error_reason    = ""
     │    └─ accountTransferListener.updateTransactionStatusForRejectedTransaction(dto)
     │         └─ For SWITCH_WALLET_DEPOSIT: no-op (only CUSTOMER_CARE_TO_AMAL_BANK
     │            gets REJECTED status here; SWITCH_WALLET gets FAILED)
     │
     ├─ RabbitMQ: publish "AccountTransferFailedEvent"
     │
     └─ Return SUCCESS { "Transaction Rejected, Transfer Id: DSW-..." }

══════════════════════════════════════════════════
FINAL DB STATE:
  account_transfer.transfer_status          = FAILED
  account_transfer.transfer_approved_by_id  = adminId
  wallet[AMT01].balance                    = UNCHANGED (no money moved)
  wallet[SWITCH_WALLET].balance            = UNCHANGED
══════════════════════════════════════════════════
```

---

## PART 2 — The Four Key Architectural Insights

### Insight 1: The "PENDING" state IS `TransferStatus.STARTED`

There is no separate PENDING status in this system. `STARTED` is the pending state. Every transfer sits in `account_transfer` with `transfer_status = STARTED` until an admin explicitly approves or rejects it.

```
STARTED  →  (approve)  →  SUCCESS
STARTED  →  (reject)   →  FAILED
```

### Insight 2: `beginTransaction()` = Initiation Only (no money movement)

`beginTransaction()` does three things and stops:

1. Validates the request
2. Writes the DB record with `transfer_status = STARTED`
3. Publishes `AccountTransferCreateEvent` to RabbitMQ

**Money does NOT move here.** Wallets are untouched.

### Insight 3: `processTransaction()` = The Execution Gate

`processTransaction()` is the method that moves money. For "already initiated" types (the `ALREADY_INITIATED_TRANSFER_TYPES` set), it **skips re-initiation** and goes straight to:

- Balance update
- Third-party call
- `finishTransaction()` → `updateTransactionStatus()` → SUCCESS

This is precisely the pattern used for the approve flow.

### Insight 4: The "Already Initiated" Set Is the Approval-Gate Registry

```java
// In GPayAccountTransferService / NewAccountTransferService
private static final Set<TransferType> ALREADY_INITIATED_TRANSFER_TYPES = EnumSet.of(
    ...
    TransferType.SWITCH_WALLET_DEPOSIT,   // ← sits here because it needs approval
    TransferType.SWITCH_WALLET_WITHDRAW,  // ← same
    ...
);
```

Any `TransferType` in this set tells `processTransaction()`: "don't re-run validation and initiation — this was already saved to DB by `beginTransaction()`, just execute the money movement."

**Adding your new `COMMISSION_DISBURSEMENT_BATCH` type to this set is the central enabler of the approval flow.**

---

## PART 3 — The Decision Gate: `validateTransferRequestNew()`

Called in the approve path but NOT in the reject path. This is important.

For the approval case, when `dto.amount == null` (the front-end only sends the `transferAggregateId`, not the amount again):

```java
if (amount == null) { // approval transfer request
    AccountTransfer accountTransfer = accountTransfeRepository.findOne(transferId);
    fromAccountAggregateId = accountTransfer.getFromAccountAggregateId();
    toAccountAggregateId   = accountTransfer.getToAccountAggregateId();
    dto.setAmount(accountTransfer.getAmount()); // re-hydrate from DB
}
```

It re-reads the original amounts from DB and re-validates wallet status and balance at approval time. This prevents approving a transfer where the source wallet has since been blocked or emptied.

---

## PART 4 — Mapping to Commission Disbursement

### Current (Broken) Flow

```
Admin clicks Pay (UI)
    → POST /admin/commission-disbursement/execute/{transferType}
    → processDisbursement() runs IMMEDIATELY
    → Money moves RIGHT NOW
    No pending. No approval. No audit trail.
```

### Target Flow (Switch-Wallet Pattern)

```
Admin clicks Pay (UI)
    → POST /transfer/initiate-commission-disbursement/{transferType}     ← NEW
    → beginTransaction()    → DB: transfer_status = STARTED             ← PENDING
    → Admin 2 sees it in pending list
    → Admin 2 clicks Approve
    → POST /transfer/approve-commission-disbursement                     ← NEW
    → processTransaction()  → processDisbursement() → wallets updated  ← EXECUTION
    → DB: transfer_status = SUCCESS
```

---

## PART 5 — Minimal Change Strategy: What to Add

### Step 1: Add a New TransferType

Add one entry to the `TransferType` enum:

```java
COMMISSION_DISBURSEMENT_BATCH,
```

This represents "an admin has requested commission disbursement for a transfer type — pending approval."

This type is the container/token for the approval workflow. It is distinct from the individual `COMMISSION_DISBURSEMENT_TO_AGENT` etc. which are the actual child transfers fired by `processDisbursement()`.

---

### Step 2: Register It in `ALREADY_INITIATED_TRANSFER_TYPES`

In **`GPayAccountTransferService`** (and the parallel list in `NewAccountTransferService.processTransaction()`):

```java
private static final Set<TransferType> ALREADY_INITIATED_TRANSFER_TYPES = EnumSet.of(
    // ... existing entries ...
    TransferType.SWITCH_WALLET_DEPOSIT,
    TransferType.SWITCH_WALLET_WITHDRAW,
    TransferType.COMMISSION_DISBURSEMENT_BATCH,   // ← ADD THIS
    TransferType.AMT02_TO_AMT01
);
```

This tells `processTransaction()`: skip re-initiation, go straight to execution.

---

### Step 3: Register It in `updateApprovalDetail()`

In **`AccountTransferListener.updateApprovalDetail()`**, add the new type to the list that saves who approved it:

```java
|| dto.getTransferType().equals(TransferType.COMMISSION_DISBURSEMENT_BATCH)
```

---

### Step 4: Register It in `updateTransactionStatus()`

In **`AccountTransferListener.updateTransactionStatus()`**, add the new type so it gets marked `SUCCESS` on completion:

```java
|| dto.getTransferType().equals(TransferType.COMMISSION_DISBURSEMENT_BATCH)
```

---

### Step 5: Add Two New Endpoints to CommissionDisbursementExecutionController

```java
@RestController
@RequestMapping("/transfer")          // ← same base path as SwitchWalletController
@Slf4j
public class CommissionDisbursementExecutionController extends BaseController {

    @Autowired private CommissionDisbursementProcessingService processingService;
    @Autowired private MfsUtils utils;
    @Autowired private UserService userService;
    @Autowired private GPayAccountTransferService gPayAccountTransferService;
    @Autowired private AccountTransferQueryRepository accountTransferQueryRepository;

    /**
     * STEP 1: Initiate disbursement (replaces the direct execute).
     * Creates a PENDING transfer record. No money moves.
     * Mirrors: SwitchWalletController.switchWalletDeposit() → beginTransaction()
     */
    @RequestMapping(value = "/initiate-commission-disbursement/{transferType}", method = RequestMethod.POST)
    public BaseResponseDto initiateDisbursement(
            @PathVariable TransferType transferType,
            @RequestParam Integer initiatedById) {

        try {
            String batchTransferId = utils.getAccountTransferId(TransferType.COMMISSION_DISBURSEMENT_BATCH);

            TransferEventDto dto = TransferEventDto.builder()
                    .transferAggregateId(batchTransferId)
                    .fromAccountAggregateId(Constants.AMT03_WALLET_AGGREGATE_ID)
                    .toAccountAggregateId(Constants.AMT03_WALLET_AGGREGATE_ID) // placeholder; actual targets determined at execution
                    .fromAccountId(Constants.AMT03_WALLET_AGGREGATE_ID)
                    .toAccountId(Constants.AMT03_WALLET_AGGREGATE_ID)
                    .amount(utils.getMoney(0))               // placeholder; actual per-commission amounts computed at execution
                    .serviceCharge(utils.getMoney(0))
                    .transferType(TransferType.COMMISSION_DISBURSEMENT_BATCH)
                    .transferInitietedByType(TransferInitietedByType.ADMIN)
                    .transferInitiedByAggregateId(userService.getUserInfoFromId(initiatedById).getAggregateId())
                    .transferInitiedById(initiatedById)
                    .transferStatus(TransferStatus.STARTED)
                    .createdAt(new Date())
                    // Store the real disbursement transfer type in metadata so the approve step knows what to run
                    .transferMetadata(transferType.name())
                    .build();

            // beginTransaction: validates, writes STARTED to DB, publishes CreateEvent
            return gPayAccountTransferService.beginTransaction(dto);

        } catch (Exception e) {
            log.error("Failed to initiate commission disbursement", e);
            return BaseResponseDto.builder().status("FAILED").message(e.getMessage()).build();
        }
    }

    /**
     * STEP 2: List pending disbursement requests.
     * Mirrors: SwitchWalletController.listPendingDepositSwitchWallet()
     */
    @RequestMapping(value = "/list-pending-commission-disbursement", method = RequestMethod.GET)
    public Object listPendingDisbursements(@RequestParam(required = false) String page) {
        PageRequest pageRequest = Utility.getPageRequestObj(page);
        return accountTransferQueryRepository.findAllByTransferTypeAndTransferStatus(
                TransferType.COMMISSION_DISBURSEMENT_BATCH, TransferStatus.STARTED, pageRequest);
    }

    /**
     * STEP 3: Approve — triggers actual money movement.
     * Mirrors: SwitchWalletController.approveSwitchWalletDeposit() → processTransaction()
     */
    @RequestMapping(value = "/approve-commission-disbursement", method = RequestMethod.POST)
    public BaseResponseDto approveDisbursement(@RequestBody TransferDto requestDto) {

        try {
            // Load the pending batch transfer record from DB
            AccountTransfer batchTransfer = accountTransferQueryRepository
                    .findOne(requestDto.getTransferAggregateId());

            if (batchTransfer == null || batchTransfer.getTransferStatus() != TransferStatus.STARTED) {
                return BaseResponseDto.builder()
                        .status("FAILED")
                        .message("Pending disbursement not found or already processed.")
                        .build();
            }

            // Validate approving admin
            UserInfo admin = userService.getUserInfoFromIdStatusAndType(
                    requestDto.getTransferInitiedById(), UserStatus.ACTIVE, UserType.ADMIN);

            // Recover the target disbursement transfer type from metadata
            TransferType disbursementTransferType = TransferType.valueOf(batchTransfer.getTransferMetadata());

            // Build the DTO for processTransaction
            TransferEventDto dto = TransferEventDto.builder()
                    .transferAggregateId(batchTransfer.getTransferAggregateId())
                    .fromAccountAggregateId(batchTransfer.getFromAccountAggregateId())
                    .toAccountAggregateId(batchTransfer.getToAccountAggregateId())
                    .fromAccountId(batchTransfer.getFromAccountId())
                    .toAccountId(batchTransfer.getToAccountId())
                    .amount(batchTransfer.getAmount())
                    .transferType(TransferType.COMMISSION_DISBURSEMENT_BATCH)
                    .transferInitietedByType(TransferInitietedByType.ADMIN)
                    .transferInitiedById(batchTransfer.getTransferInitiedById())
                    .transferApprovedById(admin.getUserId())
                    .transferApprovedByAggregateId(admin.getAggregateId())
                    .transferMetadata(batchTransfer.getTransferMetadata())
                    .updatedAt(new Date())
                    .build();

            // *** THE ACTUAL EXECUTION ***
            // processTransaction will skip re-initiation (ALREADY_INITIATED_TRANSFER_TYPES)
            // and go straight to finishTransaction() which calls our hook below.
            // We override executeTransactionCore by intercepting it — or we handle
            // it inside finishTransaction via a new completeTransaction() case:
            processingService.processDisbursement(disbursementTransferType);  // ← the real work

            // Now mark the batch record as completed
            return gPayAccountTransferService.processTransaction(dto);

        } catch (Exception e) {
            log.error("Failed to approve commission disbursement", e);
            return BaseResponseDto.builder().status("FAILED").message(e.getMessage()).build();
        }
    }

    /**
     * STEP 4: Reject.
     * Mirrors: SwitchWalletController.rejectSwitchWalletDeposit() → rejectTransaction()
     */
    @RequestMapping(value = "/reject-commission-disbursement", method = RequestMethod.POST)
    public BaseResponseDto rejectDisbursement(@RequestBody TransferDto requestDto) {

        try {
            UserInfo admin = userService.getUserInfoFromIdStatusAndType(
                    requestDto.getTransferInitiedById(), UserStatus.ACTIVE, UserType.ADMIN);

            AccountTransfer batchTransfer = accountTransferQueryRepository
                    .findOne(requestDto.getTransferAggregateId());

            TransferEventDto dto = TransferEventDto.builder()
                    .transferAggregateId(batchTransfer.getTransferAggregateId())
                    .fromAccountAggregateId(batchTransfer.getFromAccountAggregateId())
                    .toAccountAggregateId(batchTransfer.getToAccountAggregateId())
                    .fromAccountId(batchTransfer.getFromAccountId())
                    .toAccountId(batchTransfer.getToAccountId())
                    .amount(batchTransfer.getAmount())
                    .transferType(TransferType.COMMISSION_DISBURSEMENT_BATCH)
                    .transferApprovedById(admin.getUserId())
                    .transferApprovedByAggregateId(admin.getAggregateId())
                    .transferMetadata(batchTransfer.getTransferMetadata())
                    .updatedAt(new Date())
                    .build();

            // rejectTransaction: marks FAILED, records who rejected, fires FailedEvent
            return gPayAccountTransferService.rejectTransaction(dto);

        } catch (Exception e) {
            log.error("Failed to reject commission disbursement", e);
            return BaseResponseDto.builder().status("FAILED").message(e.getMessage()).build();
        }
    }
}
```

---

### Step 6: Update the UI

The existing UI HTML already has the "Pay" button wired to `POST /admin/commission-disbursement/execute/{transferType}`.

Change that to call the new initiation endpoint instead:

```javascript
// BEFORE (current — executes immediately):
url: '/admin/commission-disbursement/execute/' + transferType

// AFTER (new — creates pending record):
url: '/transfer/initiate-commission-disbursement/' + transferType
```

Add a new "Pending Disbursements" section that mirrors the switch-wallet pending list, with Approve and Reject buttons that POST to the new endpoints.

---

## PART 6 — Complete Side-by-Side Comparison

|Aspect|Switch-Wallet Deposit|Commission Disbursement (after change)|
|---|---|---|
|Initiation endpoint|`POST /transfer/deposit-switch-wallet`|`POST /transfer/initiate-commission-disbursement/{type}`|
|Initiation method|`beginTransaction()`|`gPayAccountTransferService.beginTransaction()`|
|DB state after initiation|`transfer_status = STARTED`|`transfer_status = STARTED`|
|Money moved at initiation?|**NO**|**NO**|
|Pending list endpoint|`GET /transfer/list-pending-deposit-switch-wallet`|`GET /transfer/list-pending-commission-disbursement`|
|Pending list query|`WHERE type='SWITCH_WALLET_DEPOSIT' AND status='STARTED'`|`WHERE type='COMMISSION_DISBURSEMENT_BATCH' AND status='STARTED'`|
|Approve endpoint|`POST /transfer/approve-deposit-switch-wallet`|`POST /transfer/approve-commission-disbursement`|
|Approve method|`processTransaction()`|`processTransaction()` + `processDisbursement()`|
|What approve does|Moves AMT01→SWITCH wallet|Runs all individual disbursement transfers|
|DB state after approve|`transfer_status = SUCCESS`|`transfer_status = SUCCESS`|
|Reject endpoint|`POST /transfer/reject-deposit-switch-wallet`|`POST /transfer/reject-commission-disbursement`|
|Reject method|`rejectTransaction()`|`gPayAccountTransferService.rejectTransaction()`|
|DB state after reject|`transfer_status = FAILED`|`transfer_status = FAILED`|
|Money moved at reject?|**NO**|**NO**|
|RabbitMQ on initiate|`AccountTransferCreateEvent`|`AccountTransferCreateEvent`|
|RabbitMQ on approve|`AccountTransferDoneEvent` + `AccountTransferCompletedEvent`|`AccountTransferCompletedEvent`|
|RabbitMQ on reject|`AccountTransferFailedEvent`|`AccountTransferFailedEvent`|
|Approval audit trail|`transfer_approved_by_id` in `account_transfer`|Same — `updateApprovalDetail()` writes it|

---

## PART 7 — Exact Files to Change (Checklist)

|File|Change|
|---|---|
|`TransferType.java` (enum)|Add `COMMISSION_DISBURSEMENT_BATCH`|
|`MfsUtils.java`|Add prefix for the new type (e.g., `"CDB"`) in prefix map|
|`GPayAccountTransferService.java`|Add `COMMISSION_DISBURSEMENT_BATCH` to `ALREADY_INITIATED_TRANSFER_TYPES`|
|`NewAccountTransferService.java`|Add `COMMISSION_DISBURSEMENT_BATCH` to the already-initiated `if` condition in `processTransaction()`|
|`AccountTransferListener.updateApprovalDetail()`|Add `COMMISSION_DISBURSEMENT_BATCH` to the list|
|`AccountTransferListener.updateTransactionStatus()`|Add `COMMISSION_DISBURSEMENT_BATCH` to the list|
|`CommissionDisbursementExecutionController.java`|Replace single `/execute` with four new endpoints as shown above|
|UI HTML|Change Pay button URL; add pending list + approve/reject buttons|

**Total: 8 files.** No new database tables. No new entities. No redesign.

---

## PART 8 — Key Design Note on the Approval Execute Order

In the approve endpoint, the order matters:

```java
// 1. Run the actual disbursements first (child transfers)
processingService.processDisbursement(disbursementTransferType);

// 2. Mark the batch record as SUCCESS
gPayAccountTransferService.processTransaction(dto);
```

This mirrors how switch-wallet works internally: the real work happens inside `executeTransactionCore()` before `finishTransaction()` marks the parent as SUCCESS. If `processDisbursement()` throws, the batch transfer stays STARTED (or you catch and call `rejectTransaction()` to mark it FAILED), preserving the ability to retry.

This is the correct failure semantics: the parent batch record only becomes SUCCESS after all child disbursement transfers have been attempted.