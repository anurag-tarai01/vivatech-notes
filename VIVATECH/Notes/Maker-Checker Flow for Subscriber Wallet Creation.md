# Maker-Checker Flow for Subscriber Wallet Creation

This document outlines the plan to introduce a maker-checker flow to the subscriber wallet creation process and integrate it with the `mfs-reporting` audit subsystem.

## Open Questions

IMPORTANT

1. Should the `RequestCreateSubscriberWalletCommand` be handled by a new `PendingWallet` aggregate, or should we create the `Wallet` aggregate immediately but with a `PENDING` status? (The plan assumes creating the `Wallet` aggregate with `PENDING` status for simplicity and tracking).
2. Are there any existing notification mechanisms (e.g., sending SMS to the subscriber) that need to be deferred from the initial request to the approval phase?
3. Does the maker-checker flow apply only to `SUBSCRIBER` wallets, or should we design the events to be generic for all wallet types?

## Proposed Changes

We will implement this by separating the current direct wallet creation into a three-step command-event cycle (Request -> Approve/Reject) and extending the reporting service to consume these events.

---

### 1. Define New Commands and Events

We will define new commands and events in the Kotlin core-api module to represent the lifecycle of a wallet creation request.

#### [MODIFY] Wallet.kt

Add new commands:

- `RequestCreateSubscriberWalletCommand`
- `ApproveSubscriberWalletCommand`
- `RejectSubscriberWalletCommand`

Add new events:

- `SubscriberWalletCreateRequestedEvent`
- `SubscriberWalletApprovedEvent`
- `SubscriberWalletRejectedEvent`

Ensure `WalletStatus.PENDING` is utilized (it already exists in the enum).

---

### 2. Update the Wallet Aggregate

The Wallet Aggregate must be updated to handle the new lifecycle.

#### [MODIFY] Wallet.java

- Add `@CommandHandler` for `RequestCreateSubscriberWalletCommand` (constructor). It will validate and apply `SubscriberWalletCreateRequestedEvent`.
- Add `@EventSourcingHandler` for `SubscriberWalletCreateRequestedEvent` to set the aggregate status to `PENDING`.
- Add `@CommandHandler` for `ApproveSubscriberWalletCommand`. It will check if the wallet is `PENDING` and apply `SubscriberWalletApprovedEvent` (and potentially the standard `SubscriberWalletCreateEvent` if needed for backward compatibility with other listeners).
- Add `@CommandHandler` for `RejectSubscriberWalletCommand`. It will check if the wallet is `PENDING` and apply `SubscriberWalletRejectedEvent`.

---

### 3. Controller and Read-Model Updates

We need new endpoints in the controller and logic to handle the read-model updates (saving as pending, deleting on rejection).

#### [MODIFY] WalletController.java

- Update `/account/create-subscriber-wallet` to dispatch `RequestCreateSubscriberWalletCommand` instead of `CreateSubscriberWalletCommand`.
- Add endpoint `POST /account/approve-subscriber-wallet` to dispatch `ApproveSubscriberWalletCommand`.
- Add endpoint `POST /account/reject-subscriber-wallet` to dispatch `RejectSubscriberWalletCommand`.
- Add endpoint `GET /account/pending-subscriber-wallets` to fetch wallets where `status == WalletStatus.PENDING`.

#### [MODIFY] WalletListener.java

- Handle `SubscriberWalletCreateRequestedEvent` to save the wallet entity with `PENDING` status.
- Handle `SubscriberWalletApprovedEvent` to update the wallet status to `ACTIVE`.
- Handle `SubscriberWalletRejectedEvent` to **delete** the pending entry from the database.

---

### 4. Audit Flow Integration (mfs-reporting)

Integrate the new events into the reporting module to generate audit logs.

#### [MODIFY] AuditActionType.java (or `User.kt` depending on exact location)

- Add enum values: `WALLET_CREATION_REQUESTED`, `WALLET_APPROVED`, `WALLET_REJECTED`.

#### [NEW] `NewWalletAuditReportService.java`

Create a new service in `com.vivacom.mfs.reporting.audit` package.

- Inject `AuditService`.
- Implement methods to handle the three new wallet events.
- Each method will prepare a description, assign the correct `AuditActionType`, and call `auditService.saveData(...)`.

#### [MODIFY] RabbitMQService.java

- Add routing logic in the event dispatcher (e.g., checking `payloadType`).
- Route `SubscriberWalletCreateRequestedEvent`, `SubscriberWalletApprovedEvent`, and `SubscriberWalletRejectedEvent` to `NewWalletAuditReportService`.

## Verification Plan

### Automated Tests

- If unit tests exist for Wallet Aggregate, we will add test cases simulating the Request -> Approve and Request -> Reject lifecycles.

### Manual Verification

1. Call `/account/create-subscriber-wallet`. Verify the API returns a success response for request creation and a pending wallet appears in `/account/pending-subscriber-wallets`.
2. Check the `user_audit_report` table to verify a `WALLET_CREATION_REQUESTED` audit log was created.
3. Call `/account/reject-subscriber-wallet`. Verify the wallet disappears from the pending list (deleted from DB) and a `WALLET_REJECTED` audit log is created.
4. Repeat step 1, then call `/account/approve-subscriber-wallet`. Verify the wallet status becomes `ACTIVE` and a `WALLET_APPROVED` audit log is created.