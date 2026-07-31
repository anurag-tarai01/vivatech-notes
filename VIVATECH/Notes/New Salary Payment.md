# `/third-party/salary-payment-new` — Complete Execution Flow Analysis

---

## Request & Response DTOs

**Request:** `PayrollVerificationRequestDto`

- Used fields: `submittedById`, `uuid`
- The `uuid` is a pre-created payroll batch identifier (from a prior `/create-payroll` call)

**Response:** `TransferResponseDto`

- Always returns: `statusCode = PROCESSING`, `message = "Salary Payment is being processed."`
- This fixed response is returned regardless of per-employee outcomes; there is no per-employee success/failure aggregation in the HTTP response

---

## Step-by-Step Execution Trace

### Step 1 — `TransferController.salaryPaymentNew()`

**Class:** `TransferController`  
**Entry point:** `POST /third-party/salary-payment-new`

**What it does:**

1. Calls `adminUserQueryRepository.findOne(requestDto.getSubmittedById())`  
    → DB read on `Admin` table by primary key
2. Calls `thirdPartyService.getThirdPartyFromAdminId(requestDto.getSubmittedById())`  
    → `ThirdPartyAdminQueryRepository.findOne(adminId)` → get `thirdPartyAggregateId`  
    → `ThirdPartyQueryRepository.findByThirdPartyAggregateId(...)` → returns `ThirdParty` entity
3. Calls `salaryPaymentRequestRepository.findByUuid(requestDto.getUuid())`  
    → DB read on `salary_payment_request` table by UUID  
    → Returns `SalaryPaymentRequest` (the batch record created in a previous step)
4. Calls `thirdPartyService.getSalaryPayeeInfosNew(payroll)`  
    → Validates `payroll != null` and `payroll.getStatus() == FileStatus.PENDING`; throws `MFSException` if either fails  
    → Calls `salaryPaymentRepository.findAllByBulkPaymentId(payroll.getUuid())`  
    → Iterates each `SalaryPayment` row and maps to `SalaryPayeeInfo` (name, msisdn, amount, paymentType, paymentMode, switchPaymentType, orgCode, service)
5. **Branch on `payroll.getFileUrl()`:**
    - `null` → calls `processBulkSalaryPay(infos, thirdParty, admin, requestDto)`
    - non-null → calls `processBulkIncentivePay(infos, thirdParty, admin, requestDto)`
6. After whichever bulk loop completes, calls `thirdPartyService.approvePayroll(uuid, submittedById)`  
    → Loads `SalaryPaymentRequest` again by UUID, sets `status = APPROVED`, saves
7. Returns `TransferResponseDto(PROCESSING, "Salary Payment is being processed.")`

---

### Step 2A — `processBulkSalaryPay()` (when `payroll.fileUrl == null`)

**Class:** `TransferController` (private method)

Iterates over each `SalaryPayeeInfo`. For each entry:

**2A-1.** Re-fetches the payer wallet on every iteration:  
`walletService.getWalletOfUserByIdAndTypeFromRepo(admin.getId(), WalletType.THIRD_PARTY)`  
→ `WalletQueryRepository.findOneByUserIdAndType(userId, type)` — direct DB hit every loop

**2A-2.** Looks up employee:  
`employeeRepository.findByMsisdnAndThirdPartyAndStatus(msisdn, thirdParty, UserStatus.ACTIVE)`  
→ If `employee == null`, **skips this payee silently** (no error logged, no SalaryPayment status updated)

**2A-3.** Amount calculation (only when `paymentType == "Salary"`):

- If `employee.advancePayment > 0` and `advancePaymentStatus == ACTIVE`:  
    `amount = max(employee.salary - employee.advancePayment, 0)`
- Otherwise: `amount = salaryPayeeInfo.getAmount()` (from the SalaryPayment row)
- For any other paymentType (Bonus, Commission): `amount = salaryPayeeInfo.getAmount()` directly

**2A-4.** `toWallet` resolution:

- `PaymentMode.WALLET` → calls `userService.getUserInfoFromMsisdn(internationalMsisdn)` → calls `walletService.getWalletOfUserByIdAndTypeFromRepo(userId, WalletType.SUBSCRIBER)`
- `PaymentMode.SWITCH` (or other) → calls `walletService.getWalletFromWalletRepoByWalletId(Constants.SWITCH_WALLET)`  
    → `WalletQueryRepository.findByAggregateId(Constants.SWITCH_WALLET)`

**2A-5.** Transfer type assignment from `salaryPayeeInfo.getPaymentType()`:

- `"Salary"` → `TransferType.SALARY_PAYMENT` (transferId prefix: `"SAP"`)
- `"Bonus"` → `TransferType.EMPLOYEE_BONUS` (transferId prefix: `"EMB"`)
- anything else → `TransferType.EMPLOYEE_COMMISSION` (transferId prefix: `"EMC"`)

**2A-6.** Transfer ID generation:  
`utils.getAccountTransferId(transferType)` — uses `prefixMap` keyed by `TransferType`, appends a ULID

**2A-7.** `SwitchWalletDisburseRequest` built:

- If `accountTypeCode == "B"`: includes `orgCode` and `service` in `BeneficiaryAccount`
- Otherwise: simpler `BeneficiaryAccount` without bank fields
- `callBackURL + "/switch/webhook/disburse"` is set as the callback URL

**2A-8.** `TransferEventDto` built with:

- `fromAccountAggregateId/Id` = payer (third-party) wallet ID
- `toAccountAggregateId/Id` = subscriber wallet or `Constants.SWITCH_WALLET`
- `amount` = calculated amount
- `serviceCharge = 0` (hardcoded)
- `transferType = SALARY_PAYMENT` (hardcoded — NOTE: overrides the type set above)
- `channelType = WEB` (bypasses pin-code validation in validators)
- `transferInitietedByType = ADMIN`
- `commissionInfos = []` (empty list)
- `acquiredCommission = 0`
- `isWalletPayment = (paymentMode == WALLET)`
- `lastTransferId = lastTransaction` (tracks chain of transfers for optimistic locking)

**2A-9.** Metadata serialization:

- `dto.setTransferMetadata(utils.serializeMetadata(switchWalletDisburseRequest))` — JSON of the Switch disburse payload
- `dto.setSalaryMetadata(utils.serializeMetadata(SalaryPaymentMetadata))` — JSON with thirdPartyId, thirdPartyName, paymentId (UUID)

**2A-10.** Dispatch:

- `PaymentMode.WALLET` → `gPayAccountTransferService.processTransaction(dto)` (synchronous flow)
- `PaymentMode.SWITCH` → `dto.setTransferType(...)` overwritten again:
    - `accountTypeCode == "B"` → `SWITCH_WALLET_DEPOSIT_BANK`
    - otherwise → `SWITCH_WALLET_DEPOSIT_MOBILE_MONEY`
    - Then calls `gPayAccountTransferService.beginTransaction(dto)` (async/webhook flow)

**2A-11.** `lastTransaction` tracking:  
If `responseDto.getStatus().equalsIgnoreCase("success")` → `lastTransaction = responseDto.getEntityId()`  
(The entity ID is the transferAggregateId if prefix is SAP/EMB/EMC, else null)

**2A-12.** Testing mode check:  
`if (switchWalletTesting && paymentMode == SWITCH)` → calls `switchWalletCommandService.disburseDemo(transferId)`  
→ Directly invokes `SwitchWalletHookController.handleDisburseCallback(payload)` in-process

**2A-13.** `Thread.sleep(salaryPaymentThreadSleep)` — configurable via `${salary.payment.thread.sleep}`, called after every single employee in the loop

---

### Step 2B — `processBulkIncentivePay()` (when `payroll.fileUrl != null`)

**Class:** `TransferController` (private method)

Mostly identical to `processBulkSalaryPay` with these differences:

- Does **not** look up the Employee entity — uses `SalaryPayeeInfo` fields directly
- Amount is always `salaryPayeeInfo.getAmount()` — no advance-payment deduction logic
- `SalaryPaymentMetadata` includes `msisdn` field (for per-row lookup later)
- `toWallet` resolution uses `salaryPayeeInfo.getPaymentMode()` instead of `employee.getPaymentMode()`
- `switchWalletDisburseRequest` uses `salaryPayeeInfo.getName()/getMsisdn()` instead of `employee.getName()/getMsisdn()`

---

### Step 3 — `GPayAccountTransferService.processTransaction()` (WALLET path)

**Class:** `GPayAccountTransferService`

**3-1.** Checks `requiresInitialization(transferType)`:  
SALARY_PAYMENT, EMPLOYEE_BONUS, EMPLOYEE_COMMISSION are **not** in `ALREADY_INITIATED_TRANSFER_TYPES`  
→ Therefore initialization runs

**3-2.** Calls `executeInterceptorsAndInitiate(dto)`:

**3-2a.** Amount scaling: `MfsUtils.getScaledMoney(dto.getAmount())`

**3-2b.** Validation: `transferValidationService.checkTransactionValidations(dto)`  
→ `TransferValidatorService.validate(dto)`:

- Loads `fromWallet` and `toWallet` directly from DB: `walletService.getWalletInfoFromDB(walletId)`
- Loads `fromUser` and `toUser`: `userService.getUserInfoFromId(wallet.getUserId())`
- Builds `FromAccountInfo` and `ToAccountInfo`
- Runs all registered `AbstractTransferValidationRule` implementations in order
- Relevant rules for this flow: `UserStatusValidationRule`, `AccountBalanceLimitValidaionRule`, `MaxMinBalanceLimitValidationRule` (pin validation is bypassed because `channelType = WEB`)
- On failure: sets `dto.errorReason` and clears pincode

**3-2c.** Commission/service charge: `serviceAndCommissionChargeService.calculateServiceAndCommissionCharge(dto)`:

- Transfer type is `SALARY_PAYMENT`, `EMPLOYEE_BONUS`, or `EMPLOYEE_COMMISSION`
- None of these are in `isServiceChargeApplicable()`'s list
- → `setServiceChargeAndCommission` → `utils.isServiceChargeApplicable(service)` returns `false`
- → `dto.serviceCharge = 0`, `dto.commissionInfos = []`
- No service charge or commission is ever calculated for salary payments

**3-2d.** DB initiation: `accountTransferListener.initiateTransaction(dto)`:

- Checks if `AccountTransfer` already exists by transferId → skips if so
- Creates `AccountTransfer` entity with `status = STARTED`
- For SAP/EMB/EMC prefixes: sets `transfer.transferMetadata = dto.getSalaryMetadata()` and `transfer.reversedTransferId = dto.getLastTransferId()` (the previous transfer's ID)
- Calls `saveMetadata(dto, a)`: creates `AccountTransferMetadata` row, saves with `saveAndFlush`
- Saves `AccountTransfer` with `saveAndFlush` → `AccountTransferQueryRepository`

**3-2e.** After initiation: `rabbitMQProducer.rabbitProducer(dto)` with `eventType = "AccountTransferCreateEvent"`

**3-3.** If `dto.errorReason` not empty → `handleFailedTransactionAndReturnResponse(dto)` → marks failed in DB, sends RabbitMQ event

**3-4.** Otherwise: `executeTransactionCore(dto)`:

**3-4a.** `checkMetadata(dto)`: only checks `COMMISSION_PAYMENT` and `ACCOUNT_MANAGER_COMMISSION` types — does nothing for salary types

**3-4b.** `newWalletService.updateWalletBalances(dto)` (marked `@Transactional`):

- `totalWithdrawAmount = amount + serviceCharge` (serviceCharge = 0, so totalWithdrawAmount = amount)
- Reads current balance: `getCurrentWalletBalance(fromWallet)` → `walletQueryRepository.findByAggregateId(...)`
- Runs optimistic-locking debit: `walletQueryRepository.withdrawFromWallet(walletId, amount, date, newTransferId, oldTransferId, lastTransferId)`  
    → A single UPDATE with WHERE clause checking `last_transfer_id` — throws `MFSException` if `rowAffected != 1`
- Sends RabbitMQ message: `MoneySubtractedEvent`
- Reads current balance of toWallet
- Runs credit: `walletQueryRepository.depositToWallet(toWalletId, amount, date)`
- Sends RabbitMQ message: `MoneyAddedEvent`
- Calls `handleCommissions(dto, ...)` — does nothing for salary types (no commissions)

**3-4c.** `newThirdPartyService.handleThirdPartyService(dto)`:

- For WALLET path, `transferType` is `SALARY_PAYMENT`
- This type has no `case` in the switch — hits `default: break`
- No external API call is made for wallet-mode salary payments

**3-4d.** `finishTransaction(dto)` (marked `@Transactional`):

- `accountTransferListener.processTransaction(dto)`:
    - Sets `AccountTransfer.status = DONE`
    - Creates `AccountTransferOperation` for WITHDRAW from `fromWallet`
    - Creates `AccountTransferOperation` for DEPOSIT to `toWallet`
    - Saves both operations and the AccountTransfer
    - No special-case handling for SALARY_PAYMENT in `processTransaction()`
- `accountTransferListener.updateApprovalDetail(dto)`: SALARY_PAYMENT not in the list → no-op
- `accountTransferListener.updateTransactionStatus(dto)`: SALARY_PAYMENT **is** in the list → sets `AccountTransfer.status = SUCCESS`
- `accountTransferListener.completeTransaction(dto)`:
    - Checks `transferAggregateId.startsWith("SAP")` → calls `processSalaryAndAdvancePayment(dto)` (see Step 4)
    - Checks `startsWith("EMB")` or `startsWith("EMC")` → calls `processIncentivePayment(dto)` (see Step 5)
    - Saves commission operations (none for salary)
- `commissionInfoListener.saveCommissionData(dto)`: no-op (commissionInfos is empty)
- `promotionListener.handlePromotionalOffer(dto)`: runs promotion check
- `fixTransferMetadataProcessor.handleMetadataForCompleteTransaction(dto)`: metadata post-processing

**3-4e.** On success: sends RabbitMQ message with `eventType = "AccountTransferCompletedEvent"`, fetches User by `transferInitiedById` to set aggregateId

**3-4f.** Returns `BaseResponseDto(SUCCESS, "Transaction Success, Transfer Id: SAP...")`

---

### Step 3B — `GPayAccountTransferService.beginTransaction()` (SWITCH path)

**Class:** `GPayAccountTransferService`

**3B-1.** Calls `executeInterceptorsAndInitiate(dto)` — same as Step 3-2 above:

- Amount scaling, validation, commission calculation (zero), DB initiation, RabbitMQ event

**3B-2.** If `transferType == SWITCH_WALLET_DEPOSIT_MOBILE_MONEY` or `SWITCH_WALLET_DEPOSIT_BANK` and `!switchWalletTesting`:  
→ calls `newThirdPartyService.handleThirdPartyService(dto)`  
→ Deserializes `SwitchWalletDisburseRequest` from `dto.transferMetadata`  
→ Calls `switchWalletClient.disburseTransaction(disburseRequest)` — **external HTTP call to Switch API**  
→ If exception: `dto.errorReason = e.getMessage()`, status = FAILED, returns immediately

**3B-3.** Sends RabbitMQ `AccountTransferCreateEvent`

**3B-4.** Builds entityId: if transferId starts with SAP/EMB/EMC → entityId = transferId (otherwise null)

**3B-5.** Returns `BaseResponseDto(SUCCESS, "Transaction Processed, Transfer Id: ...")`  
**This does not finalize the transaction** — the wallet balances are NOT yet updated at this point for Switch payments. The flow pauses and waits for the Switch webhook.

---

### Step 4 — Salary/Advance Payment Completion: `processSalaryAndAdvancePayment()`

**Class:** `AccountTransferListener` (called from `completeTransaction`)

Triggered when `transferAggregateId.startsWith("SAP")` (Salary type, wallet payment path):

1. Deserializes `SalaryPaymentMetadata` from `dto.getSalaryMetadata()`
2. Loads all `SalaryPayment` rows: `salaryPaymentRepository.findAllByBulkPaymentId(paymentId)`
3. Loads `SalaryPaymentRequest` by UUID; loads `ThirdParty` by `thirdPartyAggregateId`
4. For each `SalaryPayment`:
    - Looks up `Employee`: `employeeRepository.findByMsisdnAndThirdParty(msisdn, thirdParty)` — if null, skips
    - Advance payment clear: if `paymentType == "Salary"` and `advancePayment > 0` and `status == ACTIVE` and `salary - salaryPaid == advancePayment` → sets `employee.advancePayment = 0.0` and saves
    - Checks `SwitchWalletCallbackMetadata` by `dto.getId()` (for Switch path)
    - If `dto.transferType == SALARY_PAYMENT`: sets `SalaryPayment.status = PAID` and saves

---

### Step 5 — Incentive Payment Completion: `processIncentivePayment()`

**Class:** `AccountTransferListener` (called from `completeTransaction`)

Triggered when `transferAggregateId.startsWith("EMB")` or `"EMC"`:

1. Deserializes `SalaryPaymentMetadata` from `dto.getSalaryMetadata()`
2. Loads single `SalaryPayment` by `bulkPaymentId` + `msisdn`: `salaryPaymentRepository.findTopByBulkPaymentIdAndMsisdn(paymentId, msisdn)`
3. Checks `SwitchWalletCallbackMetadata`
4. If `dto.transferType == SALARY_PAYMENT`: sets status `PAID` and saves

---

### Step 6 — Switch Webhook Callback (SWITCH path only)

**Class:** `SwitchWalletHookController`  
**Endpoint:** `POST /switch/webhook/disburse`

When the Switch API completes asynchronously:

1. Parses `SwitchDisburseWebhookResponse` from raw JSON body
2. Loads `AccountTransfer` by `reference` (= transferAggregateId)
3. Saves `SwitchWalletCallbackMetadata` with status `SUCCESS` or `FAILED`
4. Maps `AccountTransfer` → `TransferEventDto`, sets `transferType`, `salaryMetadata`, `lastTransferId` from persisted entity
5. If already `FAILED` in DB → returns immediately
6. If SUCCESS and `payoutStatus != "PENDING"`:
    - Calls `gPayAccountTransferService.processTransaction(dto)` — this time `requiresInitialization()` for SWITCH_WALLET_DEPOSIT_MOBILE_MONEY/BANK **is false** (they are in `ALREADY_INITIATED_TRANSFER_TYPES`)
    - Therefore skips initiation, goes straight to `executeTransactionCore`
    - `updateWalletBalances` runs — **this is when wallet balances are actually debited/credited for Switch payments**
    - `finishTransaction` runs — `completeTransaction` detects SAP/EMB/EMC prefix, calls `processSalaryAndAdvancePayment` or `processIncentivePayment`
7. If FAILED: calls `gPayAccountTransferService.rejectTransaction(dto)` → marks DB as FAILED, sends RabbitMQ event

---

### Step 7 — Testing Mode Short-Circuit

**Only active when `switchWalletTesting = true`**

After `beginTransaction()` returns in Step 3B, the controller calls:  
`switchWalletCommandService.disburseDemo(transferId)`  
→ Builds a hardcoded SUCCESS webhook payload  
→ Calls `hookController.handleDisburseCallback(payload)` **in-process synchronously**  
→ This immediately completes Step 6 without any external HTTP call

---

## Repositories Touched (in execution order)

|Repository|Operation|Trigger|
|---|---|---|
|`AdminUserQueryRepository`|`findOne(submittedById)`|Entry — load admin|
|`ThirdPartyAdminQueryRepository`|`findOne(adminId)`|Load third-party admin|
|`ThirdPartyQueryRepository`|`findByThirdPartyAggregateId(...)`|Load third-party entity|
|`SalaryPaymentRequestRepository`|`findByUuid(uuid)`|Load batch payroll|
|`SalaryPaymentRepository`|`findAllByBulkPaymentId(uuid)`|Load all payees|
|`WalletQueryRepository`|`findOneByUserIdAndType(adminId, THIRD_PARTY)`|Payer wallet — per loop|
|`EmployeeRepository`|`findByMsisdnAndThirdPartyAndStatus(...)`|Per-employee lookup (processBulkSalaryPay only)|
|`WalletQueryRepository`|`findByAggregateId(SWITCH_WALLET)` or `findOneByUserIdAndType(userId, SUBSCRIBER)`|Payee wallet — per loop|
|`WalletQueryRepository`|`findByAggregateId(walletId)` × 2|Validation — from/to wallet|
|`AccountTransferQueryRepository`|`findOne(transferId)`|Duplicate check in initiation|
|`AccountTransferMetadataQueryRepository`|`saveAndFlush(metadata)`|Save transfer metadata|
|`AccountTransferQueryRepository`|`saveAndFlush(accountTransfer)`|Persist STARTED record|
|`WalletQueryRepository`|`findByAggregateId(...)` × 2|Balance reads for debit/credit|
|`WalletQueryRepository`|`withdrawFromWallet(...)`|Debit — optimistic UPDATE|
|`WalletQueryRepository`|`depositToWallet(...)`|Credit|
|`AccountTransferOperationQueryRepository`|`save(fromOp)`, `save(toOp)`|Transfer operation ledger|
|`AccountTransferQueryRepository`|`save(accountTransfer)` × 2|Status updates (DONE → SUCCESS)|
|`SalaryPaymentRepository`|`save(salaryPayment)`|Mark individual payment as PAID|
|`EmployeeRepository`|`save(employee)`|Clear advance payment (if applicable)|
|`SalaryPaymentRequestRepository`|`findByUuid(uuid)` + `save(...)`|Approve the batch (status = APPROVED)|
|`SwitchWalletCallbackMetadataQueryRepository`|`save(...)`|Switch path: save callback|
|`AccountTransferQueryRepository`|`findOne(reference)`|Switch path: webhook lookup|

---

## Entities Involved

- `Admin` — payer identity
- `ThirdPartyAdmin` — maps admin → third party
- `ThirdParty` — owner of the payment batch
- `SalaryPaymentRequest` — the payroll batch record
- `SalaryPayment` — one row per payee in the batch
- `Employee` — looked up to resolve payment mode, advance payment (WALLET path only)
- `Wallet` — payer (THIRD_PARTY type) and payee (SUBSCRIBER or SWITCH_WALLET)
- `AccountTransfer` — the transfer ledger record
- `AccountTransferMetadata` — JSON metadata associated with the transfer
- `AccountTransferOperation` — debit/credit operation records (WITHDRAW + DEPOSIT)
- `SwitchWalletCallbackMetadata` — Switch webhook response record (SWITCH path only)

---

## End-to-End Execution Sequence

```
POST /third-party/salary-payment-new
│
├─ 1. adminUserQueryRepository.findOne(submittedById)
├─ 2. thirdPartyService.getThirdPartyFromAdminId()
│      ├─ ThirdPartyAdminQueryRepository.findOne()
│      └─ ThirdPartyQueryRepository.findByThirdPartyAggregateId()
├─ 3. salaryPaymentRequestRepository.findByUuid()
├─ 4. thirdPartyService.getSalaryPayeeInfosNew()
│      └─ salaryPaymentRepository.findAllByBulkPaymentId()
│
├─ [BRANCH: fileUrl == null → processBulkSalaryPay, else processBulkIncentivePay]
│
└─ FOR EACH payee:
       ├─ walletService.getWalletOfUserByIdAndTypeFromRepo() → WalletQueryRepository.findOneByUserIdAndType()
       ├─ [WALLET path only] employeeRepository.findByMsisdnAndThirdPartyAndStatus()
       ├─ [WALLET path] userService.getUserInfoFromMsisdn() → WalletQueryRepository.findOneByUserIdAndType()
       ├─ [SWITCH path] walletService.getWalletFromWalletRepoByWalletId(SWITCH_WALLET)
       ├─ Build TransferEventDto + SwitchWalletDisburseRequest + SalaryPaymentMetadata
       │
       ├─ [WALLET path] gPayAccountTransferService.processTransaction(dto)
       │      ├─ executeInterceptorsAndInitiate()
       │      │      ├─ MfsUtils.getScaledMoney()
       │      │      ├─ transferValidationService.checkTransactionValidations()
       │      │      │      ├─ walletService.getWalletInfoFromDB() × 2
       │      │      │      ├─ userService.getUserInfoFromId() × 2
       │      │      │      └─ run validation rules (no pin check, channelType=WEB)
       │      │      ├─ serviceAndCommissionChargeService.calculateServiceAndCommissionCharge()
       │      │      │      └─ isServiceChargeApplicable() = false → serviceCharge=0, commissions=[]
       │      │      └─ accountTransferListener.initiateTransaction()
       │      │             ├─ AccountTransferMetadataQueryRepository.saveAndFlush()
       │      │             └─ AccountTransferQueryRepository.saveAndFlush() [status=STARTED]
       │      ├─ rabbitMQProducer.rabbitProducer() [AccountTransferCreateEvent]
       │      └─ executeTransactionCore()
       │             ├─ checkMetadata() → no-op for salary types
       │             ├─ newWalletService.updateWalletBalances() [@Transactional]
       │             │      ├─ WalletQueryRepository.findByAggregateId() (balance read)
       │             │      ├─ WalletQueryRepository.withdrawFromWallet() (optimistic UPDATE)
       │             │      ├─ RabbitMQ MoneySubtractedEvent
       │             │      ├─ WalletQueryRepository.depositToWallet()
       │             │      └─ RabbitMQ MoneyAddedEvent
       │             ├─ newThirdPartyService.handleThirdPartyService() → no-op (SALARY_PAYMENT hits default)
       │             └─ finishTransaction() [@Transactional]
       │                    ├─ accountTransferListener.processTransaction()
       │                    │      ├─ AccountTransferOperationQueryRepository.save() × 2
       │                    │      └─ AccountTransferQueryRepository.save() [status=DONE]
       │                    ├─ accountTransferListener.updateApprovalDetail() → no-op
       │                    ├─ accountTransferListener.updateTransactionStatus() → status=SUCCESS
       │                    ├─ accountTransferListener.completeTransaction()
       │                    │      └─ processSalaryAndAdvancePayment() or processIncentivePayment()
       │                    │             ├─ salaryPaymentRepository.findAllByBulkPaymentId()
       │                    │             ├─ employeeRepository.findByMsisdnAndThirdParty()
       │                    │             ├─ [if advance] employeeRepository.save()
       │                    │             └─ salaryPaymentRepository.save() [status=PAID]
       │                    ├─ commissionInfoListener.saveCommissionData() → no-op
       │                    ├─ promotionListener.handlePromotionalOffer()
       │                    └─ fixTransferMetadataProcessor.handleMetadataForCompleteTransaction()
       │
       ├─ [SWITCH path] gPayAccountTransferService.beginTransaction(dto)
       │      ├─ executeInterceptorsAndInitiate() [same as above]
       │      ├─ newThirdPartyService.handleThirdPartyService()
       │      │      └─ switchWalletClient.disburseTransaction() → EXTERNAL HTTP CALL
       │      ├─ rabbitMQProducer.rabbitProducer() [AccountTransferCreateEvent]
       │      └─ returns SUCCESS (wallets NOT yet updated)
       │
       ├─ [if switchWalletTesting=true && SWITCH] switchWalletCommandService.disburseDemo()
       │      └─ SwitchWalletHookController.handleDisburseCallback() [in-process]
       │             └─ gPayAccountTransferService.processTransaction(dto)
       │                    └─ [skips initiation, runs executeTransactionCore with balance updates]
       │
       └─ Thread.sleep(salaryPaymentThreadSleep)  ← every loop iteration
│
├─ thirdPartyService.approvePayroll()
│      └─ SalaryPaymentRequestRepository.findByUuid() + save() [status=APPROVED]
│
└─ return TransferResponseDto(PROCESSING, "Salary Payment is being processed.")
```

---

## Critical Business Logic Points

1. **Single fixed HTTP response regardless of outcomes.** The controller always returns `PROCESSING`. There is no rollback if one employee transfer fails. Failures are only reflected in individual `SalaryPayment.status` rows and in individual `AccountTransfer` records.
2. **`payroll.fileUrl` is the branch selector.** `null` → employee-DB-lookup path (`processBulkSalaryPay`). Non-null → file-upload path (`processBulkIncentivePay`). The file-upload path never touches the `Employee` table; it uses `SalaryPayeeInfo` fields directly.
3. **Advance payment deduction only happens in `processBulkSalaryPay` for `paymentType = "Salary"`.** It computes `max(employee.salary - advancePayment, 0)` as the transfer amount. Bonus and Commission types always use the stored `SalaryPayeeInfo.amount`.
4. **Advance payment clearing (zeroing) happens post-success in `processSalaryAndAdvancePayment`.** The condition is exact: `salary - salaryPaid == advancePayment`. If any amount mismatch exists, the advance is not cleared.
5. **Transfer type `SALARY_PAYMENT` is hardcoded in the `TransferEventDto` construction regardless of what `transferType` was set from the paymentType string.** For SWITCH employees, it's then overwritten to `SWITCH_WALLET_DEPOSIT_BANK` or `SWITCH_WALLET_DEPOSIT_MOBILE_MONEY` just before dispatch. The SAP/EMB/EMC prefix on the transfer ID, not the `transferType`, is what drives `completeTransaction()` branching.
6. **Service charge and commission are always zero for all salary payment types.** `SALARY_PAYMENT`, `EMPLOYEE_BONUS`, `EMPLOYEE_COMMISSION` are all absent from `isServiceChargeApplicable()`, so no service charge is calculated, no commission entries are created.
7. **`channelType = WEB` bypasses all pin validation.** This is set explicitly so that the third-party admin does not need a pin code to initiate bulk payments.
8. **Switch payments are asynchronous — wallet balances are NOT updated until the webhook arrives.** `beginTransaction()` only persists the STARTED record and calls the external API. Balances change only when `processTransaction()` is called from the webhook handler.
9. **`lastTransferId` chaining for optimistic locking.** Each iteration tracks the previous transfer's ID. The wallet `withdrawFromWallet` UPDATE includes this as a WHERE condition. If the chain is broken (e.g., by a concurrent transfer), `rowAffected == 0` and `MFSException` is thrown, rolling back that wallet update.

---

## Potentially Risky / Complex Sections

1. **`Thread.sleep()` inside a hot loop on every employee.** The controller thread is blocked for `${salary.payment.thread.sleep}` milliseconds per payee. A batch of 1000 employees with a 1-second sleep would hold the HTTP thread for 1000+ seconds. This is synchronous and will tie up a servlet thread.
2. **Payer wallet re-fetched from DB on every loop iteration (`getWalletOfUserByIdAndTypeFromRepo`).** This is intentional to get current balance after prior deductions, but it means N DB reads for N employees. There is no transactional wrapper around the outer loop — each transfer is its own transaction.
3. **Silent skip when employee is not found in `processBulkSalaryPay`.** If `employeeRepository.findByMsisdnAndThirdPartyAndStatus()` returns null, the payee is skipped entirely — no error, no status update on the `SalaryPayment` row. That row stays `PENDING` indefinitely.
4. **`approvePayroll()` is called unconditionally after the loop.** Even if every single transfer failed or was skipped, the `SalaryPaymentRequest` batch is marked `APPROVED`. The approval reflects that the disbursement was attempted, not that it succeeded.
5. **Optimistic locking in `withdrawFromWallet`.** Uses `last_transfer_id` as an optimistic lock. If a concurrent transaction has already updated the wallet, `rowAffected == 0` → `MFSException`. This exception propagates up through `updateWalletBalances` → `executeTransactionCore` → caught in the outer try/catch → returns a FAILED response for that employee. The `AccountTransfer` record remains in STARTED state in this failure path since `@Transactional` on `updateWalletBalances` will roll back.
6. **`processSalaryAndAdvancePayment` iterates ALL salary payments in the batch for every WALLET-mode transfer that completes.** For a batch of N employees with WALLET mode, this method is called N times, and each call does `findAllByBulkPaymentId` + N iterations. This is O(N²) DB reads as N grows.
7. **`switchWalletTesting = true` calls the webhook handler in-process synchronously** inside the outer salary payment loop. This means for testing, each SWITCH employee triggers a full synchronous completion before `Thread.sleep()` is called — but this is single-threaded and correct. In production with `switchWalletTesting = false`, the webhook is asynchronous and the `AccountTransfer` record could be completed concurrently with subsequent iterations of the loop.
8. **`transferType` is set twice for SWITCH employees.** It is first set in the `TransferEventDto` to `SALARY_PAYMENT` (hardcoded), then overwritten to `SWITCH_WALLET_DEPOSIT_BANK` or `SWITCH_WALLET_DEPOSIT_MOBILE_MONEY` just before calling `beginTransaction()`. However, the SAP/EMB/EMC prefix on the ID — set from the original type — is what determines the `processSalaryAndAdvancePayment` / `processIncentivePayment` branch in `completeTransaction()`. So `SALARY_PAYMENT` prefix `"SAP"` is used for Switch salary, but the actual stored transfer type is `SWITCH_WALLET_DEPOSIT_*`.
9. **No maker-checker flow exists in this endpoint.** The admin who calls this API is both maker and checker. `approvePayroll()` is called inline. There is no separate approval step or second-admin confirmation.