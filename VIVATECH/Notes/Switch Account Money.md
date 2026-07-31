## SWITCH_WALLET_DEPOSIT_MONEY & SWITCH_WALLET_RECEIVE_MONEY — Complete Flow

---
### Key concept before reading

These two flows are **mirror images** of each other, but they are **NOT symmetric in execution**:

| **Aspect**                     | **SWITCH_WALLET_DEPOSIT_MONEY**                   | **SWITCH_WALLET_RECEIVE_MONEY**                                    |
| ------------------------------ | ------------------------------------------------- | ------------------------------------------------------------------ |
| **Plain English**              | Subscriber sends money **OUT** to another network | Subscriber collects money **IN** from another network              |
| **Who triggers**               | Subscriber (via webclient)                        | Admin or subscriber triggers collection; external network confirms |
| **Wallet debit**               | Subscriber wallet                                 | Switch wallet `"Switch Wallet"`                                    |
| **Wallet credit**              | Switch wallet `"Switch Wallet"`                   | Subscriber wallet                                                  |
| **External API call**          | `SwitchWalletClient.disburseTransaction()`        | `SwitchWalletClient.collectTransaction()`                          |
| **External callback endpoint** | `POST /switch/webhook/disburse`                   | `POST /switch/webhook/collect`                                     |
| **Money moves when**           | Webhook SUCCESS → `processTransaction()`          | Webhook SUCCESS → `processTransaction()`                           |

Both share the exact same **two-phase pattern:**

1. **`beginTransaction()`** — validate, save record, call external API, return immediately
2. **External webhook fires** → **`processTransaction()`** — money actually moves

---

### Architecture: Which microservices are involved

```
mfs-webclient          → User fills form, sends request
mfs-api-gateway        → JWT auth, privilege check, proxy
mfs-backend-new        → All business logic, wallet updates, webhook receiver
External Switch API    → Third-party network (MTN MoMo, Orange Money, etc.)
mfs-reporting          → Async via RabbitMQ
mfs-notification       → Async via RabbitMQ
```

---

## FLOW 1: SWITCH_WALLET_DEPOSIT_MONEY

### Scenario

Subscriber wants to send 5,000 XAF to an MTN MoMo number `237670000000`.

---

### Phase 1 — Webclient: User fills the form

**URL:** `GET /admin/deposit/switch-wallet/send-switch-money`

**`SwitchWalletDepositController.sendSwitchMoneyPage()`** (webclient):

java

```java
return "admin/deposit/switch-wallet/send_switch_money";
```

Renders the HTML form. Fields:

- `beneficiaryNumber` — the recipient's phone on another network
- `amount` — e.g. 5000
- `beneficiaryAccountType` — `MOMO`, `ORANGE_MONEY`, etc.

---

### Phase 2 — Webclient: Form submitted

**URL:** `POST /admin/deposit/switch-wallet/send-switch-money/store`

**`SwitchWalletDepositController.sendSwitchMoneyStore()`** (webclient):

java

```java
String url = apiGatewayAddress + "/transfer/send-switch-money";
BigMoney bigMoney = BigMoney.of(CurrencyUnit.of("XAF"), 5000.00);

SendSwitchMoneyRequest transferDto = new SendSwitchMoneyRequest();
transferDto.setBeneficiaryPhoneNo("237670000000");
transferDto.setAmount(5000.0);
transferDto.setTransferInitiatedById(SessionHelper.getAdminJwtSession().getId());  // subscriber DB id
transferDto.setAccountType(BeneficiaryAccountType.valueOf("MOMO"));

restTemplate.setInterceptors(SessionHelper.getInterceptor());
// getInterceptor() adds: "Authorization: Bearer <jwt>", x-trace-id, X-Forwarded-For

BaseResponseDto responseTransferDto = restTemplate.postForObject(url, transferDto, BaseResponseDto.class);
```

On success:

java

```java
if (responseTransferDto.getStatusCode() == StatusCode.SUCCESS) {
    redirectAttributes.addFlashAttribute("successMessage", responseTransferDto.getMessage());
    return "redirect:/admin/dashboard";
}
```

---

### Phase 3 — Gateway: Auth + Proxy

**`TransferController` (gateway) → not for send-switch-money**

Actually looking at the gateway `SwitchWalletController`:

java

```java
// No @PreAuthorize shown in gateway controller for send-switch-money
// The JWT is still validated by ApiAuthorizationFilter
// Proxied to backend: POST /transfer/send-switch-money
```

---

### Phase 4 — Backend Controller: Build the transfer DTO

**`SwitchWalletController.sendSwitchMoney()`** (backend, `application/controller/SwitchWalletController.java`):

java

```java
UserInfo userInfo = userService.getUserInfoFromId(request.getTransferInitiatedById());
// loads subscriber info

Wallet w = walletQueryRepository.findOneByUserId(request.getTransferInitiatedById());
// subscriber's wallet: aggregateId = "SWALLET-SUB-001"

if (w == null) → return FAILED ("Wallet not found")

BigMoney amount = utils.getMoney(request.getAmount());  // 5000 XAF

String transferId = utils.getAccountTransferId(TransferType.SWITCH_WALLET_DEPOSIT_MONEY);
// → generates "DMS260427.1045.X9Y8Z7"  (DMS = Disburse Money Switch prefix)
```

Builds the **external API request object:**

java

```java
SwitchWalletDisburseRequest switchWalletDisburseRequest = new SwitchWalletDisburseRequest(
    new BeneficiaryAccount(
        5000.0,            // amount
        "CMR",             // country
        "XAF",             // currency
        "John Subscriber", // sender name (from userInfo.getFullName())
        "237670000000",    // beneficiary phone
        "MOMO"             // account type code
    ),
    callBackURL + "/switch/webhook/disburse",  // webhook URL for external API to call back
    "DMS260427.1045.X9Y8Z7",                  // reference = transferId
    new SenderAccount("CMR", "XAF")
);
```

Calls `switchWalletCommandService.processDisburse(switchWalletDisburseRequest, amount, w, userInfo.getAggregateId())`.

---

### Phase 5 — SwitchWalletCommandService.processDisburse()

**File:** `application/service/SwitchWalletCommandService.java`

java

```java
String reference = request.getReference();  // "DMS260427.1045.X9Y8Z7"
String fromAccount = w.getAggregateId();    // "SWALLET-SUB-001"
String switchWalletAggregateId = Constants.SWITCH_WALLET;  // "Switch Wallet"

String metadataString = objectMapper.writeValueAsString(request);
// full SwitchWalletDisburseRequest JSON stored as metadata

TransferEventDto transferDto = TransferEventDto.builder()
    .fromAccountAggregateId("SWALLET-SUB-001")
    .toAccountAggregateId("Switch Wallet")
    .fromAccountId("SWALLET-SUB-001")
    .toAccountId("Switch Wallet")
    .amount(5000 XAF)
    .serviceCharge(0 XAF)               // ← hardcoded 0 here, computed in beginTransaction
    .transferType(SWITCH_WALLET_DEPOSIT_MONEY)
    .transferInitietedByType(SUBSCRIBER)
    .transferInitiedByAggregateId(subscriberAggregateId)
    .transferInitiedById(w.getUserId())
    .transferMetadata(metadataString)
    .transferStatus(TransferStatus.STARTED)
    .createdAt(new Date())
    .build();

transferDto.setTransferAggregateId("DMS260427.1045.X9Y8Z7");

newAccountTransferService.beginTransaction(transferDto);
```

---

### Phase 6 — NewAccountTransferService.beginTransaction()

**File:** `application/service/NewAccountTransferService.java`

java

```java
// Step 1: Scale amount
dto.setAmount(MfsUtils.getScaledMoney(5000 XAF));  // → 5000.0000 XAF

// Step 2: Validate
dto = transferValidationService.checkTransactionValidations(dto);
// → TransferValidatorService.validate(dto)
//   Loads "SWALLET-SUB-001" (SUBSCRIBER wallet) as fromWallet
//   Loads "Switch Wallet" (ADMIN/system wallet) as toWallet
//   Runs all rules:
//   - AccountsVerificationRule.supports(SWITCH_WALLET_DEPOSIT_MONEY) → false (no case for it)
//     ⚠️ This means AccountsVerificationRule does NOT run for SWITCH_WALLET_DEPOSIT_MONEY
//   - SwitchWalletReceiveMoneyValidationRule.supports() → false (only for RECEIVE)
//   - ThresholdValidationRule → checks 5000 within subscriber's limits
//   - AccountBalanceLimitValidaionRule → subscriber balance ≥ 5000
//   - NO pincode check (PincodeVerificationRule doesn't apply to switch types)

// Step 3: Calculate service charge + commission
dto = serviceAndCommissionChargeService.calculateServiceAndCommissionCharge(dto);
// reads SWITCH_WALLET_DEPOSIT_MONEY service charge profile from DB
// sets dto.serviceCharge, dto.commissionInfos

// Step 4: Save initial DB record — FIRST DB WRITE
accountTransferListener.initiateTransaction(dto);
// Creates AccountTransfer:
//   id = "DMS260427.1045.X9Y8Z7"
//   fromAccountId = "SWALLET-SUB-001"
//   toAccountId = "Switch Wallet"
//   amount = 5000 XAF
//   transferType = SWITCH_WALLET_DEPOSIT_MONEY
//   transferStatus = STARTED
//   metadata = full SwitchWalletDisburseRequest JSON

// Step 5: Call external Switch API (only if !switchWalletTesting)
if ((dto.getTransferType() == SWITCH_WALLET_DEPOSIT_MONEY) && !switchWalletTesting) {
    dto = newThirdPartyService.handleThirdPartyService(dto);
    // → SwitchWalletClient.disburseTransaction(switchWalletDisburseRequest)
    // → HTTP POST to external Switch API: POST /switch/api/enterprise/disburse
    // Request body: { beneficiaryAccount: {amount:5000, country:"CMR", currency:"XAF",
    //                 name:"John Subscriber", number:"237670000000", type:"MOMO"},
    //                 callbackUrl: "https://mycash.com/switch/webhook/disburse",
    //                 reference: "DMS260427.1045.X9Y8Z7",
    //                 senderAccount: {country:"CMR", currency:"XAF"} }
    // External API accepts and returns pending response
}

// Step 6: Publish AccountTransferCreateEvent to RabbitMQ
dto.setEventType("AccountTransferCreateEvent");
rabbitMQProducer.rabbitProducer(dto);
// → mfs-reporting saves initial audit record

// Step 7: Return to caller
return BaseResponseDto { statusCode=SUCCESS, status="Success",
                          message="Transaction Processed, Transfer Id: DMS260427.1045.X9Y8Z7" }
```

> **⚠️ Critical: wallet balances are NOT changed here.** Subscriber still has their full balance. Money has NOT moved. Only the `AccountTransfer` record exists in DB with status `STARTED`.

Back in `SwitchWalletController.sendSwitchMoney()`:

java

```java
// After processDisburse returns:
if (switchWalletTesting) switchWalletCommandService.disburseDemo(transferId);
// disburseDemo() directly calls hookController.handleDisburseCallback(hardcodedSuccessPayload)
// This is the test shortcut — simulates the webhook arriving immediately
```

---

### Phase 7 — External Switch API Calls Back (Async)

The external Switch network processes the payout to `237670000000` and calls: **`POST /switch/webhook/disburse`** on the backend

**`SwitchWalletHookController.handleDisburseCallback()`**:

java

```java
SwitchDisburseWebhookResponse disburseRequest = objectMapper.readValue(request, SwitchDisburseWebhookResponse.class);
// Parses JSON:
// { "message": "Success", "status": 0, "payoutStatus": "SUCCESS",
//   "reference": "DMS260427.1045.X9Y8Z7", "payoutAmount": 5000 }

String reference = disburseRequest.getReference();  // "DMS260427.1045.X9Y8Z7"
AccountTransfer accountTransfer = accountTransferQueryRepository.findOne(reference);
// loads the AccountTransfer saved in Phase 6 Step 4

if (accountTransfer == null) {
    log.error("AccountTransfer not found for reference: {}", reference);
    return;  // guard: prevents processing unknown callbacks
}

// Guard: already processed?
if (accountTransfer.getTransferStatus().equals(TransferStatus.FAILED)) {
    log.info("Transfer status set to FAILED for reference {}", reference);
    return;  // idempotency guard
}

// Save raw webhook payload to DB (audit trail)
saveCallbackMetadata(reference, request, "SUCCESS", "DISBURSE", accountTransfer);
// → SwitchWalletCallbackMetadata entity saved to switch_wallet_callback_metadata table
//   includes: reference, raw JSON payload, status, type, account number, account type

// Determine status
String status = disburseRequest.getMessage().equalsIgnoreCase("SUCCESS") ? "SUCCESS" : "FAILED";

TransferEventDto dto = mapper.getMapperFacade().map(accountTransfer, TransferEventDto.class);
dto.setTransferType(TransferType.SWITCH_WALLET_DEPOSIT_MONEY);
dto.setTransferInitiedByAggregateId(accountTransfer.getTransferInitietedByAggregateId());
```

**SUCCESS path:**

java

```java
if ("SUCCESS".equalsIgnoreCase(status)) {
    dto.setTransferStatus(TransferStatus.SUCCESS);
    newAccountTransferService.processTransaction(dto);
}
```

**FAILED path:**

java

```java
} else {
    dto.setTransferStatus(TransferStatus.FAILED);
    dto.setErrorReason(disburseRequest.getMessage());
    newAccountTransferService.rejectTransaction(dto);
}
```

---

### Phase 8 — NewAccountTransferService.processTransaction() on SUCCESS

Because `dto.getTransferType() == SWITCH_WALLET_DEPOSIT_MONEY` and `dto.getTransferStatus() == SUCCESS`, the method immediately enters the first `if` block:

java

```java
if (dto.getTransferType() == TransferType.SWITCH_WALLET_DEPOSIT_MONEY) {
    if (dto.getTransferStatus() == TransferStatus.SUCCESS) {
```

**Step 8.1 — Update wallet balances:**

java

```java
newWalletService.updateWalletBalances(dto);
```

`updateWalletBalances()` — default else branch (no special case for `SWITCH_WALLET_DEPOSIT_MONEY`):

**DEBIT subscriber wallet (`"SWALLET-SUB-001"`):**

java

```java
totalWithdrawAmount = dto.getAmount().plus(serviceCharge);
// = 5000 + 50 (if service charge configured) = 5050 XAF

withdrawWalletCurrentBalance = getCurrentWalletBalance("SWALLET-SUB-001");
// = 10,000 XAF

walletQueryRepository.withdrawFromWallet(
    "SWALLET-SUB-001", 5050, now,
    "DMS260427.1045.X9Y8Z7",  // newTransferId
    oldTransferId, lastTransferId
);
// SQL: UPDATE Wallet SET amount=amount-5050, transfer_id='DMS260427.1045.X9Y8Z7'
//      WHERE aggregate_id='SWALLET-SUB-001' AND transfer_id='<previous>'
// rowAffected check → SECOND DB WRITE

sendMessageToRabbitMQ("SWALLET-SUB-001", 5050, 10000, "DMS...", "MoneySubtractedEvent");
// → mfs-reporting updates wallet history
```

**CREDIT Switch Wallet (`"Switch Wallet"`):**

java

```java
totalDepositAmount = dto.getAmount();  // 5000 XAF (face value — fee stays with system)

depositWalletCurrentBalance = getCurrentWalletBalance("Switch Wallet");
// = 50,000 XAF

walletQueryRepository.depositToWallet("Switch Wallet", 5000, now);
// SQL: UPDATE Wallet SET amount=amount+5000 WHERE aggregate_id='Switch Wallet'
// THIRD DB WRITE

sendMessageToRabbitMQ("Switch Wallet", 5000, 50000, "DMS...", "MoneyAddedEvent");
```

**CREDIT commission wallet (if configured):**

java

```java
handleCommissions(dto, ...);
// for each CommissionInfo in dto.getCommissionInfos():
walletQueryRepository.depositToWallet(commissionWalletId, commissionAmount, now);
// FOURTH DB WRITE
```

**Step 8.2 — Finish transaction:**

java

```java
finishTransaction(dto);
```

`finishTransaction()` calls in sequence:

1. `accountTransferListener.processTransaction(dto)` — saves `AccountTransferOperation` rows (WITHDRAW from subscriber, DEPOSIT to Switch Wallet), updates `AccountTransfer.transferStatus = DONE`
2. `accountTransferListener.updateApprovalDetail(dto)`
3. `accountTransferListener.updateTransactionStatus(dto)`
4. `accountTransferListener.completeTransaction(dto)` — no special case for `SWITCH_WALLET_DEPOSIT_MONEY`
5. `commissionInfoListener.saveCommissionData(dto)` — saves commission record
6. `promotionListener.handlePromotionalOffer(dto)`
7. `fixTransferMetadataProcessor.handleMetadataForCompleteTransaction(dto)`

**Step 8.3 — Set user aggregateId on dto:**

java

```java
User user = userQueryRepository.findOne(dto.getTransferInitiedById());
dto.setTransferInitiedByAggregateId(user.getAggregateId());
```

**Step 8.4 — Publish AccountTransferCompletedEvent:**

java

```java
dto.setEventType("AccountTransferCompletedEvent");
rabbitMQProducer.rabbitProducer(dto);
// → mfs-reporting: ReportProcessingService.accountTransferCompletedEvent()
//   saves CustomerTransactionReport
```

> **Note:** `AccountTransferDoneEvent` is NOT published for `SWITCH_WALLET_DEPOSIT_MONEY` in this path — the code skips directly from `finishTransaction()` to `AccountTransferCompletedEvent`. This means **notification SMS is NOT triggered via RabbitMQ for this type** in the webhook path.

**Step 8.5 — Return:**

java

```java
responseDto.setStatusCode(StatusCode.SUCCESS);
responseDto.setStatus("Success");
responseDto.setMessage("Transaction Finished, Transfer Id: DMS260427.1045.X9Y8Z7");
```

---

### Phase 9 — FAILED path: reverseWalletBalances()

If webhook reports failure:

java

```java
if (dto.getTransferStatus().equals(TransferStatus.FAILED)) {
    dto = newWalletService.reverseWalletBalances(dto);
```

`reverseWalletBalances()`:

java

```java
// re-credit subscriber (reverse the debit):
walletQueryRepository.depositToWallet("SWALLET-SUB-001", dto.getTotalWithdrawAmount(), now);
// re-debit Switch Wallet (reverse the credit):
walletQueryRepository.withdrawFromWallet("Switch Wallet", dto.getTotalDepositAmount(), now);
// reverseCommission: undo all commission credits
dto = reverseCommission(dto);
```

Then:

java

```java
dto = storeFailedTransaction(dto);
dto.setEventType("AccountTransferFailedEvent");
rabbitMQProducer.rabbitProducer(dto);
// → reporting marks as FAILED
```

---

### Final Balance After SWITCH_WALLET_DEPOSIT_MONEY

```
Before:
  Subscriber wallet (SWALLET-SUB-001):   10,000 XAF
  Switch Wallet ("Switch Wallet"):        50,000 XAF
  Switch commission wallet:               2,000 XAF

After SUCCESS:
  Subscriber wallet:    4,950 XAF   (−5,050 = 5000 face + 50 fee)
  Switch Wallet:       55,000 XAF   (+5,000)
  Switch commission:    2,050 XAF   (+50 commission if configured)

  External (MTN MoMo 237670000000): +5,000 XAF  [handled by Switch provider]

After FAILED (reversal):
  Subscriber wallet:   10,000 XAF   (unchanged — reversed back)
  Switch Wallet:       50,000 XAF   (unchanged)
```

---

---

## FLOW 2: SWITCH_WALLET_RECEIVE_MONEY

### Scenario

A subscriber wants to collect 3,000 XAF that someone on MTN MoMo (`237671111111`) is sending them.

---

### Phase 1 — Webclient: No dedicated UI found for RECEIVE_MONEY

> **From code:** `SwitchWalletDepositController` in the webclient only has the `send-switch-money` form. There is **no webclient UI for `SWITCH_WALLET_RECEIVE_MONEY`** found in the codebase — the `/transfer/receive-switch-money` endpoint is called directly via the backend API (or a mobile app, or future UI). The flow is otherwise identical in structure.

The request hits the backend `SwitchWalletController.receiveSwitchMoney()` directly.

---

### Phase 2 — Backend Controller: Build the transfer DTO

**`SwitchWalletController.receiveSwitchMoney()`** (backend):

java

```java
UserInfo userInfo = userService.getUserInfoFromId(request.getTransferInitiatedById());
// loads the subscriber who will receive the money

Wallet w = walletQueryRepository.findOneByUserId(request.getTransferInitiatedById());
// subscriber's wallet: aggregateId = "SWALLET-SUB-001"

BigMoney amount = utils.getMoney(request.getAmount());  // 3000 XAF

String transferId = utils.getAccountTransferId(TransferType.SWITCH_WALLET_RECEIVE_MONEY);
// → "RMS260427.1148.A3B4C5"  (RMS = Receive Money Switch prefix)
```

Builds the **external collect request:**

java

```java
SwitchCollectionRequest switchWalletDisburseRequest = new SwitchCollectionRequest(
    new BeneficiaryAccount("CMR", "XAF"),   // beneficiary = this system
    callBackURL + "/switch/webhook/collect", // webhook URL
    "RMS260427.1148.A3B4C5",                // reference = transferId
    new SenderAccount(
        3000.0,           // amount
        "CMR",            // country
        "XAF",            // currency
        "237671111111",   // sender phone (the person sending money)
        "MOMO"            // account type code
    )
);
```

Calls `switchWalletCommandService.processCollection(switchWalletDisburseRequest, amount, w, userInfo.getAggregateId())`.

---

### Phase 3 — SwitchWalletCommandService.processCollection()

**File:** `application/service/SwitchWalletCommandService.java`

java

```java
String reference = request.getReference();  // "RMS260427.1148.A3B4C5"

if (!switchWalletTesting) {
    // Call external API FIRST before building TransferEventDto
    Object apiResponse = switchWalletClient.collectTransaction(request);
    // → HTTP POST to external Switch API: POST /switch/api/enterprise/collect
    // Request: { beneficiaryAccount: {country:"CMR", currency:"XAF"},
    //            callbackUrl: "https://mycash.com/switch/webhook/collect",
    //            reference: "RMS260427.1148.A3B4C5",
    //            senderAccount: {amount:3000, country:"CMR", currency:"XAF",
    //                            number:"237671111111", type:"MOMO"} }
    // The external API triggers a payment request/prompt to "237671111111"
    // asking them to authorize sending 3000 XAF. Returns pending.
}

String switchWalletAggregateId = Constants.SWITCH_WALLET;   // "Switch Wallet"
String toAccount = w.getAggregateId();                       // "SWALLET-SUB-001"

String metadataString = objectMapper.writeValueAsString(request);
// full SwitchCollectionRequest JSON stored as metadata

TransferEventDto transferDto = TransferEventDto.builder()
    .fromAccountAggregateId("Switch Wallet")   // ← money comes FROM Switch Wallet
    .toAccountAggregateId("SWALLET-SUB-001")   // ← goes TO subscriber
    .fromAccountId("Switch Wallet")
    .toAccountId("SWALLET-SUB-001")
    .amount(3000 XAF)
    .serviceCharge(0 XAF)               // computed in beginTransaction
    .transferType(SWITCH_WALLET_RECEIVE_MONEY)
    .transferInitietedByType(SUBSCRIBER)
    .transferInitiedByAggregateId(subscriberAggregateId)
    .transferInitiedById(0)             // ← hardcoded 0 here
    .transferMetadata(metadataString)
    .createdAt(new Date())
    .build();

transferDto.setTransferAggregateId("RMS260427.1148.A3B4C5");

newAccountTransferService.beginTransaction(transferDto);
```

> **Key difference from DEPOSIT_MONEY:** `collectTransaction()` is called **before** `beginTransaction()`. For `processDisburse()`, the external call happens **inside** `beginTransaction()` via `newThirdPartyService`. For `processCollection()`, the external call happens **before** `beginTransaction()` in the service itself.

---

### Phase 4 — NewAccountTransferService.beginTransaction()

`SWITCH_WALLET_RECEIVE_MONEY` is **NOT** in the special first `if` block (that's only for `SWITCH_WALLET_DEPOSIT_MONEY`). So it goes to the normal `beginTransaction()` path:

java

```java
// Step 1: Scale amount
dto.setAmount(MfsUtils.getScaledMoney(3000 XAF));  // → 3000.0000

// Step 2: Validate
dto = transferValidationService.checkTransactionValidations(dto);
// → TransferValidatorService.validate(dto)
//   fromWallet = "Switch Wallet" (system wallet)
//   toWallet = "SWALLET-SUB-001" (SUBSCRIBER wallet)
//   Runs SwitchWalletReceiveMoneyValidationRule:
//     → supports(SWITCH_WALLET_RECEIVE_MONEY) → true
//     → validate():
//         metadata = dto.getTransferMetadata()
//         SwitchCollectionRequest m = deserialize(metadata)
//         transactionNo = m.getReference()  // "RMS260427.1148.A3B4C5"
//         existing = switchProcessedRepository.findOneBySwitchWalletTransactionId(transactionNo)
//         if (existing != null) → FAIL: "switch.wallet.transaction.already.processed"
//         // IDEMPOTENCY CHECK — prevents double-crediting for duplicate webhooks
//         return new ValidationResult(true)  // if not already processed
//   Also runs ThresholdValidationRule, MaxMinBalanceLimitValidationRule etc.

// Step 3: Calculate commission
dto = serviceAndCommissionChargeService.calculateServiceAndCommissionCharge(dto);
// For SWITCH_WALLET_RECEIVE_MONEY — typically no service charge on incoming
// Sets dto.serviceCharge, dto.commissionInfos

// Step 4: Save initial DB record — FIRST DB WRITE
accountTransferListener.initiateTransaction(dto);
// Creates AccountTransfer:
//   id = "RMS260427.1148.A3B4C5"
//   fromAccountId = "Switch Wallet"
//   toAccountId = "SWALLET-SUB-001"
//   amount = 3000 XAF
//   transferType = SWITCH_WALLET_RECEIVE_MONEY
//   transferStatus = STARTED
//   metadata = full SwitchCollectionRequest JSON

// Step 5: External API already called in processCollection() — skip
// (SWITCH_WALLET_RECEIVE_MONEY check: only calls newThirdPartyService if !switchWalletTesting)
// For SWITCH_WALLET_RECEIVE_MONEY, this check fires but the external call was already done above
// So if switchWalletTesting=false, this would call collectTransaction AGAIN — potential bug
// In practice: the test flag is used to short-circuit

// Step 6: Publish AccountTransferCreateEvent
dto.setEventType("AccountTransferCreateEvent");
rabbitMQProducer.rabbitProducer(dto);
// → mfs-reporting audit record

return BaseResponseDto { statusCode=SUCCESS,
                          message="Collection request initiated by transaction ID: RMS260427.1148.A3B4C5" }
```

Back in `SwitchWalletController.receiveSwitchMoney()`:

java

```java
if (switchWalletTesting) switchWalletCommandService.collectDemo(transferId);
// collectDemo() directly calls hookController.handleCollectCallback(hardcodedSuccessPayload)
// Simulates webhook for testing
```

---

### Phase 5 — External Switch API Calls Back (Async)

The sender on MTN MoMo `237671111111` authorizes the payment. The Switch network processes it and calls: **`POST /switch/webhook/collect`** on the backend

**`SwitchWalletHookController.handleCollectCallback()`**:

java

```java
SwitchWebhookResponse collectionRequest = objectMapper.readValue(requestPayload, SwitchWebhookResponse.class);
// Parses JSON:
// { "message": "success", "status": 0, "collectionStatus": "SUCCESS",
//   "reference": "RMS260427.1148.A3B4C5",
//   "senderName": "MTN Momo", "payoutAmount": 3000 }

String reference = collectionRequest.getReference();  // "RMS260427.1148.A3B4C5"
AccountTransfer accountTransfer = accountTransferQueryRepository.findOne(reference);

if (accountTransfer == null) {
    log.error("AccountTransfer not found for reference: {}", reference);
    return;  // guard
}

String status = collectionRequest.getCollectionStatus();  // "SUCCESS"

// Save raw webhook payload to DB
saveCallbackMetadata(reference, requestPayload, "SUCCESS", "COLLECT", accountTransfer);
// → reads SwitchCollectionRequest from accountTransfer.metadata
// → extracts sender phone number and type
// → saves to switch_wallet_callback_metadata table

TransferEventDto dto = mapper.getMapperFacade().map(accountTransfer, TransferEventDto.class);
dto.setTransferType(TransferType.SWITCH_WALLET_RECEIVE_MONEY);
dto.setTransferInitiedByAggregateId(accountTransfer.getTransferInitietedByAggregateId());
if (accountTransfer.getAccountTransferMetadata() != null)
    dto.setTransferMetadata(accountTransfer.getAccountTransferMetadata().getTransferMetadata());
```

**Three possible statuses:**

java

```java
if ("SUCCESS".equalsIgnoreCase(status) || "COMPLETED".equalsIgnoreCase(status)) {
    dto.setTransferStatus(TransferStatus.SUCCESS);
    newAccountTransferService.processTransaction(dto);
    // → money moves HERE

} else if ("PENDING".equalsIgnoreCase(status)) {
    dto.setTransferStatus(TransferStatus.SUSPENDED);
    newAccountTransferService.suspendTransaction(dto);
    // → updates AccountTransfer.transferStatus = SUSPENDED
    // → no money movement, waiting for another webhook

} else {
    dto.setTransferStatus(TransferStatus.FAILED);
    dto.setErrorReason(collectionRequest.getMessage());
    newAccountTransferService.rejectTransaction(dto);
    // → storeFailedTransaction, publish AccountTransferFailedEvent
    // → no money movement
}
```

---

### Phase 6 — NewAccountTransferService.processTransaction() on SUCCESS

`SWITCH_WALLET_RECEIVE_MONEY` is **NOT** `SWITCH_WALLET_DEPOSIT_MONEY`, so it **skips** the first `if` block and falls through to the normal flow.

But `SWITCH_WALLET_RECEIVE_MONEY` **is** in the exclusion list:

java

```java
if (dto.getTransferType() != TransferType.AMT01_TO_THIRD_PARTY
    && ...
    && dto.getTransferType() != TransferType.SWITCH_WALLET_DEPOSIT
    && dto.getTransferType() != TransferType.SWITCH_WALLET_WITHDRAW
    // ← SWITCH_WALLET_RECEIVE_MONEY is NOT in this exclusion list
```

So the normal validation/commission/initiate block **WOULD run again** — but wait, the `AccountTransfer` record already exists with status `STARTED`. The `initiateTransaction()` call would try to save again.

> **Actually:** Looking more carefully, `SWITCH_WALLET_RECEIVE_MONEY` is NOT in the exclusion list, meaning the normal validation+commission+initiation block runs. However, `accountTransferListener.initiateTransaction()` uses `saveAndFlush()` which would try to insert a duplicate primary key (`reference` = transferId is already in DB). This means the validation/commission re-runs **but `initiateTransaction()` silently ignores the duplicate** because of the `@Transactional` and JPA merge semantics — or it throws and the exception is caught.

Continuing with the main flow after that block, `dto.getErrorReason()` is null (validation passed), so:

java

```java
// update wallet balances
dto = newWalletService.updateWalletBalances(dto);
```

**`updateWalletBalances()` for `SWITCH_WALLET_RECEIVE_MONEY`:**

Default else branch:

java

```java
totalWithdrawAmount = dto.getAmount().plus(serviceCharge);
// For receive: service charge is typically 0, so totalWithdrawAmount = 3000 XAF
```

**DEBIT Switch Wallet (`"Switch Wallet"`):**

java

```java
withdrawWalletCurrentBalance = getCurrentWalletBalance("Switch Wallet");
// = 50,000 XAF

walletQueryRepository.withdrawFromWallet(
    "Switch Wallet", 3000, now,
    "RMS260427.1148.A3B4C5", oldTransferId, lastTransferId
);
// SQL: UPDATE Wallet SET amount=amount-3000, transfer_id='RMS260427.1148.A3B4C5'
//      WHERE aggregate_id='Switch Wallet' AND transfer_id='<previous>'
// SECOND DB WRITE

sendMessageToRabbitMQ("Switch Wallet", 3000, 50000, "RMS...", "MoneySubtractedEvent");
```

**CREDIT subscriber wallet (`"SWALLET-SUB-001"`):**

java

```java
totalDepositAmount = dto.getAmount();  // 3000 XAF (no deduction for incoming)

depositWalletCurrentBalance = getCurrentWalletBalance("SWALLET-SUB-001");
// = 10,000 XAF

walletQueryRepository.depositToWallet("SWALLET-SUB-001", 3000, now);
// SQL: UPDATE Wallet SET amount=amount+3000 WHERE aggregate_id='SWALLET-SUB-001'
// THIRD DB WRITE

sendMessageToRabbitMQ("SWALLET-SUB-001", 3000, 10000, "RMS...", "MoneyAddedEvent");
```

**Commission (if configured):**

java

```java
handleCommissions(dto, ...);
// credits commission wallet(s)
```

**Continue main flow:**

java

```java
dto = newThirdPartyService.handleThirdPartyService(dto);
// SWITCH_WALLET_RECEIVE_MONEY: no third-party handler — no-op

finishTransaction(dto);
// → processTransaction(): save AccountTransferOperation (WITHDRAW + DEPOSIT)
// → updateApprovalDetail()
// → updateTransactionStatus()
// → completeTransaction()
// → commissionInfoListener.saveCommissionData()
// → promotionListener
// → fixTransferMetadataProcessor

// SWITCH_WALLET_RECEIVE_MONEY is in the notification exclusion:
if (dto.getTransferType() != TransferType.AMAL_EXPRESS_RECEIVE_REMMITANCE
        && dto.getTransferType() != TransferType.SWITCH_WALLET_RECEIVE_MONEY) {
    dto.setEventType("AccountTransferDoneEvent");
    rabbitMQProducer.rabbitProducer(dto);
}
// ← AccountTransferDoneEvent is SKIPPED for SWITCH_WALLET_RECEIVE_MONEY

dto.setEventType("AccountTransferCompletedEvent");
rabbitMQProducer.rabbitProducer(dto);
// → mfs-reporting: CustomerTransactionReport saved
```

---

### Final Balance After SWITCH_WALLET_RECEIVE_MONEY

```
Before:
  Subscriber wallet (SWALLET-SUB-001):   10,000 XAF
  Switch Wallet ("Switch Wallet"):        50,000 XAF

After SUCCESS:
  Subscriber wallet:   13,000 XAF   (+3,000)
  Switch Wallet:       47,000 XAF   (−3,000)

  External (MTN Momo 237671111111): −3,000 XAF  [handled by Switch provider]
```

---

## Complete Side-by-Side Diagram

```
SWITCH_WALLET_DEPOSIT_MONEY                      SWITCH_WALLET_RECEIVE_MONEY
(Subscriber sends OUT)                           (Subscriber collects IN)
════════════════════════════════════════════════════════════════════════════════

mfs-webclient                                    [No webclient UI found]
  SwitchWalletDepositController                    Directly via API
  .sendSwitchMoneyStore()
  ↓ POST /transfer/send-switch-money               ↓ POST /transfer/receive-switch-money
mfs-api-gateway                                  mfs-api-gateway
  JWT auth + proxy                                 JWT auth + proxy
  ↓                                                ↓
mfs-backend-new                                  mfs-backend-new
  SwitchWalletController.sendSwitchMoney()         SwitchWalletController.receiveSwitchMoney()
    loads subscriber wallet                          loads subscriber wallet
    generates transferId "DMS..."                    generates transferId "RMS..."
    builds SwitchWalletDisburseRequest               builds SwitchCollectionRequest
    calls processDisburse()                          calls processCollection()
      ↓                                                ↓
  SwitchWalletCommandService                       SwitchWalletCommandService
  .processDisburse()                               .processCollection()
    builds TransferEventDto:                         [if !testing] SwitchWalletClient
      from = subscriber wallet                         .collectTransaction(request)
      to   = "Switch Wallet"                           → HTTP POST /switch/api/enterprise/collect
      type = DEPOSIT_MONEY                             → Switch prompts sender to authorize
    calls beginTransaction()                         builds TransferEventDto:
      ↓                                                from = "Switch Wallet"
  beginTransaction()                                   to   = subscriber wallet
    scale + validate + commission                       type = RECEIVE_MONEY
    initiateTransaction()                            calls beginTransaction()
      → AccountTransfer STARTED saved                  ↓
    [if !testing] calls                            beginTransaction()
      newThirdPartyService → SwitchWalletClient      scale + validate + commission
        .disburseTransaction(request)                SwitchWalletReceiveMoneyValidationRule
        → HTTP POST /switch/api/enterprise/disburse    → idempotency check: already processed?
        → Switch sends money to beneficiary            initiateTransaction()
    publish AccountTransferCreateEvent               → AccountTransfer STARTED saved
    return "Transaction Processed"                 publish AccountTransferCreateEvent
  ↓                                                return "Collection request initiated"
  if (testing) disburseDemo()                      ↓
    → directly calls handleDisburseCallback()      if (testing) collectDemo()
                                                     → directly calls handleCollectCallback()

══════════ ASYNC: External Switch API Sends Webhook ══════════

  POST /switch/webhook/disburse                    POST /switch/webhook/collect
  SwitchWalletHookController                       SwitchWalletHookController
  .handleDisburseCallback()                        .handleCollectCallback()
    parse JSON, find AccountTransfer by ref          parse JSON, find AccountTransfer by ref
    saveCallbackMetadata() → DB                      saveCallbackMetadata() → DB
    map to TransferEventDto                          map to TransferEventDto
    status = msg.equalsIgnoreCase("SUCCESS")         status = collectionStatus (SUCCESS/PENDING/FAILED)

    if SUCCESS:                                      if PENDING:
      dto.transferStatus = SUCCESS                     suspendTransaction() → status=SUSPENDED
      processTransaction()                           if FAILED:
    else FAILED:                                       rejectTransaction() → status=FAILED
      dto.transferStatus = FAILED                    if SUCCESS:
      rejectTransaction()                              dto.transferStatus = SUCCESS
                                                       processTransaction()

══════════ processTransaction() — Money Moves Here ══════════

  SWITCH_WALLET_DEPOSIT_MONEY path:                SWITCH_WALLET_RECEIVE_MONEY path:
  → enters first if block:                         → skips first if block (not DEPOSIT_MONEY type)
    if (DEPOSIT_MONEY && SUCCESS)                  → normal main flow:
      updateWalletBalances()                           validate + commission + initiateTransaction again
                                                       updateWalletBalances()

  updateWalletBalances() (same method both):
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │ DEBIT:                                                                       │
  │   DEPOSIT_MONEY: subscriber wallet  −(amount + fee)                         │
  │   RECEIVE_MONEY: Switch Wallet      −amount                                 │
  │                                                                              │
  │ CREDIT:                                                                      │
  │   DEPOSIT_MONEY: Switch Wallet      +amount                                 │
  │   RECEIVE_MONEY: subscriber wallet  +amount                                 │
  │                                                                              │
  │ COMMISSION: handleCommissions() → credit commission wallet(s)               │
  └─────────────────────────────────────────────────────────────────────────────┘

  finishTransaction()  (same for both):
    processTransaction() → AccountTransferOperation saved, status = DONE
    updateApprovalDetail()
    updateTransactionStatus()
    completeTransaction()
    saveCommissionData()
    promotionListener
    fixTransferMetadataProcessor

  AccountTransferDoneEvent:
    DEPOSIT_MONEY: SKIPPED (not published in webhook path)
    RECEIVE_MONEY: ALWAYS SKIPPED (excluded by code:
                   dto.getTransferType() != SWITCH_WALLET_RECEIVE_MONEY)

  AccountTransferCompletedEvent:
    BOTH: Published → mfs-reporting
      → RabbitMQConsumer.rabbitConsumer()
      → checkTransferEvent()
      → reportProcessingService.accountTransferCompletedEvent()
      → CustomerTransactionReport saved to reporting DB

══════════ RabbitMQ MoneySubtracted/Added events ══════════

  Both flows publish MoneySubtractedEvent + MoneyAddedEvent
  → mfs-reporting: newTransactionReportingService updates wallet history
```

---

### Code Navigation Reference

```
GATEWAY:
  gw TransferController (proxy for switch endpoints)
  gw SwitchWalletController
  → mfs-api-gateway/controller/SwitchWalletController.java

BACKEND CONTROLLERS:
  SwitchWalletController.sendSwitchMoney()         ← DEPOSIT_MONEY entry
  SwitchWalletController.receiveSwitchMoney()      ← RECEIVE_MONEY entry
  SwitchWalletHookController.handleDisburseCallback() ← DEPOSIT_MONEY webhook
  SwitchWalletHookController.handleCollectCallback()  ← RECEIVE_MONEY webhook
  → application/controller/SwitchWalletController.java
  → application/controller/SwitchWalletHookController.java

CORE SERVICE:
  SwitchWalletCommandService.processDisburse()     ← builds DEPOSIT_MONEY dto
  SwitchWalletCommandService.processCollection()   ← builds RECEIVE_MONEY dto
  SwitchWalletCommandService.disburseDemo()        ← test shortcut for disburse
  SwitchWalletCommandService.collectDemo()         ← test shortcut for collect
  → application/service/SwitchWalletCommandService.java

  NewAccountTransferService.beginTransaction()     ← Phase 1 (save + call API)
  NewAccountTransferService.processTransaction()   ← Phase 2 (money moves)
  NewAccountTransferService.rejectTransaction()    ← failure path
  NewAccountTransferService.suspendTransaction()   ← PENDING path (RECEIVE only)
  → application/service/NewAccountTransferService.java

EXTERNAL API CLIENT:
  SwitchWalletClient.disburseTransaction()  ← POST /switch/api/enterprise/disburse
  SwitchWalletClient.collectTransaction()   ← POST /switch/api/enterprise/collect
  → core-api/dto/remittance/SwitchWalletClient.java  (Feign client interface)

VALIDATION:
  SwitchWalletReceiveMoneyValidationRule  ← idempotency check (RECEIVE_MONEY only)
  → wallet/validator/rule/SwitchWalletReceiveMoneyValidationRule.java

WALLET UPDATES:
  NewWalletService.updateWalletBalances()  ← debit + credit
  NewWalletService.reverseWalletBalances() ← failure reversal
  WalletQueryRepository.withdrawFromWallet() ← SQL UPDATE with concurrency guard
  WalletQueryRepository.depositToWallet()    ← SQL UPDATE credit
  → application/service/NewWalletService.java
  → wallet-query/repository/WalletQueryRepository.java

DB PERSISTENCE:
  AccountTransferListener.initiateTransaction()  ← STARTED record
  AccountTransferListener.processTransaction()   ← DONE + Operations
  SwitchWalletCallbackMetadataQueryRepository    ← raw webhook audit log
  → wallet-query/listener/AccountTransferListener.java

REPORTING (async):
  RabbitMQConsumer.rabbitConsumer()              ← @RabbitListener entry
  ReportProcessingService.accountTransferCompletedEvent() ← CustomerTransactionReport
  → mfs-reporting/listener/rebbitMQ/RabbitMQConsumer.java
  → mfs-reporting/handler/ReportProcessingService.java
```
