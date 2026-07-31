A **Saga** is a pattern used in distributed systems and microservices to manage a **business transaction that spans multiple services**, without using a single database transaction.

Instead of one large transaction:

```sql
BEGIN TRANSACTION
    Update A
    Update B
    Update C
COMMIT
```

a Saga breaks the process into a series of **local transactions**. Each service commits its own data and then triggers the next step. If something fails, **compensating actions** are executed to undo previous steps.

---

## Example: Money Transfer

Suppose transferring money involves three services:

1. Account Service
2. Ledger Service
3. Notification Service

### Traditional Transaction

```txt
Start Transaction
    Debit Sender
    Credit Receiver
    Create Ledger Entry
Commit
```

Works only when everything is in one database.

---

## Saga Approach

### Step 1

Account Service

```
Debit Sender
```

Publishes:

```
MoneyDebitedEvent
```

---

### Step 2

Ledger Service consumes event

```
Create Ledger Entry
```

Publishes:

```
LedgerCreatedEvent
```

---

### Step 3

Notification Service consumes event

```
Send SMS
```

Publishes:

```
NotificationSentEvent
```

Saga completed.

```
Debit → Ledger → Notification
```

---

## What if Step 2 Fails?

Suppose:

```
Debit Sender ✅Create Ledger ❌
```

Now sender's money is already debited.

Saga executes a **compensation transaction**:

```
Credit Sender Back
```

Result:

```
Debit Sender
    ↓
Ledger Failed
    ↓
Refund Sender
```

System returns to a consistent state.

---

# Two Types of Saga

## 1. Choreography Saga (Event Driven)

Services communicate only through events.

```
Account Service
      |
      v
MoneyDebitedEvent
      |
      v
Ledger Service
      |
      v
LedgerCreatedEvent
      |
      v
Notification Service
```

No central coordinator.

### Pros

- Loosely coupled
- Easy to add new listeners

### Cons

- Hard to understand end-to-end flow
- Event chains become complex

This is what many Kafka/RabbitMQ-based systems use.

---

## 2. Orchestration Saga

A central Saga Coordinator controls everything.

```
Saga Coordinator
      |
      +--> Debit Account
      |
      +--> Create Ledger
      |
      +--> Send Notification
```

If a step fails:

```
Saga Coordinator
      |
      +--> Compensation
```

### Pros

- Easier to track
- Easier debugging

### Cons

- Central component becomes important

---

# Saga vs Event Driven

People often confuse these.

### Event Driven

Simply means:

```
Something Happens
    ↓
Publish Event
    ↓
Other Service Reacts
```

Example:

```
UserRegisteredEvent
```

Email service sends welcome email.

No rollback logic.

---

### Saga

Saga is a **business transaction pattern** that often uses events.

```
Transfer Started
    ↓
Debit Account
    ↓
Credit Account
    ↓
Update Ledger
```

and includes:

```
Compensation Logic
```

when something fails.

So:

```
Event Driven != Saga
```

But:

```
Saga often uses Event Driven communication.
```

---

# In Your Backend Context

From your earlier descriptions:

### User Registration

You mentioned:

> both event-driven and saga are used

Likely flow:

```
Create User
    ↓
Create Wallet
    ↓
Create KYC Record
    ↓
Send Notification
```

These are multiple business steps.

If one fails:

```
Rollback WalletRollback User
```

or execute compensation.

That becomes a Saga.

---

### Account Transfer

You mentioned:

> account transfer only event driven

A typical flow may be:

```
TransferController
    ↓
GPayAccountTransferService
    ↓
Transfer Success
    ↓
Publish TransferCompletedEvent
```

Listeners:

```
Update ReportSend SMSGenerate Audit Log
```

These are independent reactions.

No compensation workflow across services.

Therefore:

```
Account Transfer = Event DrivenUser 
Registration = Saga
```

(if your architecture follows the pattern you described).

---

## Simple Interview Definition

> A Saga is a distributed transaction pattern where a business process is split into multiple local transactions. Each step commits independently, and if a later step fails, compensating transactions are executed to maintain consistency. Sagas can be implemented using choreography (events) or orchestration (coordinator).