# Mini Project: Employee Payroll System

Imagine you're building:

```
Employee Payroll System
```

Features:

- Create Payroll
- Send SMS
- Generate Report
- Transfer Salary
- Handle Failures

We'll evolve it step by step.

---

# Phase 1 — Simple Service-Based Architecture

## Requirement V1

When payroll is processed:

1. Save payroll
2. Send SMS
3. Generate report

Everything inside one application.

---

## Implementation

```
@Servicepublic class PayrollService {    public void processPayroll() {        payrollRepository.save(...);        smsService.sendSalarySms();        reportService.generatePayrollReport();    }}
```

Flow:

```
PayrollService      |      +--> Save Payroll      |      +--> Send SMS      |      +--> Generate Report
```

---

## Problem Appears

Business adds:

```
Audit LogEmail NotificationSlack Notification
```

Now:

```
public void processPayroll() {    payrollRepository.save(...);    smsService.sendSms();    emailService.sendEmail();    auditService.saveAudit();    reportService.generateReport();    slackService.notify();}
```

---

Problems:

```
Huge serviceTight couplingHard to maintainAdding new feature requiresmodifying PayrollService
```

---

# Phase 2 — Spring In-Memory Events

## New Requirement

Business says:

> Payroll processing should not know about SMS, Email, Audit, Report.

Payroll only processes payroll.

---

## Solution

Use:

```
ApplicationEventPublisher
```

---

Payroll Service

```
payrollRepository.save(...);publisher.publishEvent(    new PayrollProcessedEvent(...));
```

---

SMS Listener

```
@EventListenerpublic void handle(        PayrollProcessedEvent event) {    smsService.sendSms();}
```

---

Report Listener

```
@EventListenerpublic void handle(        PayrollProcessedEvent event) {    reportService.generate();}
```

---

Flow

```
Payroll Processed        |        vPayrollProcessedEvent        |    -------------    |           |    v           v SMS       Report
```

---

## Why We Needed It

Before:

```
PayrollService    knows SMS    knows Report    knows Audit
```

After:

```
PayrollService    knows only Event
```

Loose coupling achieved.

---

# Phase 3 — Async Spring Events

## New Requirement

Business says:

> Payroll processing takes 200ms.
> 
> SMS takes 5 seconds.
> 
> Report takes 10 seconds.

Users complain:

```
Payroll API is slow.
```

---

Current flow:

```
Payroll   |   +--> SMS (5 sec)   |   +--> Report (10 sec)
```

Response waits.

---

## Solution

Make listeners async.

```
@Async@EventListenerpublic void handle(...) {}
```

---

Flow

```
Payroll Processed      |      v Event Published      |      +--> Thread-1 SMS      |      +--> Thread-2 Report      |      +--> Thread-3 Audit
```

API returns immediately.

---

## Why We Needed It

In-memory events solved coupling.

Async events solved performance.

---

# Phase 4 — RabbitMQ Event Driven

## New Requirement

Company creates separate teams.

Now:

```
Payroll TeamNotification TeamReporting Team
```

Management says:

```
Notification must be deployed independently.Reporting must be deployed independently.
```

---

Now architecture becomes:

```
payroll-servicenotification-servicereporting-service
```

Three applications.

---

Problem:

```
publisher.publishEvent(...)
```

works only inside same JVM.

---

Need cross-service communication.

---

## Solution

RabbitMQ

---

Flow

```
Payroll Service      |      vRabbitMQ      |   ----------   |        |   v        vNotificationReporting
```

---

Payroll Service

```
rabbitTemplate.convertAndSend(    "payroll.exchange",    "payroll.processed",    event);
```

---

Notification Service

```
@RabbitListenerpublic void consume(        PayrollProcessedEvent event) {    sendSms();}
```

---

Reporting Service

```
@RabbitListenerpublic void consume(        PayrollProcessedEvent event) {    updateReport();}
```

---

## Why We Needed It

Spring Events:

```
One Application
```

RabbitMQ:

```
Multiple Applications
```

---

# Phase 5 — RabbitMQ Still Works

New Requirement

Add:

```
Audit ServiceAnalytics ServiceEmail Service
```

No payroll changes.

Just add new consumers.

---

Flow

```
Payroll   | RabbitMQ   |----------------------|    |    |    |     |SMS Report Audit Mail Analytics
```

This is pure Event Driven Architecture.

---

# Phase 6 — Saga Appears

Now new requirement changes everything.

---

## Requirement

When paying salary:

1. Create Payroll
2. Debit Company Wallet
3. Transfer Money
4. Mark Payroll Paid

---

This is NOT:

```
NotificationReportAudit
```

These are business steps.

---

Current flow:

```
Payroll Created      |      +--> Notification      +--> Report
```

Fine.

---

But now:

```
Create Payroll      |      vDebit Wallet      |      vTransfer Money      |      vMark Payroll Paid
```

---

What if:

```
Payroll Created ✅Wallet Debited ✅Transfer Failed ❌
```

Money already deducted.

Bad state.

---

# Need Saga

---

Step 1

Payroll Service

```
PayrollPaymentInitiatedEvent
```

---

Step 2

Wallet Service

Consumes:

```
PayrollPaymentInitiatedEvent
```

Debits wallet.

Publishes:

```
WalletDebitedEvent
```

---

Step 3

Bank Service

Consumes:

```
WalletDebitedEvent
```

Transfers salary.

Publishes:

```
SalaryTransferredEvent
```

---

Step 4

Payroll Service

Consumes:

```
SalaryTransferredEvent
```

Marks payroll paid.

---

Happy Path

```
Payroll   |   vWallet   |   vBank   |   vPayroll
```

---

# Failure Path

Bank transfer fails.

Publishes:

```
SalaryTransferFailedEvent
```

---

Wallet Service consumes.

```
walletService.creditBack();
```

Publishes:

```
WalletRefundedEvent
```

---

Payroll Service:

```
markFailed();
```

---

Flow

```
Payroll Started      |      vWallet Debited      |      XBank Failed      |      vRefund Wallet      |      vPayroll Failed
```

---

# What Each Step Teaches

|Phase|Architecture|Why Needed|
|---|---|---|
|1|Service Calls|Simple application|
|2|Spring Events|Remove coupling|
|3|Async Events|Improve performance|
|4|RabbitMQ|Communicate between microservices|
|5|More RabbitMQ Consumers|Easily add new services|
|6|Saga|Maintain consistency across multiple business services|

The key realization is:

```
NotificationReportingAuditAnalytics
```

are usually **event-driven side effects**.

But:

```
Debit WalletTransfer MoneyMark Payroll PaidRefund Wallet
```

are **business transaction steps**.

When multiple business steps must either complete together or be compensated on failure, you move from simple EDA to a Saga pattern. That's usually the moment developers truly understand why Saga exists.