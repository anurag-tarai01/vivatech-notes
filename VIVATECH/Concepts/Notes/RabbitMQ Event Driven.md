Popular alternative to Kafka.

---

## Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

---

## application.yml

```bash
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
```

---

## Exchange Config

```java
@Configuration
public class RabbitConfig {

    @Bean
    public TopicExchange exchange() {

        return new TopicExchange(
                "payroll.exchange"
        );
    }

    @Bean
    public Queue queue() {

        return new Queue(
                "payroll.queue"
        );
    }

    @Bean
    public Binding binding(
            Queue queue,
            TopicExchange exchange) {

        return BindingBuilder
                .bind(queue)
                .to(exchange)
                .with("payroll.created");
    }
}
```

---

## Producer

```java
@Service
@RequiredArgsConstructor
public class PayrollService {

    private final RabbitTemplate rabbitTemplate;

    public void createPayroll() {

        rabbitTemplate.convertAndSend(
                "payroll.exchange",
                "payroll.created",
                new PayrollCreatedEvent(101L)
        );
    }
}
```

---

## Consumer

```java
@Component
public class PayrollConsumer {

    @RabbitListener(
            queues = "payroll.queue"
    )
    public void consume(
            PayrollCreatedEvent event) {

        System.out.println(
                "Received event"
        );
    }
}
```

---

# Production Problem

Suppose:

```
savePayroll();publishEvent();
```

What if:

```
savePayroll()    SUCCESS
publishEvent()   FAILED
```

Now database says payroll exists.

Kafka says it doesn't.

Data inconsistency.

---

# Solution: Outbox Pattern

Instead of:

```
savePayroll();
publishKafkaEvent();
```

Do:

```
savePayroll();
saveOutboxRecord();
```

same database transaction.

---

## Outbox Table

```sql
CREATE TABLE outbox_event
(
    id BIGINT,
    event_type VARCHAR(50),
    payload TEXT,
    status VARCHAR(20)
);
```

---

Transaction:

```java
@Transactional
public void createPayroll() {

    payrollRepository.save(payroll);

    outboxRepository.save(
            eventRecord
    );
}
```

---

Background Job

```java
@Scheduled(fixedDelay = 5000)
public void publishEvents() {

    List<OutboxEvent> events =
            repository.findPending();

    for(OutboxEvent event : events) {

        kafkaTemplate.send(...);

        event.setStatus("PUBLISHED");
    }
}
```

---

Flow:

```
DB Transaction
      |
      +--> Payroll Table
      |
      +--> Outbox Table

Scheduler
      |
      v
Kafka
```

This is how many production systems guarantee reliable event delivery.

---

# What You Should Learn First

For your current backend work, learn in this order:

### Level 1

```java
ApplicationEventPublisher
@EventListener
```

---

### Level 2

```
@AsyncEventListener
```

---

### Level 3

```
Kafka Producer
Kafka Consumer
Topics
Consumer Groups
Retries
DLQ
```

---

### Level 4

```
Outbox Pattern
```

---

### Level 5

```
Saga Pattern(Event Choreography)
```

Most enterprise payroll, wallet, transfer, and payment systems are effectively:

```
REST API
   |
Database Transaction
   |
Outbox Event
   |
Kafka
   |
Multiple Consumers
   |
Saga (for business transactions)
```