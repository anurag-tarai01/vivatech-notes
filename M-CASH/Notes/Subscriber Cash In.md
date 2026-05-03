## Final Flow Summary :
```java
Subscriber Cash-In — Complete Flow

Agent (mobile app)
  │
  ▼ POST /transfer/subscriber-cash-in  {fromAccount=agent wallet, toAccount=subscriber wallet, amount, pincode}
  │
[mfs-api-gateway: TransferController.SubscriberCashIn()]
  │  • JWT validated, DEPOSIT_SUBSCRIBER_CASH_IN privilege checked
  │  • RestTemplate.POST → http://core-backend/transfer/subscriber-cash-in
  │
[mfs-backend-new: TransferController.SubscriberCashIn() line 541]
  │  • Map TransferDto → TransferEventDto
  │  • Generate transferId (e.g., SCI240426.1230.123456)
  │  • Call validateTransferRequestNew(dto)
  │
  ├─[TransferValidatorService.validate()]
  │    • Fetch agent wallet + subscriber wallet from DB
  │    • Run all AbstractTransferValidationRule beans (sorted by order):
  │      1. AccountsVerificationRule → fromWallet must be AGENT/CUSTOMER_CARE,
  │                                    toWallet must be SUBSCRIBER, both ACTIVE
  │      2. PincodeVerificationRule  → subscriber's pincode BCrypt match
  │      3. AccountBalanceLimitValidaionRule → agent balance >= amount + fee
  │      4. ThresholdValidationRule  → amount within subscriber's grade limits
  │      5. MaxMinBalanceLimitValidationRule → agent balance won't go below minimum
  │      6. CoolDownValidationRule   → subscriber not in cooldown period
  │    • If any rule fails → dto.setErrorReason() → flow exits with FAILED response
  │
[NewAccountTransferService.processTransaction()]
  │  • Scale amount to 4 decimal places
  │  • Re-run TransferValidationService.checkTransactionValidations(dto)
  │  • ServiceAndCommissionChargeService.calculateServiceAndCommissionCharge(dto)
  │    → sets dto.serviceCharge, dto.commissionInfos (agent commission wallet + amount)
  │
  ├─[AccountTransferListener.initiateTransaction()]
  │    • Create AccountTransfer entity
  │    • Set transferStatus = STARTED
  │    • Save to SQL Server (account_transfer table) — FIRST DB WRITE
  │
  ├─[RabbitMQProducer.rabbitProducer("AccountTransferCreateEvent")]
  │    • Publish to RabbitMQ exchange "mfs-core-exchange"
  │
  ├─[NewWalletService.updateWalletBalances()]
  │    • walletQueryRepository.withdrawFromWallet(agentWalletId, amount+fee)
  │      → SQL UPDATE wallet SET balance = balance - X WHERE id = ? (rowAffected check)
  │    • Publish "MoneySubtractedEvent" to RabbitMQ
  │    • walletQueryRepository.depositToWallet(subscriberWalletId, amount)
  │      → SQL UPDATE wallet SET balance = balance + X WHERE id = ?
  │    • Publish "MoneyAddedEvent" to RabbitMQ
  │    • walletQueryRepository.depositToWallet(commissionWalletId, commission)
  │    • Publish "MoneyAddedEvent" to RabbitMQ
  │
  ├─[NewThirdPartyService.handleThirdPartyService(dto)]
  │    • No-op for SUBSCRIBER_CASH_IN
  │
  ├─[finishTransaction(dto)]
  │    ├─ AccountTransferListener.processTransaction() → status=DONE,
  │    │    save AccountTransferOperation (WITHDRAW + DEPOSIT rows)
  │    ├─ AccountTransferListener.updateApprovalDetail()
  │    ├─ AccountTransferListener.updateTransactionStatus()
  │    ├─ AccountTransferListener.completeTransaction()
  │    ├─ CommissionInfoListener.saveCommissionData()
  │    ├─ PromotionListener.handlePromotionalOffer()
  │    └─ FixTransferMetadataProcessor.handleMetadataForCompleteTransaction()
  │
  ├─[RabbitMQProducer.rabbitProducer("AccountTransferDoneEvent")]
  │    • For notifications (mfs-notification service picks this up)
  │
  ├─[RabbitMQProducer.rabbitProducer("AccountTransferCompletedEvent")]
  │    • For reporting (mfs-reporting picks this up)
  │
  └─ Return BaseResponseDto { status:"Success", transferId:"SCI240426..." }

                          ↓ (async via RabbitMQ)
[mfs-reporting: RabbitMQConsumer.rabbitConsumer()]
  │  • @RabbitListener on queue "${rabbit.mq.queue.name}"
  │  • message has "transferAggregateId" → calls rabbitMQService.checkTransferEvent()
  │
  ├─[RabbitMQService.checkTransferEvent()]
  │    switch(eventType):
  │    case "AccountTransferCreateEvent"
  │         → newTransactionAuditReportService.accountTransferCreateEvent() [audit trail]
  │    case "AccountTransferCompletedEvent"
  │         → reportProcessingService.accountTransferCompletedEvent() ← MAIN REPORTING
  │         → newTransactionAuditReportService.accountTransferCompletedEvent() [audit]
  │    case "MoneySubtractedEvent" / "MoneyAddedEvent"  (separate message routing)
  │         → newTransactionReportingService.handleMoneySubtractedEvent()
  │         → newTransactionReportingService.handleMoneyAddedEvent()
  │
  └─[ReportProcessingService.accountTransferCompletedEvent()]
       • Check if transferId already processed (idempotency guard)
       • Build CustomerTransactionReport entity:
         - payer (agent): name, msisdn, zone, area, grade, balance before/after
         - payee (subscriber): name, msisdn, balance before/after
         - amount, serviceCharge, commissionInfos
         - transferType = SUBSCRIBER_CASH_IN, status = SUCCESS
       • CustomerTransactionReportRepository.save(report) → reporting DB write
```


## Subscriber Cash-In — Complete Flow Analysis

---

### Step 1: Entry Point

**Gateway class:** `TransferController` **File:** `m-cash/mfs-api-gateway/mfs-api-gateway/api-gateway-application/src/main/java/com/vivacom/mfs/api/gateway/controller/TransferController.java` **Method:** `SubscriberCashIn()` **Exact path:** `POST /transfer/subscriber-cash-in` **Auth:** `@PreAuthorize("@permissionService.check('DEPOSIT_SUBSCRIBER_CASH_IN')")` — only agents or internal agents holding this privilege can call it.

**What the gateway method does — in full:**

java

```java
public Object SubscriberCashIn(@RequestBody @Valid TransferDto requestDto) {
    String uri = coreServerAddress + "/transfer/subscriber-cash-in";
    return getPOSTApiResponse(restTemplate, uri, requestDto, Object.class);
}
```

That's it. The gateway does **zero business logic**. It validates the JWT, checks the privilege, and forwards the raw `TransferDto` body via `RestTemplate` to the backend at `http://core-server/transfer/subscriber-cash-in`.

---

### Step 2: Architecture Classification

**This flow is Service-based, NOT CQRS.**

**Proof from code:**

Old CQRS path is commented out in `TransferController.java` (backend):

java

```java
// CreateAccountTransferCommand command = new CreateAccountTransferCommand(...);
// out = dispatchTransferCommandInActor(command, transferId);
```

Active path is:

java

```java
validateTransferRequestNew(dto);
BaseResponseDto responseDto = newAccountTransferService.processTransaction(dto);
```

`newAccountTransferService` is a `@Service` class called directly. No `commandBus.dispatch()`. No Axon aggregate. This is **Controller → Service → Repository** — service-based.

---

### Step 3: Full Execution Trace (Line-by-Line)

#### 3.1 — Backend Controller entry

**Class:** `TransferController` (backend) **File:** `application/src/main/java/com/vivacom/mfs/application/controller/TransferController.java` **Method:** `SubscriberCashIn()` at line 541

java

```java
TransferEventDto dto = mapper.getMapperFacade().map(requestDto, TransferEventDto.class);
dto.setUpdatedAt(new Date());
dto.setCreatedAt(new Date());
String transferId = utils.getAccountTransferId(dto.getTransferType());
dto.setTransferAggregateId(transferId);
dto.setTransferMetadata("");
validateTransferRequestNew(dto);
BaseResponseDto responseDto = newAccountTransferService.processTransaction(dto);
```

1. Maps `TransferDto` → `TransferEventDto` (the internal working DTO)
2. Sets timestamps
3. Calls `utils.getAccountTransferId(TransferType.SUBSCRIBER_CASH_IN)` — generates a unique transfer ID like `SCI240426.1230.123456` (format: type prefix + date + time + random)
4. Calls `validateTransferRequestNew(dto)` — a local private method that calls `NewAccountTransferService.processTransaction()` is NOT called here. Let me correct: `validateTransferRequestNew(dto)` calls `TransferValidationService.checkTransactionValidations(dto)` directly:

java

```java
private void validateTransferRequestNew(TransferEventDto dto) {
    ValidationResult result = transferValidationService.checkTransactionValidations(dto);
    // throws exception if not valid
}
```

5. Then calls `newAccountTransferService.processTransaction(dto)`

---

#### 3.2 — Validation: TransferValidationService

**Class:** `TransferValidationService` **File:** `application/src/main/java/com/vivacom/mfs/application/service/TransferValidationService.java` **Method:** `checkTransactionValidations(dto)`

java

```java
ValidationResult result = validatorService.validate(dto);
if (!result.isValid()) {
    dto.setErrorReason(result.getError());
}
dto.setPincode(""); // clears pincode from DTO after use
```

Calls `TransferValidatorService.validate(dto)`.

---

#### 3.3 — TransferValidatorService (the rule engine)

**Class:** `TransferValidatorService` **File:** `wallet/src/main/java/com/vivacom/mfs/wallet/service/validator/TransferValidatorService.java` **Method:** `validate(dto)`

What it does:

1. Fetches `fromWallet` from DB: `walletService.getWalletInfoFromDB(dto.getFromAccountAggregateId())` — this is the **Agent's wallet** (the one doing the cash-in)
2. Fetches `toWallet` from DB: `walletService.getWalletInfoFromDB(dto.getToAccountAggregateId())` — this is the **Subscriber's wallet**
3. Fetches `fromUser` and `toUser` from DB via `userService.getUserInfoFromId()`
4. Builds `FromAccountInfo` and `ToAccountInfo` objects with wallet type, balance, status, pincode, grade
5. Loops through all `AbstractTransferValidationRule` beans sorted by `getOrder()` and calls `rule.supports(dto)` first, then `rule.validate(dto, fromAccountInfo, toAccountInfo)` — **stops at first failure**

---

#### 3.4 — Validation Rules that apply to SUBSCRIBER_CASH_IN

**Rule 1: `AccountsVerificationRule`** `supports()` returns `true` for `SUBSCRIBER_CASH_IN`. `validate()` calls `handleSubscriberCashIn()`:

java

```java
private ValidationResult handleSubscriberCashIn(...) {
    List<String> validWalletTypes = Arrays.asList(
        WalletType.AGENT.toString(), WalletType.CUSTOMER_CARE.toString()
    );
    if (!validWalletTypes.contains(fromAccountInfo.getAccountWalletType().toString())) {
        return new ValidationResult(false,
            "account X is not a valid agent or internal agent account");
    }
    if (toAccountInfo.getAccountWalletType() != WalletType.SUBSCRIBER) {
        return new ValidationResult(false,
            "account X is not a valid subscriber account");
    }
    return new ValidationResult(true);
}
```

Then also calls `validateAccountStatus()`:

java

```java
if (WalletStatus.ACTIVE != fromAccountInfo.getAccountStatus()
        && fromAccountInfo.getAccountStatus() != WalletStatus.BARRED_AS_RECEIVER) {
    // fail
}
if (WalletStatus.ACTIVE != toAccountInfo.getAccountStatus()
        && toAccountInfo.getAccountStatus() != WalletStatus.BARRED_AS_SENDER) {
    // fail
}
```

**Rule 2: `ThresholdValidationRule`** — checks min/max transaction limits and daily/weekly/monthly limits from the user's grade profile.

**Rule 3: `MaxMinBalanceLimitValidationRule`** — checks if the agent's balance after deduction won't go below the minimum allowed balance.

**Rule 4: `PincodeVerificationRule`** — checks the subscriber's pincode sent in the request against the BCrypt-stored pincode.

**Rule 5: `AccountBalanceLimitValidaionRule`** — checks that the agent has enough balance to cover the transfer amount + service charge.

**Rule 6: `CoolDownValidationRule`** — checks if the subscriber has a cooldown period since their last transaction.

If **any** rule fails, `dto.setErrorReason(result.getError())` is set and the validation service returns with the error. The caller checks this field.

---

#### 3.5 — NewAccountTransferService.processTransaction()

**Class:** `NewAccountTransferService` **File:** `application/src/main/java/com/vivacom/mfs/application/service/NewAccountTransferService.java` **Method:** `processTransaction(dto)`

`SUBSCRIBER_CASH_IN` does **not** match any of the special-case `TransferType` checks at the top of the method (those are for Switch Wallet, Deposit types, etc.). So it goes into the **main branch**:

**Sub-step A: Scale the amount**

java

```java
BigMoney amount = dto.getAmount();
dto.setAmount(MfsUtils.getScaledMoney(amount));
```

Rounds to 4 decimal places.

**Sub-step B: Validate (again via the same rule engine)**

java

```java
dto = transferValidationService.checkTransactionValidations(dto);
```

Yes — validation runs twice. The first call was in the controller's `validateTransferRequestNew()`. This is the second run inside `processTransaction()`.

**Sub-step C: Calculate service charge + commission**

java

```java
dto = serviceAndCommissionChargeService.calculateServiceAndCommissionCharge(dto);
```

`ServiceAndCommissionChargeService.calculateServiceAndCommissionCharge()` — reads the service charge profile for `SUBSCRIBER_CASH_IN` transfer type, calculates the fee to deduct from the agent and the commission to credit to the agent's commission wallet. Sets `dto.setServiceCharge()` and `dto.setCommissionInfos()`.

**Sub-step D: Check for error after commission calc**

java

```java
if (!StringUtils.isEmpty(dto.getErrorReason())) {
    dto = storeFailedTransaction(dto);
    dto.setEventType("AccountTransferFailedEvent");
    rabbitMQProducer.rabbitProducer(dto);
    return FAILED response;
}
```

**Sub-step E: Save initial transaction record to DB**

java

```java
accountTransferListener.initiateTransaction(dto);
```

(See Step 4 for detail)

**Sub-step F: Publish AccountTransferCreateEvent to RabbitMQ**

java

```java
dto.setEventType("AccountTransferCreateEvent");
rabbitMQProducer.rabbitProducer(dto);
```

**Sub-step G: Check metadata** (not applicable for cash-in — no special metadata required)

**Sub-step H: Update wallet balances**

java

```java
dto = newWalletService.updateWalletBalances(dto);
```

(See Step 4 for detail)

**Sub-step I: Third party call**

java

```java
dto = newThirdPartyService.handleThirdPartyService(dto);
```

For `SUBSCRIBER_CASH_IN`, `NewThirdPartyService` finds no matching third-party provider, so this is a no-op and returns the dto unchanged.

**Sub-step J: Finish transaction**

java

```java
finishTransaction(dto);
```

(See Step 4 for detail)

**Sub-step K: Publish AccountTransferDoneEvent**

java

```java
dto.setEventType("AccountTransferDoneEvent");
rabbitMQProducer.rabbitProducer(dto);
```

**Sub-step L: Publish AccountTransferCompletedEvent**

java

```java
dto.setEventType("AccountTransferCompletedEvent");
rabbitMQProducer.rabbitProducer(dto);
```

**Sub-step M: Return success**

java

```java
responseDto.setStatusCode(StatusCode.SUCCESS);
responseDto.setStatus("Success");
responseDto.setMessage("Transaction Success, Transfer Id: " + dto.getTransferAggregateId());
return responseDto;
```

---

### Step 4: Business Logic — Exact Methods

#### 4.1 — initiateTransaction() — first DB save

**Class:** `AccountTransferListener` **File:** `wallet-query/src/main/java/com/vivacom/mfs/wallet/query/listener/AccountTransferListener.java` **Method:** `initiateTransaction(dto)` at line 207

Creates an `AccountTransfer` entity and saves it to DB with `transferStatus = STARTED`. Fields written:

- `transferAggregateId`, `fromAccountAggregateId`, `toAccountAggregateId`
- `fromAccountId`, `toAccountId` (wallet DB integer IDs)
- `amount`, `transferType = SUBSCRIBER_CASH_IN`
- `transferStatus = STARTED`
- `createdAt`, `serviceCharge`
- `transferInitiedById`, `clientIpAddress`, `narration`

Then saves `AccountTransferMetadata` (empty string for cash-in) and calls `queryrepository.saveAndFlush(a)`.

---

#### 4.2 — updateWalletBalances() — actual money movement

**Class:** `NewWalletService` **File:** `application/src/main/java/com/vivacom/mfs/application/service/NewWalletService.java` **Method:** `updateWalletBalances(dto)`

**Debit (Agent wallet):**

java

```java
totalWithdrawAmount = dto.getAmount().plus(serviceCharge); // amount + fee
BigMoney withdrawWalletCurrentBalance = getCurrentWalletBalance(dto.getFromAccountAggregateId());
dto.setWithDrawBalanceData(withdrawWalletCurrentBalance, totalWithdrawAmount);
int rowAffected = walletQueryRepository.withdrawFromWallet(
    dto.getFromAccountAggregateId(),
    totalWithdrawAmount.getAmount(), new Date(),
    dto.getTransferAggregateId(), oldTransferId, lastTransferId
);
if (rowAffected != 1) throw new MFSException("There was some error, Please try again.");
```

`walletQueryRepository.withdrawFromWallet()` is a `@Modifying @Query` that subtracts from the wallet's `balance` field in SQL Server. The `rowAffected != 1` check is the concurrency guard — if no row was updated, it means another transaction already touched this wallet simultaneously.

Then sends `MoneySubtractedEvent` to RabbitMQ.

**Credit (Subscriber wallet):**

java

```java
totalDepositAmount = dto.getAmount(); // no commission deducted for cash-in recipient
BigMoney depositWalletCurrentBalance = getCurrentWalletBalance(dto.getToAccountAggregateId());
dto.setDepositBalanceData(depositWalletCurrentBalance, totalDepositAmount);
walletQueryRepository.depositToWallet(
    dto.getToAccountAggregateId(),
    totalDepositAmount.getAmount(), new Date()
);
```

Then sends `MoneyAddedEvent` to RabbitMQ.

**Commission deposit:**

java

```java
dto = handleCommissions(dto, amt03Commission, agentCommissionWalletId, agentCommission);
```

Deposits the agent's earned commission into their commission wallet via `walletQueryRepository.depositToWallet(commissionWalletId, ...)`.

---

#### 4.3 — finishTransaction() — final DB updates

**Class:** `NewAccountTransferService` **Method:** `finishTransaction(dto)`

Calls in sequence:

1. `accountTransferListener.processTransaction(dto)` — sets `transferStatus = DONE`, saves `AccountTransferOperation` records (WITHDRAW from agent, DEPOSIT to subscriber) via `operationQueryRepository.save()`
2. `accountTransferListener.updateApprovalDetail(dto)` — sets approver info on the `AccountTransfer` record
3. `accountTransferListener.updateTransactionStatus(dto)` — updates `AccountTransfer.transferStatus` based on transfer type
4. `accountTransferListener.completeTransaction(dto)` — any post-completion operations specific to transfer type (for `SUBSCRIBER_CASH_IN` — no special case)
5. `commissionInfoListener.saveCommissionData(dto)` — saves commission records to the commission info table
6. `promotionListener.handlePromotionalOffer(dto)` — checks and saves any promotional offer applicable
7. `fixTransferMetadataProcessor.handleMetadataForCompleteTransaction(dto)` — handles metadata cleanup

---

### Step 5: Database Impact

|Entity|Table|What changes|
|---|---|---|
|`AccountTransfer`|`account_transfer`|Created with `STARTED`, updated to `DONE`|
|`AccountTransferMetadata`|`account_transfer_metadata`|Created with transfer metadata|
|`AccountTransferOperation`|`account_transfer_operation`|2 rows: WITHDRAW (agent), DEPOSIT (subscriber)|
|`Wallet` (agent)|`wallet`|`balance` decremented by `amount + serviceCharge`|
|`Wallet` (subscriber)|`wallet`|`balance` incremented by `amount`|
|`Wallet` (commission)|`wallet`|`balance` incremented by agent's commission amount|
|`CommissionInfo`|`commission_info`|Commission record saved via `commissionInfoListener`|

---

### Step 6: mfs-reporting Microservice

**How reporting is triggered:** Via **RabbitMQ messaging** — not a REST call.

**Exact trigger point in `NewAccountTransferService.processTransaction()`:**

Three separate RabbitMQ publishes happen:

java

```java
// 1. When transaction starts
dto.setEventType("AccountTransferCreateEvent");
rabbitMQProducer.rabbitProducer(dto);      // line ~150 of NewAccountTransferService

// 2. When transaction succeeds
dto.setEventType("AccountTransferDoneEvent");
rabbitMQProducer.rabbitProducer(dto);      // notification purposes

// 3. The one reporting cares about:
dto.setEventType("AccountTransferCompletedEvent");
rabbitMQProducer.rabbitProducer(dto);      // this is what writes to CustomerTransactionReport
```

**Publisher class:** `RabbitMQProducer` **File:** `application/src/main/java/com/vivacom/mfs/application/RabbitMQProducer.java` **Method:** `rabbitProducer(TransferEventDto)` **Transport:** `rabbitTemplate.convertAndSend(EXCHANGE_NAME, ROUTING_KEY, eventMessageMap)` — sends to exchange `mfs-core-exchange` with routing key `mfs-core-key`.

Also, inside `NewWalletService.updateWalletBalances()`, two more events are published per wallet operation:

java

```java
sendMessageToRabbitMQ(..., "MoneySubtractedEvent");  // after agent debit
sendMessageToRabbitMQ(..., "MoneyAddedEvent");       // after subscriber credit
sendMessageToRabbitMQ(..., "MoneyAddedEvent");       // after commission credit
```

---

**Consumer side in mfs-reporting:**

**Entry class:** `RabbitMQConsumer` **File:** `mfs-reporting/src/main/java/com/vivacom/mfs/reporting/listener/rebbitMQ/RabbitMQConsumer.java` **Method:** `rabbitConsumer(Map<?,?> message)` — annotated with `@RabbitListener(queues = "${rabbit.mq.queue.name}")`

**Routing logic in `rabbitConsumer()`:**

java

```java
if (message.containsKey("identifier")) {
    rabbitMQService.checkEvent(message);         // user domain events
} else if (message.containsKey("transferAggregateId")) {
    rabbitMQService.checkTransferEvent(message); // ← cash-in messages go here
} else if (message.containsKey("transferId")) {
    rabbitMQService.checkUpdateWalletHistoryEvent(message); // ← MoneyAdded/Subtracted go here
}
```

**For `AccountTransferCompletedEvent`:** `RabbitMQService.checkTransferEvent()` → `case "AccountTransferCompletedEvent":` → calls:

- `reportProcessingService.accountTransferCompletedEvent(transferDto)` — creates and saves `CustomerTransactionReport` entity in the reporting DB
- `newTransactionAuditReportService.accountTransferCompletedEvent(transferDto)` — creates audit trail

**For `AccountTransferCreateEvent`:** `case "AccountTransferCreateEvent":` → `newTransactionAuditReportService.accountTransferCreateEvent(transferDto)` — saves the initial audit record.

**For `MoneySubtractedEvent` / `MoneyAddedEvent`:** `rabbitMQService.checkUpdateWalletHistoryEvent()` → calls:

- `newTransactionReportingService.handleMoneySubtractedEvent(walletId, transferId, updatedAt, currentBalance, amount)` — updates wallet history in the reporting DB
- `newTransactionReportingService.handleMoneyAddedEvent(walletId, transferId, updatedAt, currentBalance, amount)` — updates wallet history in the reporting DB

**What `reportProcessingService.accountTransferCompletedEvent()` saves to reporting DB:** Creates a `CustomerTransactionReport` entity with:

- payer info (agent: name, msisdn, zone, area, grade)
- payee info (subscriber: name, msisdn)
- transfer amount, service charge, commission info
- balance before and after for both wallets
- transfer type string (`SUBSCRIBER_CASH_IN`)
- transfer ID, timestamp, status = SUCCESS

Saves via `CustomerTransactionReportRepository.save(report)`.

---

### Step 7: Code Navigation Guide

```java
GATEWAY
  TransferController.java
  → m-cash/mfs-api-gateway/mfs-api-gateway/api-gateway-application/
      src/main/java/com/vivacom/mfs/api/gateway/controller/TransferController.java

BACKEND CONTROLLER
  TransferController.java (backend — different class, same name)
  → m-cash/mfs-backend-new/application/
      src/main/java/com/vivacom/mfs/application/controller/TransferController.java

VALIDATION LAYER
  TransferValidationService.java
  → application/src/main/java/com/vivacom/mfs/application/service/TransferValidationService.java

  TransferValidatorService.java (rule engine)
  → wallet/src/main/java/com/vivacom/mfs/wallet/service/validator/TransferValidatorService.java

  AccountsVerificationRule.java (wallet type + status check)
  → wallet/src/main/java/com/vivacom/mfs/wallet/service/validator/rule/AccountsVerificationRule.java

  ThresholdValidationRule.java
  MaxMinBalanceLimitValidationRule.java
  PincodeVerificationRule.java
  AccountBalanceLimitValidaionRule.java
  CoolDownValidationRule.java
  → wallet/src/main/java/com/vivacom/mfs/wallet/service/validator/rule/

CORE SERVICE
  NewAccountTransferService.java
  → application/src/main/java/com/vivacom/mfs/application/service/NewAccountTransferService.java

  ServiceAndCommissionChargeService.java  (commission calculation)
  NewWalletService.java                   (balance updates)
  → application/src/main/java/com/vivacom/mfs/application/service/

DB LAYER
  AccountTransferListener.java            (AccountTransfer DB operations)
  → wallet-query/src/main/java/com/vivacom/mfs/wallet/query/listener/AccountTransferListener.java

  WalletQueryRepository.java              (wallet balance SQL queries)
  → wallet-query/src/main/java/com/vivacom/mfs/wallet/query/repository/WalletQueryRepository.java

MESSAGING
  RabbitMQProducer.java                   (backend sender)
  → application/src/main/java/com/vivacom/mfs/application/RabbitMQProducer.java

REPORTING (separate service)
  RabbitMQConsumer.java                   (entry point — @RabbitListener)
  → mfs-reporting/src/main/java/com/vivacom/mfs/reporting/listener/rebbitMQ/RabbitMQConsumer.java

  RabbitMQService.java                    (event router)
  → mfs-reporting/src/main/java/com/vivacom/mfs/reporting/audit/RabbitMQService.java

  ReportProcessingService.java            (CustomerTransactionReport writer)
  → mfs-reporting/src/main/java/com/vivacom/mfs/reporting/handler/ReportProcessingService.java

  NewTransactionAuditReportService.java   (audit trail)
  NewTransactionReportingService.java     (wallet history)
  → mfs-reporting/src/main/java/com/vivacom/mfs/reporting/handler/
```

---

### Step 8: Debugging Guide

**Breakpoint 1 — Gateway entry** `TransferController.SubscriberCashIn()` (gateway) Check: Is the JWT valid? Is `DEPOSIT_SUBSCRIBER_CASH_IN` privilege present? Is `requestDto` populated correctly with `fromAccountAggregateId`, `toAccountAggregateId`, `amount`, `pincode`?

**Breakpoint 2 — Backend controller entry** `TransferController.SubscriberCashIn()` (backend, line 541) Check: Is the `transferId` generated correctly? What does `dto.getTransferType()` return?

**Breakpoint 3 — Validation rule engine** `TransferValidatorService.validate()` — step through `sortedValitatorRules` in the for loop. For each rule, check what `rule.supports(dto)` returns. If `supports()` is true, step into `rule.validate()`. This is where most "invalid account" or "insufficient balance" errors originate.

**Breakpoint 4 — Wallet type check** `AccountsVerificationRule.handleSubscriberCashIn()` — verify `fromAccountInfo.getAccountWalletType()` is `AGENT` or `CUSTOMER_CARE`, and `toAccountInfo.getAccountWalletType()` is `SUBSCRIBER`.

**Breakpoint 5 — Commission calculation** `ServiceAndCommissionChargeService.calculateServiceAndCommissionCharge()` — watch `dto.getServiceCharge()` and `dto.getCommissionInfos()` after this call. If commission is wrong, it's here.

**Breakpoint 6 — Balance deduction** `NewWalletService.updateWalletBalances()` — watch `withdrawWalletCurrentBalance` before and after. Watch `rowAffected` — if it's not 1, there's a concurrency conflict.

**Breakpoint 7 — DB save (initiateTransaction)** `AccountTransferListener.initiateTransaction()` — confirm `AccountTransfer` entity is being saved with `STARTED` status.

**Breakpoint 8 — RabbitMQ publish** `RabbitMQProducer.rabbitProducer(TransferEventDto)` — check `dto.getEventType()`. You'll hit this three times per successful cash-in: `AccountTransferCreateEvent`, `AccountTransferDoneEvent`, `AccountTransferCompletedEvent`.

**Breakpoint 9 — Reporting consumer** `RabbitMQConsumer.rabbitConsumer()` — confirm message arrives. Check which branch it enters (`transferAggregateId` branch for transfer events).

**Breakpoint 10 — Reporting DB save** `ReportProcessingService.accountTransferCompletedEvent()` near line 258 `repository.save(report)` — inspect the `report` object fields before save.

**Logs to check:**

```
mfs-backend-new logs:
  "validating account verification rule of transfer id {}"
  "wallet balance updated for wallet id: {} and amount: {}, current balance {}, updated balance {}"
  "transfer success for transfer id: {}"
  "Transfer EventMessage Sent For ID: {}"

mfs-reporting logs:
  "Transfer EventMessage Received For ID: {}"
  "processing AccountTransferCompletedEvent"
  "AccountTransferCompletedEvent processed"
```

---
