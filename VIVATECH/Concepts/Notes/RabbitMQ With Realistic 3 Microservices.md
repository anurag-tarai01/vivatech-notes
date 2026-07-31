# RabbitMQ Event-Driven Architecture (Backend → Notification → Reporting)

## High-Level Flow

```text
Backend Service
      |
      | Publish PayrollProcessedEvent
      v
+-------------------+
| payroll.exchange  |
+-------------------+
      |
      | Routing Key = payroll.processed
      |
      +-----------------------------+
      |                             |
      v                             v
notification.payroll.queue   reporting.payroll.queue
      |                             |
      v                             v
Notification Service        Reporting Service
```

---

# RabbitMQ Core Components

## Producer

The service publishing a message.

Example:

```java
rabbitTemplate.convertAndSend(
        "payroll.exchange",
        "payroll.processed",
        event
);
```

Producer only knows:

- Exchange Name
    
- Routing Key
    
- Message Payload
    

Producer does NOT know:

- Which queues exist
    
- Which services consume the message
    

---

## Exchange

Exchange is NOT storage.

Exchange acts like a router.

It receives:

```text
Exchange = payroll.exchange

Routing Key = payroll.processed

Message = PayrollProcessedEvent
```

Then decides:

```text
Which queues should receive this message?
```

After routing, exchange's job is finished.

Exchange never stores messages.

---

## Routing Key

Routing key is an address used for routing.

Example:

```text
payroll.processed
```

Other examples:

```text
payroll.failed

payroll.approved

salary.paid
```

The exchange compares the routing key with queue bindings.

---

## Queue

Queue is where messages actually live.

Example:

```text
notification.payroll.queue
```

Internal queue state:

```text
----------------------------------
Message #1
Message #2
Message #3
----------------------------------
```

RabbitMQ stores messages inside queues until consumers process them.

If consumer is down:

```text
Queue continues storing messages.
```

Messages are not lost (assuming durable queue and persistent messages).

---

# Shared Event DTO

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PayrollProcessedEvent {

    private Long payrollId;

    private Long companyId;

    private Long employeeId;

    private BigDecimal amount;

    private String paymentReference;

    private LocalDate payrollMonth;
}
```

Example JSON sent through RabbitMQ:

```json
{
  "payrollId": 101,
  "companyId": 1,
  "employeeId": 1001,
  "amount": 50000,
  "paymentReference": "SAL202506001",
  "payrollMonth": "2026-06-01"
}
```

---

# Backend Service

## Rabbit Configuration

```java
@Configuration
public class RabbitConfig {

    public static final String PAYROLL_EXCHANGE =
            "payroll.exchange";

    @Bean
    public TopicExchange payrollExchange() {

        return new TopicExchange(
                PAYROLL_EXCHANGE
        );
    }
}
```

Application startup creates:

```text
RabbitMQ

└── payroll.exchange
```

At this point:

```text
No queues exist.
No messages exist.
```

---

## Event Publisher

```java
@Service
@RequiredArgsConstructor
public class PayrollEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishPayrollProcessed(
            PayrollProcessedEvent event) {

        rabbitTemplate.convertAndSend(
                RabbitConfig.PAYROLL_EXCHANGE,
                "payroll.processed",
                event
        );
    }
}
```

This sends:

```text
Exchange:
payroll.exchange

Routing Key:
payroll.processed

Payload:
PayrollProcessedEvent
```

to RabbitMQ.

---

## Payroll Service

```java
@Service
@RequiredArgsConstructor
public class PayrollService {

    private final PayrollRepository payrollRepository;

    private final PayrollEventPublisher publisher;

    @Transactional
    public void processPayroll(
            Payroll payroll) {

        Payroll savedPayroll =
                payrollRepository.save(payroll);

        PayrollProcessedEvent event =
                PayrollProcessedEvent.builder()
                        .payrollId(savedPayroll.getId())
                        .companyId(savedPayroll.getCompanyId())
                        .employeeId(savedPayroll.getEmployeeId())
                        .amount(savedPayroll.getAmount())
                        .paymentReference(
                                savedPayroll.getReference()
                        )
                        .payrollMonth(
                                savedPayroll.getPayrollMonth()
                        )
                        .build();

        publisher.publishPayrollProcessed(
                event
        );
    }
}
```

Business flow:

```text
Save Payroll
      |
      v
Create Event
      |
      v
Publish Event
```

Backend work ends here.

---

# Notification Service

## Queue and Binding

```java
@Configuration
public class NotificationRabbitConfig {

    public static final String NOTIFICATION_QUEUE =
            "notification.payroll.queue";

    @Bean
    public Queue notificationQueue() {

        return QueueBuilder
                .durable(NOTIFICATION_QUEUE)
                .build();
    }

    @Bean
    public Binding notificationBinding(
            Queue notificationQueue,
            TopicExchange payrollExchange) {

        return BindingBuilder
                .bind(notificationQueue)
                .to(payrollExchange)
                .with("payroll.processed");
    }
}
```

Creates:

```text
notification.payroll.queue
```

Binding:

```text
payroll.exchange
       |
       +---- payroll.processed
       |
       v
notification.payroll.queue
```

Meaning:

```text
Whenever payroll.processed arrives,
copy the message into notification.payroll.queue
```

---

## Consumer

```java
@Component
@RequiredArgsConstructor
public class PayrollNotificationConsumer {

    private final SmsService smsService;

    @RabbitListener(
            queues =
            NotificationRabbitConfig
                    .NOTIFICATION_QUEUE
    )
    public void consume(
            PayrollProcessedEvent event) {

        smsService.sendSalaryCreditSms(
                event.getEmployeeId(),
                event.getAmount()
        );
    }
}
```

Consumer reads messages from queue.

Consumer never talks to exchange directly.

---

# Reporting Service

## Queue and Binding

```java
@Configuration
public class ReportingRabbitConfig {

    public static final String REPORT_QUEUE =
            "reporting.payroll.queue";

    @Bean
    public Queue reportQueue() {

        return QueueBuilder
                .durable(REPORT_QUEUE)
                .build();
    }

    @Bean
    public Binding reportBinding(
            Queue reportQueue,
            TopicExchange payrollExchange) {

        return BindingBuilder
                .bind(reportQueue)
                .to(payrollExchange)
                .with("payroll.processed");
    }
}
```

Creates:

```text
reporting.payroll.queue
```

Binding:

```text
payroll.exchange
       |
       +---- payroll.processed
       |
       v
reporting.payroll.queue
```

---

## Consumer

```java
@Component
@RequiredArgsConstructor
public class PayrollReportConsumer {

    private final PayrollReportService reportService;

    @RabbitListener(
            queues =
            ReportingRabbitConfig.REPORT_QUEUE
    )
    public void consume(
            PayrollProcessedEvent event) {

        reportService.updatePayrollSummary(
                event.getPayrollId(),
                event.getAmount()
        );
    }
}
```

---

# End-to-End Runtime

### Step 1

Backend publishes:

```text
Exchange:
payroll.exchange

Routing Key:
payroll.processed

Message:
PayrollProcessedEvent
```

---

### Step 2

Exchange checks bindings:

```text
notification.payroll.queue

reporting.payroll.queue
```

Both match:

```text
payroll.processed
```

---

### Step 3

RabbitMQ copies message.

Result:

```text
notification.payroll.queue
----------------------------------
PayrollProcessedEvent
----------------------------------

reporting.payroll.queue
----------------------------------
PayrollProcessedEvent
----------------------------------
```

Important:

```text
Exchange stores nothing.

Queues store messages.
```

---

### Step 4

Notification Service consumes:

```text
notification.payroll.queue
```

and sends SMS.

---

### Step 5

Reporting Service consumes:

```text
reporting.payroll.queue
```

and updates reports.

---

# RabbitMQ Management UI

Usually:

```text
http://localhost:15672
```

You can inspect:

## Exchanges

```text
payroll.exchange
wallet.exchange
user.exchange
```

---

## Queues

```text
notification.payroll.queue

reporting.payroll.queue
```

Shows:

```text
Ready Messages

Unacked Messages

Consumers
```

---

## Message Inspection

Queue page → Get Messages

Example:

```json
{
  "payrollId":101,
  "employeeId":1001,
  "amount":50000
}
```

This allows debugging actual messages stored inside queues.

---

# Key Takeaways

```text
Producer
   |
   v
Exchange (router only)
   |
   v
Queue (message storage)
   |
   v
Consumer
```

Remember:

1. Producer sends to Exchange.
    
2. Exchange routes using Routing Key.
    
3. Queue stores messages.
    
4. Consumer reads from Queue.
    
5. Exchange never stores messages.
    
6. RabbitMQ Management UI lets you see exchanges, queues, bindings, consumers, and messages.
    
7. Multiple queues can receive copies of the same message.
    
8. New services can be added by creating new queues and bindings without changing producer code.