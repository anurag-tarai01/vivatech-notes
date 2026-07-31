# MFS Backend: Commission Disbursement Flow — Complete Analysis

> A senior-developer walkthrough for junior developers. Every method call, every table, every bug — traced end to end.

---

## TABLE OF CONTENTS

1. [Big Picture: What Is Commission Disbursement?](#1-big-picture)
2. [Step 1 — How Commission Is Generated Initially](#2-commission-generation)
3. [Step 2 — Where Commission Is Stored (AMT03 and CommissionInfo)](#3-commission-storage)
4. [Step 3 — How CommissionInfo Entries Are Created](#4-commissioninfo-creation)
5. [Step 4 — CommissionDisbursementProfile Configuration](#5-disbursement-profile)
6. [Step 5 — How processDisbursement() Works Internally](#6-processdisbursement)
7. [Step 6 — How the Original Transfer Is Identified](#7-original-transfer)
8. [Step 7 — Agent / Resale Agent / Distributor Hierarchy Resolution](#8-hierarchy)
9. [Step 8 — Why SUBSCRIBER_CASH_OUT Currently Works](#9-cash-out-works)
10. [Step 9 — Why Other Transfer Types Fail](#10-others-fail)
11. [Step 10 — fromAccountAggregateId vs toAccountAggregateId](#11-account-ids)
12. [Step 11 — gPayAccountTransferService.processTransaction() Deep Dive](#12-gpay-flow)
13. [Step 12 — Which Tables Are Updated](#13-tables)
14. [Complete Flow Examples](#14-examples)
    - SUBSCRIBER_CASH_OUT ✅
    - SUBSCRIBER_CASH_IN ❌
    - RESALE_TO_AGENT_TRANSFER ❌
    - AGENT_TO_RESALE_AGENT_TRANSFER ❌
15. [Code Quality Issues and Refactoring Plan](#15-refactoring)

---

## 1. Big Picture: What Is Commission Disbursement?

Think of it like this: every time a subscriber does a **Cash Out** at an agent's shop, the system charges them a service fee. A portion of that fee (the commission) is collected into a central "commission wallet" called **AMT03**. 

Later, an admin triggers a **disbursement run** that says: "For all the Cash Out commissions sitting in AMT03, distribute them to the right agents and their parent agents."

```
Subscriber Cash Out Transaction (e.g., $100 withdrawal, $2 service charge)
    │
    ▼
$2 goes into AMT03 (the MFS commission holding wallet)
$1.20 tagged for Agent (say 60%)
$0.50 tagged for Resale Agent (say 25%)
$0.30 stays for Admin / MFS (15%)
    │
    ▼
Admin triggers processDisbursement(SUBSCRIBER_CASH_OUT)
    │
    ▼
$1.20 transferred from AMT03 → Agent's wallet
$0.50 transferred from AMT03 → Resale Agent's wallet
$0.30 stays in AMT03 (ADMIN role = do nothing)
```

---

## 2. How Commission Is Generated Initially

**File:** `TransferCommissionInterceptor.java` (legacy path) and `ServiceAndCommissionChargeService.java` (new path in `GPayAccountTransferService`)

When a transfer is processed, `calculateServiceAndCommissionCharge()` is called. Here's the chain:

```
GPayAccountTransferService.processTransaction(dto)
	  └── executeInterceptorsAndInitiate(dto)
        └── serviceAndCommissionChargeService.calculateServiceAndCommissionCharge(dto)
              └── setServiceChargeAndCommission(dto)
                    └── profileService.getServiceChargeAndCommission(requestDto)
                          ← returns ServiceChargeSlabInfo + CommissionSlabInfo
                    └── setServiceCharge(dto, response)   ← sets dto.serviceCharge
                    └── setCommission(dto, agentId, response)
                          ← builds List<CommissionInfo> with accountAggregateId = "AMT03"
                          ← sets dto.commissionInfos
```

**Real Example — SUBSCRIBER_CASH_OUT, amount = $100:**

The service charge profile for SUBSCRIBER_CASH_OUT might say:
- Service charge: $2.00 (2%)
- Commission to AMT03: $2.00 (100% of the service charge goes to AMT03)

Inside `setCommission()`, since `SUBSCRIBER_CASH_OUT` is not `AMAL_EXPRESS_*` or `EXTERNAL_AGENT_*`, it falls to the `else` branch:
```java
mfsCommission = dto.getServiceCharge().minus(agentCommissions);
// agentCommissions = 0 here (no agent wallet directly credited at this stage)
// so mfsCommission = $2.00
```

A `CommissionInfo` object is created:
```java
commissionInfo.setAccountAggregateId("AMT03");  // goes to central wallet
commissionInfo.setAmount($2.00);
commissionInfo.setPaidStatus(false);
commissionInfo.setTransferId("SCO-20240515-XXXX");  // the transfer that earned it
```

This is added to `dto.commissionInfos`.

---

## 3. Where Commission Is Stored

### The AMT03 Wallet

`AMT03` is a **hardcoded wallet aggregate ID** (see `Constants.java`):
```java
public static final String AMT03_WALLET_AGGREGATE_ID = "AMT03";
```

It is the **MFS's own commission collection account**. Think of it as a bank holding account for earned-but-not-yet-distributed commissions.

### The `commission_info` Table (Entity: `CommissionInfo`)

```
commission_info table:
┌─────┬──────────────────┬────────────────────────┬───────────────┬────────┬──────────────┬────────────┐
│ id  │ transfer_type    │ transfer_id            │ account_       │ amount │ paid_status  │ updated_at │
│     │                  │ (original txn that     │ aggregate_id  │        │ (false=owed) │            │
│     │                  │  earned this comm.)    │ (AMT03 or     │        │              │            │
│     │                  │                        │  agent wallet)│        │              │            │
├─────┼──────────────────┼────────────────────────┼───────────────┼────────┼──────────────┼────────────┤
│  1  │ SUBSCRIBER_      │ SCO-20240515-0001      │ AMT03         │  2.00  │ false        │ ...        │
│     │ CASH_OUT         │                        │               │        │              │            │
│  2  │ SUBSCRIBER_      │ SCO-20240515-0002      │ AMT03         │  3.50  │ false        │ ...        │
│     │ CASH_OUT         │                        │               │        │              │            │
└─────┴──────────────────┴────────────────────────┴───────────────┴────────┴──────────────┴────────────┘
```

**Key insight:** `account_aggregate_id = "AMT03"` means "this commission belongs to MFS's pot and needs to be disbursed." Once disbursed, `paid_status` becomes `true`.

---

## 4. How CommissionInfo Entries Are Created

**File:** `CommissionInfoListener.java` → `saveCommissionData()` method

After `GPayAccountTransferService.finishTransaction()` is called:
```java
// Inside finishTransaction():
commissionInfoListener.saveCommissionData(dto);
```

`saveCommissionData()` takes `dto.commissionInfos` (the list built during commission calculation) and persists each entry to the `commission_info` table:

```java
List<CommissionInfo> entityCommissionInfo = mapper.mapAsList(
    aggregateCommissionInfos, CommissionInfo.class
);
for (CommissionInfo commissionInfo : entityCommissionInfo) {
    commissionInfo.setTransactionDate(dto.getUpdatedAt());
    commissionInfo.setPaidStatus(false);   // ← always starts unpaid
    commissionInfo.setTransferId(dto.getTransferAggregateId()); // original txn ID
}
commissionInfoRepository.save(entityCommissionInfo);
```

So after every successful `SUBSCRIBER_CASH_OUT`, you get a new row in `commission_info` with `paid_status = false`, waiting to be disbursed.

---

## 5. CommissionDisbursementProfile Configuration

This is the "recipe" for how to split the commission pot. There are **two tables**:

### `commission_disbursement_profiles` table

```
┌──────────┬──────────────────────┬────────┬───────────────┐
│ id       │ transfer_type        │ status │ created_by_id │
├──────────┼──────────────────────┼────────┼───────────────┤
│ CDP-001  │ SUBSCRIBER_CASH_OUT  │ ACTIVE │ 1             │
└──────────┴──────────────────────┴────────┴───────────────┘
```

### `commission_disbursements` table (the slabs/rules per profile)

```
┌──────────┬──────────┬──────────────────┬────────────┐
│ id       │profile_id│ role             │ percentage │
├──────────┼──────────┼──────────────────┼────────────┤
│ CD-001   │ CDP-001  │ AGENT            │ 60.0       │
│ CD-002   │ CDP-001  │ RESALE_AGENT     │ 25.0       │
│ CD-003   │ CDP-001  │ ADMIN            │ 15.0       │ ← ADMIN = stays in AMT03, do nothing
└──────────┴──────────┴──────────────────┴────────────┘
```

This config tells the system: "When disbursing SUBSCRIBER_CASH_OUT commissions, give 60% to the Agent who performed the cash out, 25% to their parent Resale Agent, 15% keeps in AMT03 (ADMIN gets it)."

---

## 6. How processDisbursement() Works Internally

**File:** `CommissionDisbursementProcessingService.java`

```java
public void processDisbursement(TransferType transferType) {
    // Step 1: Find the active config profile for this transfer type
    CommissionDisbursementProfile profile = profileRepository
        .findByTransferTypeAndStatus(transferType, ACTIVE);

    // Step 2: Find all unpaid commissions in AMT03 for this transfer type
    List<CommissionInfo> unpaidCommissions = commissionInfoQueryRepository
        .findByAccountAggregateIdAndTransferTypeAndPaidStatus(
            "AMT03", transferType, false
        );

    // Step 3: Process each commission one by one
    for (CommissionInfo commission : unpaidCommissions) {
        try {
            processSingleCommission(commission, profile);
        } catch (Exception e) {
            log.error("Failed to process commission {}", commission.getId(), e);
            // NOTE: continues to next item — one failure doesn't stop the batch
        }
    }
}
```

Inside `processSingleCommission()`:

```java
private void processSingleCommission(CommissionInfo commission, 
                                     CommissionDisbursementProfile profile) {
    // Step A: Find the original transfer that generated this commission
    AccountTransfer originalTransfer = 
        accountTransferQueryRepository.findOneByTransferAggregateId(commission.getTransferId());

    // Step B: Get human-readable sender/receiver types from MfsUtils
    String senderTypeStr = utils.getSenderType(profile.getTransferType());
    // e.g., for SUBSCRIBER_CASH_OUT → "Subscriber"
    
    String receiverTypeStr = utils.getReceiverType(profile.getTransferType());
    // e.g., for SUBSCRIBER_CASH_OUT → "Agent"

    // Step C: For each role in the disbursement config (Agent, Resale Agent, etc.)
    for (CommissionDisbursement slab : profile.getDisbursements()) {
        UserType role = slab.getRole();
        
        if (role == UserType.ADMIN) continue; // stays in AMT03
        
        String targetWalletId = null;
        
        // Generic mapping attempt (this FAILS for most types — see section 9)
        if (role matches senderTypeStr) {
            targetWalletId = originalTransfer.getFromAccountAggregateId();
        } else if (role matches receiverTypeStr) {
            targetWalletId = originalTransfer.getToAccountAggregateId();
        }
        
        // Hardcoded override ONLY for SUBSCRIBER_CASH_OUT
        if (profile.getTransferType() == SUBSCRIBER_CASH_OUT) {
            // ... detailed hierarchy resolution (see section 8)
        }
        
        // Step D: Calculate payout amount
        BigMoney payoutAmount = utils.calculatePercentage(commission.getAmount(), slab.getPercentage());
        
        // Step E: Build a new transfer from AMT03 → target wallet
        TransferEventDto transferDto = TransferEventDto.builder()
            .fromAccountAggregateId("AMT03")
            .toAccountAggregateId(targetWalletId)
            .amount(payoutAmount)
            .transferType(determineDisbursementTransferType(role))
            // e.g., COMMISSION_DISBURSEMENT_TO_AGENT
            .build();
        
        // Step F: Execute the actual money movement
        gPayAccountTransferService.processTransaction(transferDto);
    }
    
    // Step G: Mark the commission as paid
    commission.setPaidStatus(true);
    commissionInfoQueryRepository.save(commission);
}
```

---

## 7. How the Original Transfer Is Identified

The `CommissionInfo` entity has a `transferId` field that stores the `transferAggregateId` of the original transaction (e.g., `"SCO-20240515-0001"`).

The lookup uses a two-step fallback:
```java
// Try by transferAggregateId first (the human-readable prefixed ID)
AccountTransfer originalTransfer = 
    accountTransferQueryRepository.findOneByTransferAggregateId(commission.getTransferId());

// If not found, try by database ID
if (originalTransfer == null) {
    originalTransfer = accountTransferQueryRepository.findOne(commission.getTransferId());
}
```

The `AccountTransfer` entity contains:
- `fromAccountAggregateId` — the wallet that **sent** the money (e.g., the subscriber's wallet)
- `toAccountAggregateId` — the wallet that **received** the money (e.g., the agent's wallet)

This is what the disbursement code uses to figure out "which agent was involved?"

---

## 8. Agent / Resale Agent / Distributor Hierarchy Resolution

**Only implemented for SUBSCRIBER_CASH_OUT.** Here's how each role is resolved:

### AGENT
```java
targetWalletId = originalTransfer.getToAccountAggregateId();
// Because in Cash Out: subscriber sends to agent → toAccount = agent wallet
```

### RESALE_AGENT (the Agent's parent)
```java
// 1. Get the agent's wallet from the original transfer
Wallet agentWallet = walletQueryRepository.findByAggregateId(
    originalTransfer.getToAccountAggregateId()
);

// 2. Get the agent's user profile
UserInfo agentInfo = userService.getUserInfoFromId(agentWallet.getUserId());

// 3. Get the resale agent (= agent's parent)
UserInfo resaleAgentInfo = userService.getUserInfoFromId(agentInfo.getParentAgentId());

// 4. Get the resale agent's AGENT-type wallet
targetWalletId = walletQueryRepository
    .findOneByUserIdAndType(resaleAgentInfo.getUserId(), WalletType.AGENT)
    .getId();
```

### DISTRIBUTOR_AGENT (the Resale Agent's parent)
```java
// Same as above but one more level up:
Wallet agentWallet = walletQueryRepository.findByAggregateId(
    originalTransfer.getToAccountAggregateId()
);
UserInfo agentInfo = userService.getUserInfoFromId(agentWallet.getUserId());
UserInfo resaleAgentInfo = userService.getUserInfoFromId(agentInfo.getParentAgentId());
UserInfo distributorAgentInfo = userService.getUserInfoFromId(resaleAgentInfo.getParentAgentId());
targetWalletId = walletQueryRepository
    .findOneByUserIdAndType(distributorAgentInfo.getUserId(), WalletType.AGENT)
    .getId();
```

**Visually:**
```
DISTRIBUTOR_AGENT (grandparent)
    └── RESALE_AGENT (parent)
            └── AGENT (the one who performed the transaction)
                    └── Subscriber Cash Out
```
All three can receive a cut. The code walks up the tree using `parentAgentId`.

---

## 9. Why SUBSCRIBER_CASH_OUT Currently Works

For `SUBSCRIBER_CASH_OUT`:
- `getSenderType()` returns `"Subscriber"`
- `getReceiverType()` returns `"Agent"`

The profile has role `AGENT`. The generic code tries:
```java
if (role.name().equalsIgnoreCase(senderTypeStr))  // "AGENT" vs "Subscriber" → NO
else if (role.name().equalsIgnoreCase(receiverTypeStr)) // "AGENT" vs "Agent" → YES ✓
    targetWalletId = originalTransfer.getToAccountAggregateId();
```

So `targetWalletId` gets set to the agent's wallet from `toAccountAggregateId`.

Then **immediately afterward**, there's a hardcoded block:
```java
if (profile.getTransferType() == TransferType.SUBSCRIBER_CASH_OUT) {
    if (role.name().equalsIgnoreCase("Agent")) {
        targetWalletId = originalTransfer.getToAccountAggregateId(); // same result, redundant
    } else if (role == RESALE_AGENT) {
        // walk parent chain...
    } else if (role == DISTRIBUTOR_AGENT) {
        // walk grandparent chain...
    }
}
```

So it works because:
1. The generic mapping sets `targetWalletId` for AGENT correctly (by luck — "Agent" == "Agent").
2. The hardcoded block then also handles RESALE_AGENT and DISTRIBUTOR_AGENT.

---

## 10. Why Other Transfer Types Fail

### SUBSCRIBER_CASH_IN

`getSenderType(SUBSCRIBER_CASH_IN)` → falls to `default` → returns `""`
`getReceiverType(SUBSCRIBER_CASH_IN)` → falls to `default` → returns `""`

The profile might have `role = AGENT`. The generic mapping tries:
```java
if ("AGENT".equalsIgnoreCase(""))   → false
else if ("AGENT".equalsIgnoreCase("")) → false
```
`targetWalletId` stays `null`.

Then it hits:
```java
if (profile.getTransferType() == TransferType.SUBSCRIBER_CASH_OUT) {
    // this is SUBSCRIBER_CASH_IN, so this block is SKIPPED
}
```

Result: `targetWalletId == null` → logged as warning, skipped → **commission never disbursed.**

### RESALE_TO_AGENT_TRANSFER

`getSenderType()` → `""` (not in the switch)
`getReceiverType()` → `""` (not in the switch)

In `RESALE_TO_AGENT_TRANSFER`, the **Resale Agent sends to Agent**:
- `fromAccountAggregateId` = Resale Agent's wallet
- `toAccountAggregateId` = Agent's wallet

Disbursement needs to credit the Agent. But since `getReceiverType()` returns `""`, the generic mapping fails, and there's no hardcoded block for this type. → **commission never disbursed.**

### AGENT_TO_RESALE_AGENT_TRANSFER

Same situation — `getReceiverType()` returns `""`. 

Here the **Agent sends to Resale Agent**:
- `fromAccountAggregateId` = Agent's wallet
- `toAccountAggregateId` = Resale Agent's wallet

The Agent should receive a commission (they're sending money up), but the code can't figure out who to credit because there's no handler for this type. → **commission never disbursed.**

---

## 11. fromAccountAggregateId vs toAccountAggregateId for Agent Identification

This is the most important table to understand:

| Transfer Type              | fromAccountAggregateId | toAccountAggregateId | Who Is the Agent? | Which Field? |
|----------------------------|------------------------|----------------------|-------------------|--------------|
| SUBSCRIBER_CASH_OUT        | Subscriber wallet      | Agent wallet         | the receiver      | **to** |
| SUBSCRIBER_CASH_IN         | Agent wallet           | Subscriber wallet    | the sender        | **from** |
| RESALE_TO_AGENT_TRANSFER   | Resale Agent wallet    | Agent wallet         | Agent = receiver  | **to** |
| AGENT_TO_RESALE_AGENT      | Agent wallet           | Resale Agent wallet  | Agent = sender    | **from** |
| SUBSCRIBER_TOPUP           | Subscriber wallet      | MFS wallet (AMT01)   | Agent not involved| N/A |

**The bug in a nutshell:** The code only hardcodes logic for `SUBSCRIBER_CASH_OUT` (where the agent is `toAccount`). For other types, the code doesn't know which wallet belongs to the agent.

---

## 12. gPayAccountTransferService.processTransaction() — Complete Internal Flow

```
processTransaction(dto)
│
├── requiresInitialization(dto.transferType)?
│   ├── YES (most normal transfers including COMMISSION_DISBURSEMENT_TO_AGENT)
│   │   └── executeInterceptorsAndInitiate(dto)
│   │         ├── MfsUtils.getScaledMoney(amount)        ← normalize decimal scale
│   │         ├── transferValidationService.checkTransactionValidations(dto)
│   │         │     ← checks balance, user status, limits, etc.
│   │         ├── serviceAndCommissionChargeService.calculateServiceAndCommissionCharge(dto)
│   │         │     ← calculates service charge + builds commission info list
│   │         └── accountTransferListener.initiateTransaction(dto)
│   │               ← writes initial DB record with status=STARTED
│   │               ← sends "AccountTransferCreateEvent" to RabbitMQ
│   │
│   └── NO (already initiated types like SWITCH_WALLET, AMT03_TO_INTERNAL_AGENT, etc.)
│         ← skip initiation, go straight to core execution
│
├── dto.errorReason set?
│   └── YES → handleFailedTransactionAndReturnResponse(dto)
│               ← writes FAILED status to DB
│               ← sends "AccountTransferFailedEvent" to RabbitMQ
│               ← returns FAILED response
│
└── executeTransactionCore(dto)
      │
      ├── checkMetadata(dto)
      │     ← for COMMISSION_PAYMENT and ACCOUNT_MANAGER_COMMISSION:
      │       validates required metadata is present
      │
      ├── dto.errorReason set? → handleFailedTransactionAndReturnResponse()
      │
      ├── newWalletService.updateWalletBalances(dto)
      │     ← deducts from fromAccount, adds to toAccount
      │     ← updates wallet balance table
      │
      ├── newThirdPartyService.handleThirdPartyService(dto)
      │     ← for external integrations (Amal Express, Switch Wallet, etc.)
      │     ← for COMMISSION_DISBURSEMENT types = no-op
      │
      ├── dto.errorReason set?
      │   └── YES → processReversal(dto) ← undo the balance update
      │               ← sends "AccountTransferFailedEvent" to RabbitMQ
      │               ← returns FAILED
      │
      └── finishTransaction(dto)
            ├── accountTransferListener.processTransaction(dto)
            │     ← updates the transfer record (metadata, details)
            ├── accountTransferListener.updateApprovalDetail(dto)
            ├── accountTransferListener.updateTransactionStatus(dto)
            ├── accountTransferListener.completeTransaction(dto)
            │     ← marks transfer as COMPLETED in DB
            ├── commissionInfoListener.saveCommissionData(dto)
            │     ← saves any commission earned by THIS disbursement transfer
            │     ← (disbursement itself earns no commission, list is empty)
            ├── promotionListener.handlePromotionalOffer(dto)
            └── fixTransferMetadataProcessor.handleMetadataForCompleteTransaction(dto)
```

**Sends to RabbitMQ:** 
- `AccountTransferCreateEvent` on initiation
- `AccountTransferCompletedEvent` on success
- `AccountTransferFailedEvent` on failure

---

## 13. Which Tables/Entities Are Updated During Disbursement

When `processDisbursement(SUBSCRIBER_CASH_OUT)` runs successfully, here's every table touched:

| Table | Operation | What Changes |
|-------|-----------|--------------|
| `commission_info` | UPDATE | `paid_status = true` for the source commission record |
| `account_transfer` | INSERT | New row for the disbursement transfer (AMT03 → Agent) |
| `account_transfer` | UPDATE | Status changes: STARTED → COMPLETED |
| `wallet` | UPDATE | AMT03 balance decreases by payout amount |
| `wallet` | UPDATE | Agent's wallet balance increases by payout amount |
| RabbitMQ (not a table) | PUBLISH | `AccountTransferCreateEvent`, `AccountTransferCompletedEvent` |

For each `CommissionDisbursement` slab that results in a transfer, you get one new row in `account_transfer` and two wallet balance updates.

---

## 14. Complete Flow Examples

### ✅ SUBSCRIBER_CASH_OUT — Works Correctly

**Scenario:** Subscriber withdraws $100 at an agent's shop. Service charge = $2.00 (2%).
**Config:** Agent=60%, Resale Agent=25%, Admin=15%.

#### Phase 1: Original Transaction

```
Subscriber Wallet (SUB-001)  ──$100──►  Agent Wallet (AGT-001)
                              ──$2.00──► AMT03 (service charge)

commission_info row created:
  transfer_id = "SCO-20240515-0001"
  account_aggregate_id = "AMT03"
  amount = 2.00
  paid_status = false
  transfer_type = SUBSCRIBER_CASH_OUT
```

#### Phase 2: Admin Triggers processDisbursement(SUBSCRIBER_CASH_OUT)

```
Step 1: Find active profile for SUBSCRIBER_CASH_OUT → CDP-001
Step 2: Query commission_info WHERE account_aggregate_id='AMT03' 
        AND transfer_type='SUBSCRIBER_CASH_OUT' AND paid_status=false
        → finds our row [id=1, amount=2.00, transferId="SCO-20240515-0001"]

Step 3: processSingleCommission(commission[id=1], profile[CDP-001])

  3a: Find original transfer by "SCO-20240515-0001"
      → AccountTransfer { fromAccount=SUB-001, toAccount=AGT-001 }

  3b: getSenderType(SUBSCRIBER_CASH_OUT) → "Subscriber"
      getReceiverType(SUBSCRIBER_CASH_OUT) → "Agent"

  3c: Loop through slabs [AGENT=60%, RESALE_AGENT=25%, ADMIN=15%]

  --- SLAB 1: AGENT, 60% ---
  role="AGENT", receiverTypeStr="Agent" → MATCH → targetWalletId = AGT-001
  SUBSCRIBER_CASH_OUT override: role="Agent" → targetWalletId = AGT-001 (same)
  payoutAmount = 60% of $2.00 = $1.20
  Transfer: AMT03 → AGT-001, $1.20, type=COMMISSION_DISBURSEMENT_TO_AGENT
  → gPayAccountTransferService.processTransaction(dto)
     → wallet[AMT03] -= $1.20
     → wallet[AGT-001] += $1.20
     → account_transfer row INSERTED + COMPLETED

  --- SLAB 2: RESALE_AGENT, 25% ---
  role="RESALE_AGENT" doesn't match "Subscriber" or "Agent" → targetWalletId = null
  SUBSCRIBER_CASH_OUT override: role=RESALE_AGENT →
    agentWallet = walletRepo.findByAggregateId("AGT-001")
    agentInfo = userService.getUserInfoFromId(agentWallet.userId)
    resaleAgentInfo = userService.getUserInfoFromId(agentInfo.parentAgentId) → RSA-002
    targetWalletId = walletRepo.findOneByUserIdAndType(RSA-002.userId, AGENT).id → "RSA-WALLET-001"
  payoutAmount = 25% of $2.00 = $0.50
  Transfer: AMT03 → RSA-WALLET-001, $0.50, type=COMMISSION_DISBURSEMENT_TO_RESALE_AGENT
  → gPayAccountTransferService.processTransaction(dto)
     → wallet[AMT03] -= $0.50
     → wallet[RSA-WALLET-001] += $0.50

  --- SLAB 3: ADMIN, 15% ---
  role=ADMIN → continue (stays in AMT03)

Step 4: commission.setPaidStatus(true) → commission_info[id=1].paid_status = true
```

**Final State:**
```
AMT03: was $2.00 → now $0.30 (the 15% ADMIN share stays)
AGT-001: +$1.20
RSA-WALLET-001: +$0.50
commission_info[id=1]: paid_status = true
```

---

### ❌ SUBSCRIBER_CASH_IN — Fails

**Scenario:** Agent deposits $100 into a subscriber's account. Agent earns commission.

#### Phase 1: Original Transaction

```
Agent Wallet (AGT-001) ──$100──► Subscriber Wallet (SUB-001)

commission_info row created:
  transfer_id = "SCI-20240515-0001"
  account_aggregate_id = "AMT03"
  amount = 1.50 (some commission)
  paid_status = false
  transfer_type = SUBSCRIBER_CASH_IN
```

#### Phase 2: processDisbursement(SUBSCRIBER_CASH_IN) — FAILS

```
Step 1: Find profile for SUBSCRIBER_CASH_IN → CDP-002 (if configured)

Step 3: processSingleCommission(...)

  3b: getSenderType(SUBSCRIBER_CASH_IN) → "" (not in switch → default)
      getReceiverType(SUBSCRIBER_CASH_IN) → "" (not in switch → default)

  --- SLAB: AGENT, 60% ---
  role="AGENT", senderTypeStr="" → "AGENT".equalsIgnoreCase("") → FALSE
  role="AGENT", receiverTypeStr="" → FALSE
  targetWalletId = null

  SUBSCRIBER_CASH_OUT override block:
    profile.getTransferType() == SUBSCRIBER_CASH_OUT? → NO (it's SUBSCRIBER_CASH_IN)
    → block is SKIPPED

  targetWalletId is still null → log.warn → continue
  → AGENT NEVER GETS COMMISSION ❌

```

**The Agent who performed this Cash In NEVER gets their commission paid out.**

The fix is simple: add `SUBSCRIBER_CASH_IN` to `getSenderType()` and `getReceiverType()` in `MfsUtils`, OR extend the hardcoded block. The correct logic is:
```
SUBSCRIBER_CASH_IN: Agent is the SENDER (fromAccountAggregateId)
targetWalletId = originalTransfer.getFromAccountAggregateId()
```

---

### ❌ RESALE_TO_AGENT_TRANSFER — Fails

**Scenario:** Resale Agent sends $500 to an Agent (liquidity transfer). Commission earned.

#### Original Transaction

```
Resale Agent Wallet (RSA-001) ──$500──► Agent Wallet (AGT-001)

commission_info:
  transfer_id = "RAA-20240515-0001"
  account_aggregate_id = "AMT03"
  amount = 5.00
  transfer_type = RESALE_TO_AGENT_TRANSFER
```

#### Disbursement — FAILS

```
getSenderType(RESALE_TO_AGENT_TRANSFER) → "" (not in switch)
getReceiverType(RESALE_TO_AGENT_TRANSFER) → "" (not in switch)

AGENT role: targetWalletId = null → skipped ❌
RESALE_AGENT role: targetWalletId = null → skipped ❌
override block not triggered (it checks == SUBSCRIBER_CASH_OUT)
```

**Correct logic should be:**
- Agent (receiver): `toAccountAggregateId` = AGT-001
- Resale Agent (sender): `fromAccountAggregateId` = RSA-001

---

### ❌ AGENT_TO_RESALE_AGENT_TRANSFER — Fails

**Scenario:** Agent sends $200 to their Resale Agent (returning liquidity).

#### Original Transaction

```
Agent Wallet (AGT-001) ──$200──► Resale Agent Wallet (RSA-001)

commission_info:
  transfer_id = "ARA-20240515-0001"
  account_aggregate_id = "AMT03"
  amount = 2.00
  transfer_type = AGENT_TO_RESALE_AGENT_TRANSFER
```

#### Disbursement — FAILS

```
getSenderType(AGENT_TO_RESALE_AGENT_TRANSFER) → ""
getReceiverType(AGENT_TO_RESALE_AGENT_TRANSFER) → ""

All roles → targetWalletId = null → ALL SKIPPED ❌
```

**Correct logic should be:**
- Agent (sender): `fromAccountAggregateId` = AGT-001
- Resale Agent (receiver): `toAccountAggregateId` = RSA-001

---

## 15. Code Quality Issues and Refactoring Plan

### 🚩 Issue 1: Massive Duplication — `TransferCommissionInterceptor` vs `ServiceAndCommissionChargeService`

**Problem:** Both files contain virtually identical code:
- `setServiceChargeAndCommission()` is copy-pasted
- `setCommission()` is copy-pasted
- `setServiceCharge()` is copy-pasted
- All the `AMAL_EXPRESS_*` handling logic is duplicated

`TransferCommissionInterceptor` appears to be the OLD Axon-based path; `ServiceAndCommissionChargeService` is the NEW path used by `GPayAccountTransferService`. The old one should be deleted once the new path is confirmed as canonical.

**Fix:**
```java
// Delete TransferCommissionInterceptor (or keep as no-op shell)
// ServiceAndCommissionChargeService is the single source of truth
```

---

### 🚩 Issue 2: Hardcoded Transfer Type Logic in processSingleCommission()

**Problem:** The only disbursement that works is `SUBSCRIBER_CASH_OUT` because it has a dedicated hardcoded `if` block. Every new transfer type requires writing another `else if` block.

```java
// Current ugly code:
if (profile.getTransferType() == TransferType.SUBSCRIBER_CASH_OUT) {
    if (role.name().equalsIgnoreCase("Agent")) { ... }
    else if (role == RESALE_AGENT) { ... }
    else if (role == DISTRIBUTOR_AGENT) { ... }
}
// What about SUBSCRIBER_CASH_IN? RESALE_TO_AGENT_TRANSFER? Nothing!
```

---

### 🚩 Issue 3: getSenderType()/getReceiverType() Return Empty Strings for Many Types

**Problem:** `MfsUtils.getSenderType()` and `getReceiverType()` don't have cases for `SUBSCRIBER_CASH_IN`, `RESALE_TO_AGENT_TRANSFER`, `AGENT_TO_RESALE_AGENT_TRANSFER`, etc. They return `""`, which breaks the generic mapping logic.

---

### 🚩 Issue 4: String Comparison with UserType.name() is Fragile

```java
// This is fragile:
role.name().equalsIgnoreCase(senderTypeStr)
// role.name() = "RESALE_AGENT"
// senderTypeStr = "Resale Agent" (with spaces)
// That's why there's this hack:
role.name().equalsIgnoreCase(senderTypeStr.replace(" ", "_"))
```

The code replaces spaces with underscores to try to make `"Resale Agent"` match `"RESALE_AGENT"`. This is brittle and confusing.

---

### 🚩 Issue 5: N+1 DB Queries Per Commission

For DISTRIBUTOR_AGENT resolution, the code makes 4 separate DB calls:
1. `walletQueryRepository.findByAggregateId()`
2. `userService.getUserInfoFromId()` (agent)
3. `userService.getUserInfoFromId()` (resale agent parent)
4. `userService.getUserInfoFromId()` (distributor agent grandparent)
5. `walletQueryRepository.findOneByUserIdAndType()`

For 1,000 commission records, that's 5,000 DB calls just for DISTRIBUTOR_AGENT. No batching, no caching.

---

### 🚩 Issue 6: `userService.getUserInfoFromId(1)` — Hardcoded Admin User

```java
UserInfo userInfoFromId = userService.getUserInfoFromId(1);
```

This hardcodes the admin user as user ID 1. If the system ever migrates, this breaks silently.

---

### ✅ Refactoring Approach: Strategy Pattern for Agent Identification

**Goal:** Every transfer type should have a clean strategy for resolving "which wallet gets the commission for role X?"

**Step 1: Create an interface**

```java
public interface AgentIdentificationStrategy {
    boolean supports(TransferType transferType);
    String resolveAgentWalletId(UserType role, AccountTransfer originalTransfer);
}
```

**Step 2: Implement per transfer type**

```java
@Component
public class SubscriberCashOutStrategy implements AgentIdentificationStrategy {
    
    @Override
    public boolean supports(TransferType type) {
        return type == TransferType.SUBSCRIBER_CASH_OUT;
    }
    
    @Override
    public String resolveAgentWalletId(UserType role, AccountTransfer transfer) {
        switch (role) {
            case AGENT:
                return transfer.getToAccountAggregateId(); // agent received the cash
            case RESALE_AGENT:
                return walkToParent(transfer.getToAccountAggregateId(), 1);
            case DISTRIBUTOR_AGENT:
                return walkToParent(transfer.getToAccountAggregateId(), 2);
            default:
                return null;
        }
    }
}

@Component
public class SubscriberCashInStrategy implements AgentIdentificationStrategy {
    
    @Override
    public boolean supports(TransferType type) {
        return type == TransferType.SUBSCRIBER_CASH_IN;
    }
    
    @Override
    public String resolveAgentWalletId(UserType role, AccountTransfer transfer) {
        switch (role) {
            case AGENT:
                return transfer.getFromAccountAggregateId(); // agent sent the cash
            case RESALE_AGENT:
                return walkToParent(transfer.getFromAccountAggregateId(), 1);
            case DISTRIBUTOR_AGENT:
                return walkToParent(transfer.getFromAccountAggregateId(), 2);
            default:
                return null;
        }
    }
}

@Component
public class ResaleToAgentTransferStrategy implements AgentIdentificationStrategy {
    
    @Override
    public boolean supports(TransferType type) {
        return type == TransferType.RESALE_TO_AGENT_TRANSFER;
    }
    
    @Override
    public String resolveAgentWalletId(UserType role, AccountTransfer transfer) {
        switch (role) {
            case AGENT:
                return transfer.getToAccountAggregateId();   // Agent = receiver
            case RESALE_AGENT:
                return transfer.getFromAccountAggregateId(); // Resale Agent = sender
            default:
                return null;
        }
    }
}
```

**Step 3: Extract the hierarchy walk into a shared utility**

```java
@Service
public class AgentHierarchyResolver {
    
    @Autowired private WalletQueryRepository walletRepo;
    @Autowired private UserService userService;
    
    /**
     * Given a starting wallet ID, walk up N levels in the parent agent chain.
     * level=0 → the wallet itself
     * level=1 → parent (Resale Agent)
     * level=2 → grandparent (Distributor Agent)
     */
    public String resolveParentAtLevel(String startWalletId, int level) {
        Wallet wallet = walletRepo.findByAggregateId(startWalletId);
        UserInfo current = userService.getUserInfoFromId(wallet.getUserId());
        
        for (int i = 0; i < level; i++) {
            if (current.getParentAgentId() == null) return null;
            current = userService.getUserInfoFromId(current.getParentAgentId());
        }
        
        return walletRepo.findOneByUserIdAndType(current.getUserId(), WalletType.AGENT).getId();
    }
}
```

**Step 4: Update CommissionDisbursementProcessingService**

```java
@Autowired
private List<AgentIdentificationStrategy> strategies;

private String resolveTargetWallet(UserType role, TransferType transferType, 
                                   AccountTransfer originalTransfer) {
    return strategies.stream()
        .filter(s -> s.supports(transferType))
        .findFirst()
        .map(s -> s.resolveAgentWalletId(role, originalTransfer))
        .orElseThrow(() -> new MFSException(
            "No agent identification strategy for " + transferType
        ));
}
```

**Step 5: Fix MfsUtils.getSenderType() and getReceiverType()**

Add the missing cases:
```java
case SUBSCRIBER_CASH_IN:
    return "Agent"; // agent IS the sender

case RESALE_TO_AGENT_TRANSFER:
    return "Resale Agent";

case AGENT_TO_RESALE_AGENT_TRANSFER:
    return "Agent";
```

---

### Summary: What to Fix in What Order

| Priority | Fix | Impact |
|----------|-----|--------|
| P0 (Critical) | Implement `SubscriberCashInStrategy` | Cash In commissions never paid |
| P0 (Critical) | Implement `ResaleToAgentTransferStrategy` | Agent-Resale transfers unhandled |
| P0 (Critical) | Implement `AgentToResaleAgentTransferStrategy` | Same |
| P1 (High) | Delete/deprecate `TransferCommissionInterceptor` | Eliminate duplication |
| P1 (High) | Extract `AgentHierarchyResolver` | Stop N+1 queries, reuse logic |
| P2 (Medium) | Fix `getSenderType()`/`getReceiverType()` for missing types | Generic mapping would work |
| P2 (Medium) | Replace string comparisons with enum comparisons | Type safety |
| P3 (Low) | Replace `getUserInfoFromId(1)` with a proper admin lookup | Fragility |
| P3 (Low) | Add batch processing for commissions | Performance |
