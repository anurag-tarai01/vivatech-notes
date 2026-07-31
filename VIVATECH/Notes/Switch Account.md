### Switch Account — In Depth

#### Where is it defined?

**Constant:** `Constants.SWITCH_WALLET = "Switch Wallet"` — this string is the `aggregateId` of the switch wallet row in the `Wallet` table.

**Used in:** `SwitchWalletCommandService.java` — hardcoded reference:


```java
String switchWalletAggregateId = Constants.SWITCH_WALLET; // "Switch Wallet"
```

There is **no separate `WalletType.SWITCH`** enum value. The switch wallet is of type `ADMIN` in the DB (like all system wallets). It is distinguished only by its `aggregateId = "Switch Wallet"`.

There is also a companion commission wallet: `Constants.SWITCH_WALLET_COMMISSION = "switch_wallet_commission2"` and an admin account `Constants.SWITCH_WALLET_ADMIN = "switch_wallet_123"`.

#### What is its purpose?

The switch wallet is a **liquidity float** — a pool of money held by the mfs-backend-new system that represents funds held with the **external Switch Wallet provider** (a third-party cross-network payment rail operating in Cameroon/XAF).

When a subscriber on the m-cash network wants to send money to someone on **another mobile network** (e.g., MTN MoMo, Orange Money), the m-cash system:

1. Debits the subscriber's wallet
2. Credits the Switch Wallet (internal accounting — the system now "owes" this money to the switch provider)
3. Calls the Switch API to disburse to the other network

When someone on another network sends money **into** m-cash:

1. The Switch provider calls back via webhook
2. m-cash debits the Switch Wallet (internal accounting — switch provider "paid" this money)
3. Credits the target subscriber's wallet

The switch wallet balance at any point = total funds currently held in float with the switch provider that haven't been settled yet.

#### The 4 switch transfer types and what they mean:

|TransferType|Direction|Initiated by|Approval needed|
|---|---|---|---|
|`SWITCH_WALLET_DEPOSIT`|External cash → Switch Wallet|Admin|✅ Yes (2-step)|
|`SWITCH_WALLET_WITHDRAW`|Switch Wallet → External|Admin|✅ Yes (2-step)|
|`SWITCH_WALLET_DEPOSIT_MONEY`|Subscriber Wallet → Switch Wallet|Subscriber|❌ No (webhook-confirmed)|
|`SWITCH_WALLET_RECEIVE_MONEY`|Switch Wallet → Subscriber Wallet|External trigger (webhook)|❌ No (webhook-confirmed)|

---

### 3. Deposit & Withdraw Involving Switch

#### Switch Wallet Deposit (Admin-initiated, 2-step)

**Step 1 — Admin initiates:** `POST /transfer/deposit-switch-wallet` → `SwitchWalletController.switchWalletDeposit()` (backend) → calls `newAccountTransferService.beginTransaction(dto)` where `dto.transferType = SWITCH_WALLET_DEPOSIT`

Inside `beginTransaction()`:

- Validates, calculates commission
- `accountTransferListener.initiateTransaction(dto)` — saves `AccountTransfer` with `status = STARTED`
- `SWITCH_WALLET_DEPOSIT` is in the exclusion list — **no wallet balance change here**
- Publishes `AccountTransferCreateEvent` to RabbitMQ
- Returns `"Transaction Processed"` to admin

**Step 2 — Another admin approves:** `POST /transfer/approve-deposit-switch-wallet` → `SwitchWalletController.approveSwitchWalletDeposit()` (backend) → calls `validateTransferRequestNew(dto)` then `newAccountTransferService.processTransaction(dto)`

Inside `processTransaction()`, `SWITCH_WALLET_DEPOSIT` is NOT in the bypass list, so full flow runs:

- `newWalletService.updateWalletBalances(dto)`:
    - **DEBIT** `fromAccountAggregateId` (external / float source wallet)
    - **CREDIT** Switch Wallet (`"Switch Wallet"`) — the system's switch pool grows
- `finishTransaction(dto)` — status → `DONE`
- Publishes `AccountTransferCompletedEvent`

**Rejection:** `POST /transfer/reject-deposit-switch-wallet` → `newAccountTransferService.rejectTransaction(dto)` — no wallet changes, status → `FAILED`

---

#### Switch Wallet Withdraw (Admin-initiated, 2-step)

**Step 1:** `POST /transfer/withdraw-switch-wallet` → `SwitchWalletController.switchWalletWithdraw()` → `newAccountTransferService.beginTransaction(dto)` where `dto.transferType = SWITCH_WALLET_WITHDRAW`

- Same as deposit step 1 — saves `STARTED` record, no balance change

**Step 2:** `POST /transfer/approve-withdraw-switch-wallet` → `newAccountTransferService.processTransaction(dto)`

- `newWalletService.updateWalletBalances(dto)`:
    - **DEBIT** Switch Wallet (`"Switch Wallet"`) — pool shrinks
    - **CREDIT** `toAccountAggregateId` (agent wallet or external target)
- `finishTransaction(dto)` — status → `DONE`

---

#### Send Switch Money (Subscriber sends OUT — `SWITCH_WALLET_DEPOSIT_MONEY`)

**Subscriber initiates:** `POST /transfer/send-switch-money` → `SwitchWalletController.sendSwitchMoney(request)` (backend)

java

```java
// Method body:
String fromAccount = w.getAggregateId();                    // subscriber's wallet
String switchWalletAggregateId = Constants.SWITCH_WALLET;  // "Switch Wallet"
TransferEventDto transferDto = TransferEventDto.builder()
    .fromAccountAggregateId(fromAccount)
    .toAccountAggregateId(switchWalletAggregateId)
    .transferType(TransferType.SWITCH_WALLET_DEPOSIT_MONEY)
    ...build();
newAccountTransferService.beginTransaction(transferDto);
```

Inside `beginTransaction()`:

1. Validates and calculates commission
2. `accountTransferListener.initiateTransaction(dto)` — saves `STARTED` record
3. Because `!switchWalletTesting`:

java

```java
   dto = newThirdPartyService.handleThirdPartyService(dto);
   // → calls SwitchWalletService.sendMoney() → HTTP POST to external Switch API
```

4. Publishes `AccountTransferCreateEvent`
5. Returns `"Transaction Processed"` immediately — **wallets not yet changed**

**Switch API calls back via webhook:** `POST /switch/webhook/disburse` → `SwitchWalletHookController.handleDisburseCallback(request)`

java

```java
String status = disburseRequest.getMessage().equalsIgnoreCase("SUCCESS") ? "SUCCESS" : "FAILED";
TransferEventDto dto = mapper.map(accountTransfer, TransferEventDto.class);
dto.setTransferType(TransferType.SWITCH_WALLET_DEPOSIT_MONEY);

if ("SUCCESS".equalsIgnoreCase(status)) {
    dto.setTransferStatus(TransferStatus.SUCCESS);
    newAccountTransferService.processTransaction(dto);   // ← money moves here
} else {
    dto.setTransferStatus(TransferStatus.FAILED);
    newAccountTransferService.rejectTransaction(dto);    // ← no money moves
}
```

Inside `processTransaction()`, because `dto.getTransferType() == SWITCH_WALLET_DEPOSIT_MONEY`:

java

```java
if (dto.getTransferType() == TransferType.SWITCH_WALLET_DEPOSIT_MONEY) {
    if (dto.getTransferStatus() == TransferStatus.SUCCESS) {
        newWalletService.updateWalletBalances(dto);  // ← THIS is where money moves
        finishTransaction(dto);
        dto.setEventType("AccountTransferCompletedEvent");
        rabbitMQProducer.rabbitProducer(dto);
        return SUCCESS;
    } else if (dto.getTransferStatus() == TransferStatus.FAILED) {
        dto = newWalletService.reverseWalletBalances(dto);   // ← reversal if failed
        storeFailedTransaction(dto);
        return FAILED;
    }
}
```

`updateWalletBalances(dto)` for `SWITCH_WALLET_DEPOSIT_MONEY`:

- **DEBIT** subscriber wallet: `amount + serviceCharge`
- **CREDIT** Switch Wallet (`"Switch Wallet"`): `amount`
- If commissions configured: **CREDIT** commission wallet

---

#### Receive Switch Money (Money comes IN — `SWITCH_WALLET_RECEIVE_MONEY`)

**Subscriber/admin initiates collection:** `POST /transfer/collect-switch-money` → gateway `SwitchWalletController.receiveSwitchMoney()` → backend `SwitchWalletController.receiveSwitchMoney(request)`

java

```java
String switchWalletAggregateId = Constants.SWITCH_WALLET;   // "Switch Wallet"
String toAccount = w.getAggregateId();                       // subscriber's wallet
TransferEventDto transferDto = TransferEventDto.builder()
    .fromAccountAggregateId(switchWalletAggregateId)  // money comes FROM switch
    .toAccountAggregateId(toAccount)                  // INTO subscriber wallet
    .transferType(TransferType.SWITCH_WALLET_RECEIVE_MONEY)
    ...build();
switchWalletCommandService.processCollection(switchWalletDisburseRequest, amount, w, ...);
```

Inside `processCollection()`:

java

```java
newAccountTransferService.beginTransaction(transferDto);
// → initiates the transfer (STARTED record)
// → calls SwitchWalletClient.collectTransaction(request) → HTTP POST to Switch API
// → no wallet balance change yet
```

**Switch API calls back:** `POST /switch/webhook/collect` → `SwitchWalletHookController.handleCollectCallback(request)`

java

```java
if ("SUCCESS".equalsIgnoreCase(status)) {
    dto.setTransferStatus(TransferStatus.SUCCESS);
    newAccountTransferService.processTransaction(dto);
}
```

Inside `processTransaction()`, `SWITCH_WALLET_RECEIVE_MONEY` is in the exclusion list for validation/commission (it was already done in `beginTransaction`), so it goes to `updateWalletBalances(dto)`:

- **DEBIT** Switch Wallet (`"Switch Wallet"`): pool shrinks (money was "collected" from external network)
- **CREDIT** subscriber wallet: subscriber receives the money

`SWITCH_WALLET_RECEIVE_MONEY` is also excluded from the `AccountTransferDoneEvent` notification:

java

```java
if (dto.getTransferType() != TransferType.SWITCH_WALLET_RECEIVE_MONEY) {
    dto.setEventType("AccountTransferDoneEvent");
    rabbitMQProducer.rabbitProducer(dto);
}
```

So only `AccountTransferCompletedEvent` is published for this type (one notification instead of two).

---

### 4. End-to-End Example — Subscriber Sends Money via Switch

**Scenario:** Subscriber with wallet `SWALLET-SUB-001` sends 5000 XAF to an MTN MoMo number `237670000000`. Switch wallet aggregateId = `"Switch Wallet"`.

```
Initial balances:
  Subscriber wallet (SWALLET-SUB-001):  10,000 XAF
  Switch Wallet ("Switch Wallet"):       50,000 XAF
  Switch commission wallet:               2,000 XAF
  Service charge configured:                50 XAF
  Commission for switch wallet:            100 XAF
```

**Step 1:** `POST /transfer/send-switch-money`

`SwitchWalletController.sendSwitchMoney()`:

- Looks up subscriber's wallet: `walletQueryRepository.findOneByUserId(userId)`
- Builds `TransferEventDto`: from=`SWALLET-SUB-001`, to=`"Switch Wallet"`, amount=5000, type=`SWITCH_WALLET_DEPOSIT_MONEY`
- Generates transferId: `DMS260427.1430.A1B2C3`
- Calls `SwitchWalletCommandService.processDisburse()`

`SwitchWalletCommandService.processDisburse()`:

- Calls `newAccountTransferService.beginTransaction(dto)`

`NewAccountTransferService.beginTransaction()`:

1. `TransferValidationService.checkTransactionValidations(dto)` — validates subscriber wallet ACTIVE, balance >= 5050
2. `ServiceAndCommissionChargeService.calculateServiceAndCommissionCharge(dto)` — sets `serviceCharge=50`, `commissionInfos=[{switchCommissionWallet, 100}]`
3. `AccountTransferListener.initiateTransaction(dto)` — writes to DB:

```
   AccountTransfer: id=DMS260427.1430.A1B2C3, status=STARTED,
                    from=SWALLET-SUB-001, to="Switch Wallet", amount=5000
```

4. `newThirdPartyService.handleThirdPartyService(dto)`:
    - → `SwitchWalletService.sendMoney()` → HTTP POST to Switch API with beneficiary=`237670000000`, amount=5000, reference=`DMS260427.1430.A1B2C3`
    - Switch API accepts and returns pending
5. `rabbitMQProducer.rabbitProducer(dto)` → publishes `AccountTransferCreateEvent`
6. Returns `"Transaction Processed, Transfer Id: DMS260427.1430.A1B2C3"`

```
Balances UNCHANGED after Step 1:
  Subscriber wallet: 10,000 XAF  (not yet debited)
  Switch Wallet:     50,000 XAF  (not yet credited)
```

**Step 2:** Switch API sends webhook to `POST /switch/webhook/disburse`

json

```json
{
  "message": "Success", "status": 0,
  "reference": "DMS260427.1430.A1B2C3",
  "payoutStatus": "SUCCESS", "payoutAmount": 5000
}
```

`SwitchWalletHookController.handleDisburseCallback()`:

- Finds `AccountTransfer` by `reference = DMS260427.1430.A1B2C3`
- `status = "SUCCESS"` → sets `dto.transferStatus = SUCCESS`
- Calls `newAccountTransferService.processTransaction(dto)`

`NewAccountTransferService.processTransaction()`:

- Matches `SWITCH_WALLET_DEPOSIT_MONEY` special branch
- `newWalletService.updateWalletBalances(dto)`:
    - `totalWithdrawAmount = 5000 + 50 = 5050`
    - `withdrawWalletCurrentBalance = getCurrentWalletBalance("SWALLET-SUB-001")` = 10,000
    - `walletQueryRepository.withdrawFromWallet("SWALLET-SUB-001", 5050, now, "DMS260427.1430.A1B2C3", oldTransferId, lastTransferId)`
    - `rowAffected = 1` ✅ — subscriber debited
    - Publishes `MoneySubtractedEvent` for reporting
    - `depositWalletCurrentBalance = getCurrentWalletBalance("Switch Wallet")` = 50,000
    - `walletQueryRepository.depositToWallet("Switch Wallet", 5000, now)`
    - Switch Wallet credited
    - Publishes `MoneyAddedEvent` for reporting
    - `handleCommissions()` → `walletQueryRepository.depositToWallet("switch_wallet_commission2", 100, now)`
    - Commission wallet credited
- `finishTransaction(dto)`:
    - `AccountTransferListener.processTransaction()` — saves `AccountTransferOperation` rows (WITHDRAW from subscriber, DEPOSIT to switch)
    - Status updated to `DONE`
- Publishes `AccountTransferCompletedEvent` → RabbitMQ → reporting

```
Final balances:
  Subscriber wallet (SWALLET-SUB-001):    4,950 XAF  (−5050)
  Switch Wallet ("Switch Wallet"):        55,000 XAF  (+5000)
  Switch commission wallet:               2,100 XAF  (+100)
  MTN MoMo account 237670000000:          5,000 XAF  (external, handled by Switch provider)
```

---

### 5. Core Wallet Update Logic

#### Where exactly balance is updated

All balance changes happen through exactly two methods in `WalletQueryRepository`:

java

```java
// DEBIT — with optimistic lock
@Modifying
@Query(value = "UPDATE Wallet SET amount = amount - :debitAmount, updated_at = :updatedAt, " +
               "transfer_id = :newTransferId " +
               "WHERE aggregate_id = :walletId " +
               "AND (:oldTransferId IS NULL OR (transfer_id = :oldTransferId OR transfer_id = :lastTransferId))",
       nativeQuery = true)
int withdrawFromWallet(String walletId, BigDecimal debitAmount, Date updatedAt,
                       String newTransferId, String oldTransferId, String lastTransferId);

// CREDIT — no lock needed
@Modifying
@Query(value = "UPDATE Wallet SET amount = amount + :creditAmount, updated_at = :updatedAt " +
               "WHERE aggregate_id = :walletId",
       nativeQuery = true)
void depositToWallet(String walletId, BigDecimal creditAmount, Date updatedAt);
```

Both are `@Modifying` JPA queries — they execute a native SQL `UPDATE` directly on the database. There is no intermediate entity fetch-and-save pattern. This is intentional: it avoids Hibernate's first-level cache masking stale reads in concurrent scenarios.

#### How concurrency is handled

The `withdrawFromWallet()` method uses a **manual optimistic locking** pattern via `transfer_id`:

java

```java
// Before debit:
String oldTransferId = getLastTransferId(walletId);
//  → walletQueryRepository.findByAggregateId(walletId).getTransferId()
//  → reads the last successful transfer ID stored in the wallet row

// The debit SQL:
WHERE aggregate_id = :walletId
AND (:oldTransferId IS NULL
     OR transfer_id = :oldTransferId
     OR transfer_id = :lastTransferId)
```

**Scenario — two concurrent cash-outs from same subscriber:**

- Thread A reads `oldTransferId = "SCI-001"`, balance = 10,000
- Thread B reads `oldTransferId = "SCI-001"`, balance = 10,000
- Thread A executes `UPDATE ... WHERE transfer_id = 'SCI-001'` → **succeeds**, sets `transfer_id = 'SCO-002'`, balance = 8,000
- Thread B executes `UPDATE ... WHERE transfer_id = 'SCI-001'` → **fails** (0 rows affected, transfer_id is now `'SCO-002'`)
- Thread B gets `rowAffected != 1` → throws `MFSException("There was some error, Please try again.")`
- Thread B's transaction rolls back — **no double debit**

#### How failures are handled

**Validation failure** (before wallet touch):

java

```java
// In NewAccountTransferService.processTransaction():
if (!StringUtils.isEmpty(dto.getErrorReason())) {
    dto = storeFailedTransaction(dto);
    dto.setEventType("AccountTransferFailedEvent");
    rabbitMQProducer.rabbitProducer(dto);
    return FAILED response;
}
// storeFailedTransaction():
//   accountTransferListener.handleFailedTransaction(dto)  → saves with status=FAILED
//   accountTransferListener.updateTransactionStatusForRejectedTransaction(dto)
// No wallet balance change
```

**Third-party call failure** (after wallet debited):

java

```java
// In NewAccountTransferService.processTransaction():
dto = newThirdPartyService.handleThirdPartyService(dto);
if (!StringUtils.isEmpty(dto.getErrorReason())) {
    dto = processReversal(dto);  // ← reverses both debit and credit
    return FAILED response;
}

// processReversal():
//   newWalletService.reverseWalletBalances(dto)
//     → walletQueryRepository.depositToWallet(fromWallet, totalWithdrawAmount)  // re-credit payer
//     → walletQueryRepository.withdrawFromWallet(toWallet, totalDepositAmount)  // re-debit receiver
//     → reverseCommission(dto)  // reverse all commission credits
```

**Webhook failure** (Switch API reports failure):

java

```java
// In SwitchWalletHookController.handleDisburseCallback():
if ("FAILED") {
    dto.setTransferStatus(TransferStatus.FAILED);
    newAccountTransferService.rejectTransaction(dto);
    // rejectTransaction():
    //   storeFailedTransaction(dto)  → no wallet balance change
    //   publishes AccountTransferFailedEvent
}
```

For `SWITCH_WALLET_DEPOSIT_MONEY` specifically, a special path in `processTransaction()` also handles failure:

java

```java
} else if (dto.getTransferStatus().equals(TransferStatus.FAILED)) {
    dto = newWalletService.reverseWalletBalances(dto);   // reverse if already debited
    dto = storeFailedTransaction(dto);
    dto.setEventType("AccountTransferFailedEvent");
    rabbitMQProducer.rabbitProducer(dto);
}
```

---

### 6. Design Reasoning — Why This Architecture

#### Why a Switch Wallet

Without the switch wallet, the system would have no way to track how much money is currently held with the external switch provider. Every time a subscriber sends money out via Switch, that money isn't gone — it's in the external provider's system. The switch wallet balance represents the **total float held externally**. It also enables admins to manually top up (`SWITCH_WALLET_DEPOSIT`) or withdraw (`SWITCH_WALLET_WITHDRAW`) the float when reconciling with the provider. The balance always answers: "how much can we disburse through switch right now?"

#### Why a Separate Commission Wallet

From `NewWalletService.handleCommissions()`: commission credits are deposited into a separate wallet ID per commission recipient. This is because commission rules differ per transfer type and per actor tier. Keeping commission in a separate wallet means:

- Commission can be paid out independently via `COMMISSION_PAYMENT` transfer type
- Balance auditing is clean — the agent's main wallet shows only real transactions, commission wallet shows only earned commissions
- Commission can be suspended (`SuspendAdminCommissionPaymentCommand2`) without affecting the main wallet

#### Why the Two-Phase Pattern for Switch Transfers

`beginTransaction()` → external API call → webhook → `processTransaction()`. This pattern exists because the external switch provider is async — it doesn't immediately confirm success. The system saves the transfer as `STARTED`, fires the external API request, and waits for the callback. Wallet balances only move **after** the webhook confirms success. This prevents the system from debiting a user for a transfer that the external network never processed.

The `is.switch.wallet.testing` flag in `NewAccountTransferService` short-circuits this by calling `SwitchWalletCommandService.disburseDemo()` / `collectDemo()` which directly call the webhook handler in the same process — enabling testing without a real external switch provider.

---

### 7. Code Navigation Reference

```
System wallet IDs:
  Constants.java
  → core-api/src/main/java/com/vivacom/mfs/common/Constants.java

All TransferType + WalletType enums:
  AccountTransfer.kt
  → mfs-api-gateway/core-api/src/main/java/com/vivacom/mfs/core/api/accountTransfer/AccountTransfer.kt

All wallet balance SQL (debit/credit):
  WalletQueryRepository.java
  → wallet-query/src/main/java/com/vivacom/mfs/wallet/query/repository/WalletQueryRepository.java

The central balance update engine:
  NewWalletService.java  (updateWalletBalances, handleCommissions, reverseWalletBalances)
  → application/src/main/java/com/vivacom/mfs/application/service/NewWalletService.java

The transaction orchestrator:
  NewAccountTransferService.java  (processTransaction, beginTransaction, rejectTransaction)
  → application/src/main/java/com/vivacom/mfs/application/service/NewAccountTransferService.java

Switch entry points:
  SwitchWalletController.java  (deposit-switch-wallet, withdraw-switch-wallet, send-switch-money, collect-switch-money)
  → application/src/main/java/com/vivacom/mfs/application/controller/SwitchWalletController.java

  SwitchWalletCommandService.java  (processCollection, processDisburse)
  → application/src/main/java/com/vivacom/mfs/application/service/SwitchWalletCommandService.java

  SwitchWalletHookController.java  (handleDisburseCallback, handleCollectCallback)
  → application/src/main/java/com/vivacom/mfs/application/controller/SwitchWalletHookController.java

DB record persistence:
  AccountTransferListener.java  (initiateTransaction, processTransaction, completeTransaction)
  → wallet-query/src/main/java/com/vivacom/mfs/wallet/query/listener/AccountTransferListener.java

RabbitMQ publisher (to reporting):
  RabbitMQProducer.java
  → application/src/main/java/com/vivacom/mfs/application/RabbitMQProducer.java
```
