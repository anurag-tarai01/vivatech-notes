#### All Wallet Types in the System

Every wallet in the system is a row in the `Wallet` table (`wallet-query/entity/Wallet.java`) with a `type` field of type `WalletType` and an `aggregateId` (string, used as wallet identifier everywhere). Here are all wallet types defined in the system, divided into two groups:

**User wallets** (created when a user is registered and approved):

|WalletType|Owner|Created by|
|---|---|---|
|`SUBSCRIBER`|Subscriber|`SubscriberRegistrationSaga`|
|`AGENT`|Agent|`AgentRegistrationSaga`|
|`AGENT_COMMISION`|Agent|`AgentRegistrationSaga`|
|`CUSTOMER_CARE`|Internal Agent (Customer Care)|`CreateCustomerCareSaga`|
|`INTERNAL_AGENT_COMMISSION`|Internal Agent|`CreateCustomerCareSaga`|
|`ADMIN`|All admin types|`CreateNetworkAdminSaga` etc.|
|`ADMIN_COMMISSION`|Admin|same sagas|
|`MERCHANT`|Merchant|`MerchantRegistrationSaga`|
|`OUTLET`|Outlet|`MerchantRegistrationSaga`|
|`THIRD_PARTY`|Third Party Admin|`ThirdPartyAdminSaga`|
|`BILLER`|Biller Admin|`BillerRegistrationSaga`|
|`AMAL_EXPRESS`|Amal Express integration|`AmalExpressSaga`|

**System wallets** (fixed, seeded at setup time, each with a hardcoded `aggregateId` from `Constants.java`):

| Constant                                 | aggregateId value              | Purpose                                         |
| ---------------------------------------- | ------------------------------ | ----------------------------------------------- |
| `Constants.SWITCH_WALLET`                | `"Switch Wallet"`              | Liquidity pool for cross-network money movement |
| `Constants.AMT01_WALLET_ID`              | `"AMT01"`                      | Master treasury / float wallet                  |
| `Constants.AMT02_WALLET_ID`              | `"AMT02"`                      | Secondary treasury                              |
| `Constants.AMT03_WALLET_ID`              | `"AMT03"`                      | Commission collection pool                      |
| `Constants.AMT04_WALLET_ID`              | `"AMT04"`                      | Topup/reseller pool                             |
| `Constants.AMT06_WALLET_ID`              | `"AMT06"`                      | _(defined but usage not found in active flows)_ |
| `Constants.AMAL_EXPRESS_WALLET_ID`       | `"Amal Express Wallet"`        | Amal Express integration float                  |
| `Constants.AMAL_BANK_WALLET_ID`          | `"AMAL_BANK"`                  | Amal Bank integration float                     |
| `Constants.AMAL_BANK_TRANSFER_WALLET_ID` | `"Amal Bank Transfer Account"` | Amal Bank transfer account                      |
| `Constants.RESELLER_WALLET_ID`           | `"Reseller Wallet"`            | Reseller topup float                            |

---

#### Standard Debit Pattern (used everywhere in `NewWalletService`)

java

```java
// Step 1: Read current balance
BigMoney currentBalance = getCurrentWalletBalance(walletId);
// → walletQueryRepository.findByAggregateId(walletId).getBalance()

// Step 2: Debit atomically with optimistic lock
int rowAffected = walletQueryRepository.withdrawFromWallet(
    walletId,
    totalWithdrawAmount.getAmount(),
    new Date(),
    newTransferId,   // the current transfer's ID — stored in wallet.transfer_id
    oldTransferId,   // the previous transfer's ID — used as WHERE condition
    lastTransferId   // a secondary fallback transfer ID
);
// SQL:
// UPDATE Wallet SET amount = amount - :debitAmount, transfer_id = :newTransferId
// WHERE aggregate_id = :walletId
// AND (:oldTransferId IS NULL OR (transfer_id = :oldTransferId OR transfer_id = :lastTransferId))

// Step 3: Concurrency guard
if (rowAffected != 1) throw new MFSException("There was some error, Please try again.");

// Step 4: Publish MoneySubtractedEvent to RabbitMQ for reporting
sendMessageToRabbitMQ(walletId, amount, currentBalance, transferId, "MoneySubtractedEvent");
```

#### Standard Credit Pattern

java

```java
// Step 1: Read current balance
BigMoney currentBalance = getCurrentWalletBalance(walletId);

// Step 2: Credit (no concurrency lock needed — adding is safe)
walletQueryRepository.depositToWallet(walletId, totalDepositAmount.getAmount(), new Date());
// SQL:
// UPDATE Wallet SET amount = amount + :creditAmount WHERE aggregate_id = :walletId

// Step 3: Publish MoneyAddedEvent to RabbitMQ for reporting
sendMessageToRabbitMQ(walletId, amount, currentBalance, transferId, "MoneyAddedEvent");
```

> **Why the lock only on debit?** Because debit can cause overdraft. The `WHERE transfer_id = oldTransferId` clause means the update only succeeds if the wallet hasn't been touched by another concurrent transaction since you read its balance. If two transactions hit the same wallet simultaneously, only one will get `rowAffected == 1`. The other throws an exception. Credits don't need this — adding money can't produce an invalid state.

---

#### All TransferTypes in the system

From `AccountTransfer.kt`, the full list of 80+ transfer types is defined in `enum class TransferType`. For money flow purposes, the core ones are:

```
User-to-user transfers:
  SUBSCRIBER_CASH_IN            Agent → Subscriber (agent pays, subscriber receives)
  SUBSCRIBER_CASH_OUT           Subscriber → Agent (subscriber pays, agent receives)
  SUBSCRIBER_TO_SUBSCRIBER_P2P  Subscriber → Subscriber
  AGENT_TO_SUBSCRIBER           Agent → Subscriber direct
  SUBSCRIBER_MERCHANT_PAYMENT   Subscriber → Merchant
  MERCHANT_CASH_IN              Internal Agent → Merchant

System/Float transfers:
  AMT01_TO_AGENT                Float (AMT01) → Agent (to fund agent wallet)
  AMT01_DEPOSIT                 External deposit into AMT01
  COMMISSION_PAYMENT            Commission wallet → Agent main wallet

Switch transfers (cross-network):
  SWITCH_WALLET_DEPOSIT         Admin-initiated: external money → Switch Wallet (pending approval)
  SWITCH_WALLET_WITHDRAW        Admin-initiated: Switch Wallet → external (pending approval)
  SWITCH_WALLET_DEPOSIT_MONEY   Subscriber sends money OUT via Switch (disburse to other network)
  SWITCH_WALLET_RECEIVE_MONEY   Subscriber receives money IN via Switch (collect from other network)
```