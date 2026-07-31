# Existing RabbitMQ Event-Driven Flow

Currently:
```
Backend Service
      |
      | PayrollProcessedEvent
      v
RabbitMQ Exchange
      |
      +--> Notification Service
      |
      +--> Reporting Service
      |
      +--> Audit Service
```

All consumers are **side effects**.

If:
```
Notification Service FAILS
```

Payroll is still processed successfully.

No rollback.

This is pure Event-Driven Architecture.

---

# Business Requirement Change

Now suppose payroll processing involves multiple business services:

### Step 1

Payroll Service

```
Create Payroll Record
```

### Step 2

Wallet Service

```
Debit Company Wallet
```

### Step 3

Bank Service

```
Transfer Salary
```

### Step 4

Payroll Service

```
Mark Payroll PAID
```

### Step 5

Notification Service

```
Send SMS
```

Now these are not merely side effects.

This is one business transaction:

```
Payroll Payment
```

If Bank Transfer fails:

```
Payroll Created ✅Wallet Debited ✅Bank Transfer ❌
```

you cannot leave the wallet debited.

You need compensation.

This becomes a Saga.

---

# Saga Using RabbitMQ (Choreography)

No central coordinator.

Services communicate through events.

---

## Step 1

Payroll Service receives API request.

```java
@PostMapping("/payroll/pay")
public void payPayroll(Long payrollId) {

    payrollRepository.save(
        PayrollStatus.PROCESSING
    );

    publish(
        PayrollPaymentInitiatedEvent
    );
}
```

---

Event:

```java
public class PayrollPaymentInitiatedEvent {

    private Long payrollId;

    private Long companyId;

    private BigDecimal amount;
}
```

---

Flow:

```
Payroll Service
       |
       v
PayrollPaymentInitiatedEvent
```

---

## Step 2

Wallet Service listens.

```java
@RabbitListener(...)
public void consume(
        PayrollPaymentInitiatedEvent event) {

    walletService.debit(
            event.getCompanyId(),
            event.getAmount()
    );

    publish(
        CompanyWalletDebitedEvent
    );
}
```

---

Event:
```java
public class CompanyWalletDebitedEvent {

    private Long payrollId;

    private Long companyId;

    private BigDecimal amount;
}
```

---

Flow:
```
PayrollPaymentInitiatedEvent
              |
              v
        Wallet Service
              |
              v
CompanyWalletDebitedEvent
```

---

## Step 3

Bank Service listens.

```java
@RabbitListener(...)
public void consume(
        CompanyWalletDebitedEvent event) {

    bankApi.transfer(
            event.getAmount()
    );

    publish(
        SalaryTransferredEvent
    );
}
```

---

If transfer succeeds:

```
CompanyWalletDebitedEvent
             |
             v
         Bank Service
             |
             v
 SalaryTransferredEvent
```

---

## Step 4

Payroll Service listens.

```java
@RabbitListener(...)
public void consume(
        SalaryTransferredEvent event) {

    payrollRepository.updateStatus(
            event.getPayrollId(),
            PayrollStatus.PAID
    );
}
```

Saga completed.

---

# Happy Path

```
PayrollPaymentInitiatedEvent
               |
               v
        Wallet Service
               |
               v
CompanyWalletDebitedEvent
               |
               v
         Bank Service
               |
               v
SalaryTransferredEvent
               |
               v
        Payroll Service
               |
               v
      Payroll = PAID
```

---

# Failure Scenario

Suppose Bank API fails.

---

Wallet already debited:

```
Company Wallet = 100000

Debit 50000

Remaining = 50000
```

---

Transfer fails:

```java
try {

    bankApi.transfer(...);

} catch(Exception e) {

    publish(
        SalaryTransferFailedEvent
    );
}
```

---

Event:

```java
public class SalaryTransferFailedEvent {

    private Long payrollId;

    private Long companyId;

    private BigDecimal amount;
}
```

---

Flow:

```
CompanyWalletDebitedEvent
             |
             v
         Bank Service
             |
             X
          FAILED
             |
             v
SalaryTransferFailedEvent
```

---

## Compensation

Wallet Service listens.

```java
@RabbitListener(...)
public void consume(
        SalaryTransferFailedEvent event) {

    walletService.credit(
            event.getCompanyId(),
            event.getAmount()
    );

    publish(
        WalletRefundedEvent
    );
}
```

---

Flow:

```
SalaryTransferFailedEvent
             |
             v
        Wallet Service
             |
             v
      Refund Wallet
             |
             v
     WalletRefundedEvent
```

---

## Payroll Service

```java
@RabbitListener(...)
public void consume(
        WalletRefundedEvent event) {

    payrollRepository.updateStatus(
            event.getPayrollId(),
            PayrollStatus.FAILED
    );
}
```

---

# Failure Flow

```
PayrollPaymentInitiatedEvent
               |
               v
        Wallet Service
               |
               v
CompanyWalletDebitedEvent
               |
               v
         Bank Service
               |
               X
           FAILURE
               |
               v
SalaryTransferFailedEvent
               |
               v
        Wallet Service
               |
               v
      Refund Wallet
               |
               v
WalletRefundedEvent
               |
               v
       Payroll Service
               |
               v
      Payroll = FAILED
```

---

# Where Reporting and Notification Fit

These are not part of the Saga transaction.

They react to final business outcomes.

---

### Success

```
SalaryTransferredEvent
         |
         +--> Reporting Service
         |
         +--> Notification Service
```

Notification:

```java
@RabbitListener(...)
public void consume(
        SalaryTransferredEvent event) {

    smsService.sendSuccessSms(...);
}
```

---

### Failure

```
WalletRefundedEvent
         |
         +--> Reporting Service
         |
         +--> Notification Service
```

Notification:

```java
@RabbitListener(...)
public void consume(
        WalletRefundedEvent event) {

    smsService.sendFailureSms(...);
}
```

---

# Final Architecture

```
                    ┌─────────────────┐
                    │ Payroll Service │
                    └────────┬────────┘
                             |
                             v
          PayrollPaymentInitiatedEvent
                             |
                             v
                    ┌────────────────┐
                    │ Wallet Service │
                    └────────┬───────┘
                             |
                             v
               CompanyWalletDebitedEvent
                             |
                             v
                     ┌─────────────┐
                     │ Bank Service│
                     └──────┬──────┘
                            |
              ┌─────────────┴─────────────┐
              |                           |
              v                           v
SalaryTransferredEvent     SalaryTransferFailedEvent
              |                           |
              |                           v
              |                 Wallet Compensation
              |                           |
              |                           v
              |                WalletRefundedEvent
              |                           |
              └──────────────┬────────────┘
                             v
                    Payroll Final Status

                     Side Effects
                             |
             --------------------------------
             |                              |
             v                              v
      Notification Service         Reporting Service
```

The key distinction is:

- **RabbitMQ Event-Driven only:** events trigger independent actions, no rollback.
- **Saga over RabbitMQ:** events represent steps of a business transaction, and failure events trigger compensating actions to restore consistency.