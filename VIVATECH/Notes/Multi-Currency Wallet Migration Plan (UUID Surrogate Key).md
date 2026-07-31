# Multi-Currency Wallet Migration Plan (UUID Surrogate Key)

## Objective

Implement a multi-currency wallet architecture where `id` acts as a purely opaque surrogate key (UUID) while preserving the stable business account number (`aggregateId`). Ensure backward compatibility and zero disruption to downstream reporting or existing UI.

## 1. Database Schema Constraints

We will rely on the database to enforce business rules and eliminate race conditions, avoiding application-level locks where possible.

- **`Wallet` Entity Changes:** No column changes are needed. Existing rows stay exactly as they are (`id` = numeric, `aggregateId` = numeric).
- **Unique Constraint:** Add a JPA unique constraint on the `Wallet` entity:
    
    java
    
    @Table(uniqueConstraints = @UniqueConstraint(columnNames = {"aggregateId", "currency", "type"}))
    
    _This guarantees a user (e.g., `100000`) can only have exactly one `CUSTOMER_CARE` wallet in `SOS`._

## 2. Resolving Data Contract Leakage (`TransferController`)

As discussed, we must prevent the internal UUID from leaking into customer receipts or reporting tools as the "Account Number".

**Action:** Update all DTO builders in `TransferController.java` (and similar controllers) to decouple the keys:

java

WalletInfo senderWalletInfo = walletService.getWalletOfUserAndType(u.getAggregateId(), WalletType.SUBSCRIBER);

TransferEventDto.builder()

    // The business number (e.g., '100000'). Safe for UI, receipts, and reporting.

    .fromAccountAggregateId(senderWalletInfo.getAggregateId()) 

    // The UUID (e.g., 'f47ac10b...'). Kept internal for Axon Aggregate routing.

    .fromAccountId(senderWalletInfo.getWalletId())

_(This must be applied to `toAccountAggregateId` / `toAccountId` as well.)_

## 3. Wallet Resolution Strategy

All internal lookups must explicitly request the currency. Relying on just `aggregateId` is deprecated.

**Action:** Ensure `WalletQueryRepository` uses:

java

Wallet findByAggregateIdAndCurrencyAndType(String aggregateId, String currency, WalletType type);

## 4. API: Create Currency Wallet (`WalletController`)

We will implement a new endpoint allowing admins/users to provision a new currency wallet.

**Endpoint:** `POST /account/create-subscriber-wallet` (or customer care equivalent) **Payload:** `{ "userId": 123, "currency": "SOS" }`

**Execution Flow:**

1. Fetch the user's base information to retrieve their `aggregateId` (e.g., `100000`).
2. Generate a new surrogate key: `String walletId = UUID.randomUUID().toString();`
3. Dispatch the Axon command using the UUID as the `@TargetAggregateIdentifier`:
    
    java
    
    CreateSubscriberWalletCommand command = new CreateSubscriberWalletCommand(
    
        walletId,                 // Axon ID (UUID)
    
        user.getFullName(), 
    
        user.getId(),
    
        user.getAggregateId(),    // Payload carries business number
    
        "SOS",                    // Currency
    
        // ...
    
    );
    
4. **Safety Net:** Catch `DataIntegrityViolationException` (triggered by the new `@UniqueConstraint`) to silently handle concurrent double-creation attempts.

## Future Considerations

While this plan safely implements multi-currency within the current schema, the `aggregateId` field is effectively functioning as a virtual `Account` parent. Future architectural phases should formally introduce an `Account` entity that `Wallet` entities foreign-key to, removing the reliance on string-matching aggregate IDs.

Implementation Plan Create Subscriber Wallet

# Implementation Plan: Create Subscriber Wallet API

## Objective

Implement a new API endpoint (`POST /account/create-subscriber-wallet`) allowing the addition of a secondary currency wallet for an existing subscriber. This API will implement the agreed-upon UUID surrogate key strategy.

## Proposed Changes

### 1. Database Constraint (Wallet Entity)

To enforce business rules and prevent race conditions safely, we will add the unique constraint to the query-model entity.

#### [MODIFY] `wallet-query/.../entity/Wallet.java`

- Add the composite unique constraint to the class-level annotations:
    
    java
    
    @Table(indexes = { ... }, 
    
           uniqueConstraints = @UniqueConstraint(columnNames = {"aggregateId", "currency", "type"}))
    

### 2. Axon Commands & Events (Wallet.kt)

Because `walletId` (UUID) and the business account number (`aggregateId`) are now decoupled, the event must carry both to properly construct the read-model.

#### [MODIFY] `core-api-kotlin/.../Wallet.kt`

- Update `CreateSubscriberWalletCommand` to accept the `accountAggregateId`:
    
    kotlin
    
    class CreateSubscriberWalletCommand(
    
        val walletId: String,
    
        // ... existing fields ...
    
        val accountAggregateId: String? = null
    
    ) : BaseCommand()
    
- Update `SubscriberWalletCreateEvent` to carry the `accountAggregateId`:
    
    kotlin
    
    class SubscriberWalletCreateEvent(
    
        override val walletId: String,
    
        // ... existing fields ...
    
        val accountAggregateId: String? = null
    
    ) : BaseWalletCreateEvent(...)
    

### 3. Aggregate Event Handling

#### [MODIFY] `wallet/.../aggregate/Wallet.java`

- In `handle(CreateSubscriberWalletCommand command)`, pass `command.getAccountAggregateId()` into the `SubscriberWalletCreateEvent` constructor.

### 4. Query-Side Event Listener

When the Axon event is projected into the database, it must explicitly set the `aggregateId` to the business number if it differs from the `walletId`.

#### [MODIFY] `wallet-query/.../listener/WalletListener.java`

- In `handleSubscriberAccountCreated(SubscriberWalletCreateEvent event)`:
    
    java
    
    Wallet a = new Wallet(event.getWalletId());
    
    if (StringUtils.hasText(event.getAccountAggregateId())) {
    
        a.setAggregateId(event.getAccountAggregateId());
    
    }
    
    // ... existing setup ...
    

### 5. The API Controller

#### [NEW] `CreateSubscriberWalletRequestDto.java`

- Simple DTO holding `userId` (ULID) and `currencyCode`.

#### [MODIFY] `application/.../controller/WalletController.java`

- Add `POST /account/create-subscriber-wallet`.
- **Logic:**
    1. Fetch the `Subscriber` entity using the provided `userId`.
    2. The subscriber's business account number is `subscriber.getAggregateId()` (e.g., `55000000`).
    3. Generate the surrogate key: `String newWalletId = UUID.randomUUID().toString();`
    4. Dispatch the `CreateSubscriberWalletCommand` using the UUID as `walletId` and the `55000000` as the `accountAggregateId`.
    5. Handle `DataIntegrityViolationException` in case a wallet with this currency already exists.

## User Review Required

Please review the scope of this plan. It focuses exclusively on provisioning the wallet safely in the new UUID format. Account transfer integration is explicitly out of scope for this step.