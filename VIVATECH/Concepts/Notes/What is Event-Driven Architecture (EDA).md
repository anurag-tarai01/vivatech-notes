
**Event-Driven Architecture** is a design pattern where one component performs an action and publishes an **event**, while other components react to that event independently.

The publisher **does not know who will consume the event**.

### Traditional Direct Call

```java
userService.createUser(userDto);
walletService.createWallet(userId);
notificationService.sendSms(userId);
auditService.saveLog(userId);
```

Problems:

- Tight coupling
- If SMS fails, user creation may fail
- Hard to add new functionality

---

## Event-Driven Approach

### Step 1: User Created

```java
User user = userRepository.save(userDto);
eventPublisher.publishEvent(new UserCreatedEvent(user.getId()));
```

The publisher only publishes:

```java
UserCreatedEvent
```

It doesn't care who listens.

---

### Step 2: Wallet Service Listens

```java
@Componentpublic class WalletCreationListener {
  @EventListener public void handle(UserCreatedEvent event) {
    walletService.createWallet(event.getUserId());
  }
}
```

---

### Step 3: SMS Service Listens

```java
@Componentpublic class NotificationListener {
  @EventListener public void handle(UserCreatedEvent event) {
    smsService.sendWelcomeSms(event.getUserId());
  }
}
```

---

### Step 4: Audit Service Listens

```java
@Componentpublic class AuditListener {
  @EventListener public void handle(UserCreatedEvent event) {
    auditService.log("User Created : " + event.getUserId());
  }
}
```

---

## Flow

```bash
User Registration
        |
        v
UserCreatedEvent
        |
        +----> Wallet Listener
        |
        +----> SMS Listener
        |
        +----> Audit Listener
```

The creator doesn't know:

- Wallet exists
- SMS exists
- Audit exists

It only emits an event.

---

# Real Example from Your Backend

Suppose in `TransferController`:

```java
@PostMapping("/transfer")
public ResponseEntity<?> transfer(
        @RequestBody TransferRequest request) {

    Transfer transfer =
        transferService.transfer(request);

    eventPublisher.publishEvent(
        new TransferCompletedEvent(
            transfer.getId()
        )
    );

    return ResponseEntity.ok().build();
}
```

---

### Listener 1 - SMS

```java
@Component
public class TransferSmsListener {

    @EventListener
    public void handle(
        TransferCompletedEvent event) {

        smsService.sendTransferSuccessSms(
            event.getTransferId()
        );
    }
}
```

---

### Listener 2 - Report

```java
@Component
public class TransferReportListener {

    @EventListener
    public void handle(
        TransferCompletedEvent event) {

        reportService.updateReport(
            event.getTransferId()
        );
    }
}
```

---

### Listener 3 - Audit

```java
@Component
public class TransferAuditListener {

    @EventListener
    public void handle(
        TransferCompletedEvent event) {

        auditService.saveTransferLog(
            event.getTransferId()
        );
    }
}
```

---

# Event Object Example

```java
@Getter
@AllArgsConstructor
public class TransferCompletedEvent {

    private Long transferId;
}
```

---

# Async Event Driven

Without async:

```bash
Transfer
   |
   +--> SMS
   +--> Report
   +--> Audit
```

Controller waits for everything.

With async:

```java
@Async
@EventListener
public void handle(
        TransferCompletedEvent event) {

    smsService.sendSms(...);
}
```

Now:

```bash
Transfer Success
      |
      +--> Publish Event
              |
              +--> SMS Thread
              +--> Report Thread
              +--> Audit Thread
```

Response returns immediately.

---

# Event Driven Using Kafka/RabbitMQ

In large systems, events are often sent to a message broker.

Publisher:

```java
kafkaTemplate.send(
    "transfer-topic",
    transferEvent
);
```

Consumer:

```java
@KafkaListener(
    topics = "transfer-topic"
)
public void consume(
        TransferCompletedEvent event) {

    smsService.sendSms(...);
}
```

Flow:

```bash
Transfer Service
        |
        v
    Kafka Topic
        |
        +--> SMS Service
        |
        +--> Report Service
        |
        +--> Analytics Service
```

---

# Event Driven vs Normal Method Call

### Direct Call

```java
transferService.transfer();

smsService.sendSms();

auditService.saveLog();
```

```bash
Transfer Service
      |
      +--> SMS Service
      |
      +--> Audit Service
```

Tightly coupled.

---

### Event Driven

```java
transferService.transfer();

publishEvent(
    new TransferCompletedEvent()
);
```

```bash
Transfer Service
      |
      +--> Event
              |
              +--> SMS
              |
              +--> Audit
              |
              +--> Report
```

Loosely coupled.

---

## One-Line Definition

> Event-driven architecture is a pattern where a component publishes an event when something happens, and one or more independent consumers react to that event without the publisher knowing about them.