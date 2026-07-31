# Audit Flow Documentation

## Service
**mfs-reporting**

**Package**
`com.vivacom.mfs.reporting`

---

# Overview

The **mfs-reporting** service is the reporting/read-side microservice in the CQRS + Event Sourcing architecture.

It does **not** modify business data.

Its responsibilities are:

- Listen to domain events published by other services.
- Consume events from RabbitMQ.
- Generate audit records.
- Generate reporting data.
- Store audit and reporting information in its own database.

The audit subsystem records:

- Who performed an action
- Which user/entity was affected
- What action occurred
- When it happened

Each audit entry is stored as a **UserAuditReport**.

---

# High-Level Architecture

```
Command Side Services
(User, Wallet, Admin, Agent, Merchant...)

            │
            │ Domain Events
            ▼
      Axon Framework
            │
            ▼
        RabbitMQ
            │
            ▼
    RabbitMQConsumer
            │
            ▼
     RabbitMQService
            │
            ▼
 Domain Audit Services
            │
            ▼
      AuditService
            │
            ▼
UserAuditReportingRepository
            │
            ▼
    user_audit_report
```

---

# Audit Flow

```
Domain Event
      │
      ▼
RabbitMQConsumer
      │
      ▼
RabbitMQService
      │
      ▼
Specific Audit Service
      │
      ▼
AuditService.saveData()
      │
      ▼
UserAuditReportingRepository
      │
      ▼
Database
```

---

# Event Ingestion

## RabbitMQConfig

Creates RabbitMQ infrastructure.

Beans:

| Bean | Purpose |
|------|----------|
| DirectExchange | Exchange |
| Queue | Event queue |
| Binding | Connects exchange to queue |

Configuration comes from:

```
rabbit.mq.exchange
rabbit.mq.queue.name
rabbit.mq.routing.key
```

---

## RabbitMQConsumer

Main entry point for every incoming event.

```java
@RabbitListener(queues = "${rabbit.mq.queue.name}")
```

Method:

```java
rabbitConsumer(Map<?, ?> message)
```

The consumer determines event type using message keys.

### User/Admin Events

```
identifier
        │
        ▼
checkEvent()
```

### Transaction Events

```
transferAggregateId
        │
        ▼
checkTransferEvent()
```

### Wallet History Events

```
transferId
      │
      ▼
checkUpdateWalletHistoryEvent()
```

---

# RabbitMQService

Location

```
com.vivacom.mfs.reporting.audit.RabbitMQService
```

Acts as the central dispatcher.

Annotated with:

```java
@Service
@Async
```

Responsibilities:

- Identify event type
- Deserialize payload
- Route event to appropriate audit service

---

# Event Routing

RabbitMQService inspects:

```
payloadType
```

Example:

```
SubscriberCreatedEvent
```

↓

```
newSubscriberAuditReportService
```

Example:

```
AdminCreatedEvent
```

↓

```
newAdminAuditReportService
```

Example:

```
MerchantCreatedEvent
```

↓

```
newMerchantAuditReportService
```

---

# Domain Audit Services

Each domain has its own audit service.

Examples:

| Service | Responsibility |
|----------|----------------|
| NewSubscriberAuditReportService | Subscriber events |
| NewAdminAuditReportService | Admin events |
| NewAgentAuditReportService | Agent events |
| NewMerchantAuditReportService | Merchant events |
| NewGeographyAndGradeAuditService | Geography & Grade |
| NewPermissionAuditService | Permissions |
| NewThresholdAuditService | Thresholds |
| NewServiceChargeProfileAuditService | Service Charge Profiles |
| NewTransactionAuditReportService | Transaction Audit |

---

# Responsibility of Audit Services

Each audit service performs three tasks:

1. Create a readable description.
2. Choose an AuditActionType.
3. Save audit through AuditService.

Example:

```java
public void subscriberCreatedEvent(UserDto userDto) {

    String description =
        "Subscriber Created. Id: "
        + userDto.getId();

    AuditActionType action =
        AuditActionType.SUBSCRIBER_CREATION;

    auditService.saveData(
        userDto.getSubmittedById(),
        userDto.getId(),
        description,
        action
    );
}
```

Notice:

- `submittedById` → actor
- `userId` → affected user

Audit services never access repositories directly.

---

# AuditService

Location

```
com.vivacom.mfs.reporting.audit.AuditService
```

This is the central persistence layer.

It exposes three overloaded methods.

```java
saveData(
    adminId,
    description,
    actionType
)
```

For system/admin actions.

---

```java
saveData(
    adminId,
    userId,
    description,
    actionType
)
```

For user-related actions.

---

```java
saveData(
    adminId,
    userId,
    description,
    reason,
    actionType
)
```

Stores update reason as well.

---

Every method:

- Creates UserAuditReport
- Sets metadata
- Saves through repository

---

# Database Entity

Table:

```
user_audit_report
```

Columns:

| Column | Description |
|----------|------------|
| id | Primary Key |
| adminId | Action performer |
| userId | Target user |
| eventDescription | Human-readable message |
| updateReason | Optional reason |
| createdAt | Timestamp |
| actionType | AuditActionType |

---

# Repository

```java
UserAuditReportingRepository
```

Extends

```java
PagingAndSortingRepository<
    UserAuditReport,
    Integer
>
```

Repository only performs persistence.

Filtering logic exists inside AuditService.

---

# Query API

AuditService exposes:

```java
getFilteredAuditReport()
```

Supports filtering by:

- adminId
- userId
- description
- actionType
- dateFrom
- dateTo
- msisdn

If MSISDN is provided:

```
MSISDN
    │
    ▼
UserQueryRepository
    │
    ▼
Resolve userId
```

Results are paginated and sorted by newest first.

---

# AuditActionType

Audit actions are categorized using the enum:

```
AuditActionType
```

Major groups include:

## Admin

- CREATION
- APPROVAL
- REJECTION
- UPDATED
- BLOCKED
- UNBLOCKED
- PASSWORD_UPDATED
- PINCODE_CHANGED

---

## Subscriber

- SUBSCRIBER_CREATION
- SUBSCRIBER_APPROVED
- SUBSCRIBER_REJECTED
- SUBSCRIBER_UPDATED
- SUBSCRIBER_UPDATE_REJECTED

---

## Agent

- AGENT_CREATED
- AGENT_APPROVED
- AGENT_UPDATED
- AGENT_REJECTED

---

## Merchant

- MERCHANT_CREATED
- MERCHANT_APPROVED
- MERCHANT_UPDATED
- MERCHANT_REJECTED

---

## Geography

- AREA_CREATED
- ZONE_CREATED
- DOMAIN_CREATED

---

## Grade

- GRADE_CREATED
- GRADE_UPDATED

---

## Threshold

- THRESHOLD_CREATED
- BALANCE_THRESHOLD_CREATED

---

## Transaction

- ACCOUNT_TRANSFER_CREATE

---

# Registration Reporting

Besides audit records, registration events generate reporting data.

Handled by:

```
NewUserReportingEventService
```

Creates:

```
CustomerRegistrationReport
```

Stores:

- User information
- Registration time
- Approval time
- Area
- Zone
- Domain
- Registered By

Duplicate prevention:

```
findByUserAggregateId()
```

If record already exists, it skips insertion.

---

# Transaction Reporting

Transaction events follow another path.

```
RabbitMQConsumer
        │
        ▼
checkTransferEvent()
        │
        ▼
ReportProcessingService
        │
        ▼
CustomerTransactionReport
```

Stores:

- Payer
- Payee
- Commission
- Balance snapshots
- Transaction report

Additionally,

```
NewTransactionAuditReportService
```

creates an audit entry with:

```
ACCOUNT_TRANSFER_CREATE
```

---

# Deprecated Components

Earlier versions relied on:

- Axon Event Handlers
- Kafka Listeners

These classes still exist but are inactive.

Examples:

- AdminAuditReportHandler
- SubscriberAuditReportHandler
- MerchantAuditReportHandler
- AgentAuditReportHandler
- KafkaTopicListener

All `@EventHandler` annotations are commented out.

Current implementation exclusively uses:

```
RabbitMQ
```

---

# End-to-End Example

## Subscriber Created

```
Admin
    │
    ▼
POST /register-subscriber
    │
    ▼
CreateSubscriberCommand
    │
    ▼
Subscriber Aggregate
    │
    ▼
SubscriberCreatedEvent
    │
    ▼
RabbitMQ
    │
    ▼
RabbitMQConsumer
    │
    ▼
RabbitMQService
    │
    ▼
NewSubscriberAuditReportService
    │
    ▼
AuditService.saveData()
    │
    ▼
UserAuditReportingRepository
    │
    ▼
user_audit_report
```

Later, after approval:

```
SubscriberCreationSuccessEvent
        │
        ▼
NewUserReportingEventService
        │
        ▼
CustomerRegistrationReport
```

---

# Important Files

| File | Responsibility |
|------|----------------|
| RabbitMQConfig.java | RabbitMQ configuration |
| RabbitMQConsumer.java | Entry point for events |
| RabbitMQService.java | Event router |
| AuditService.java | Audit persistence |
| UserAuditReport.java | Audit entity |
| AuditActionType.java | Audit categories |
| UserAuditReportingRepository.java | Repository |
| AuditReportRequestDto.java | Audit query DTO |
| NewSubscriberAuditReportService.java | Subscriber audit |
| NewAdminAuditReportService.java | Admin audit |
| NewAgentAuditReportService.java | Agent audit |
| NewMerchantAuditReportService.java | Merchant audit |
| NewGeographyAndGradeAuditService.java | Geography & Grade audit |
| NewUserReportingEventService.java | Registration reporting |
| ReportProcessingService.java | Transaction reporting |

---

# Summary

1. Domain services publish events.
2. Events reach RabbitMQ.
3. RabbitMQConsumer receives them.
4. RabbitMQService identifies event type.
5. Corresponding audit service prepares audit information.
6. AuditService persists UserAuditReport.
7. Reporting services create registration and transaction reports when applicable.