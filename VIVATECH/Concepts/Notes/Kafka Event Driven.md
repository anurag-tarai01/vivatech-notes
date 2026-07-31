Most common for microservices.

---

## Dependency

```xml
<dependency>  
<groupId>org.springframework.kafka</groupId>  
<artifactId>spring-kafka</artifactId>  
</dependency>
```

---

## application.yml

```yml
spring:
  kafka:
    bootstrap-servers: localhost:9092

    producer:
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

    consumer:
      group-id: payroll-group
      auto-offset-reset: earliest

      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer

      properties:
        spring.json.trusted.packages: "*"
```

---

## Event DTO

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class PayrollCreatedEvent {

    private Long payrollId;
}
```

---

## Producer

```java
@Service
@RequiredArgsConstructor
public class PayrollService {

    private final KafkaTemplate<String,
            PayrollCreatedEvent> kafkaTemplate;

    public void createPayroll() {

        PayrollCreatedEvent event =
                new PayrollCreatedEvent(101L);

        kafkaTemplate.send(
                "payroll-created-topic",
                event
        );
    }
}
```

---

## Consumer 1

```java
@Component
public class SmsConsumer {

    @KafkaListener(
            topics = "payroll-created-topic",
            groupId = "sms-group"
    )
    public void consume(
            PayrollCreatedEvent event) {

        System.out.println(
                "SMS sent"
        );
    }
}
```

---

## Consumer 2

```java
@Component
public class AuditConsumer {

    @KafkaListener(
            topics = "payroll-created-topic",
            groupId = "audit-group"
    )
    public void consume(
            PayrollCreatedEvent event) {

        System.out.println(
                "Audit saved"
        );
    }
}
```

---

Flow:

```
Payroll Service
       |
       v
Kafka Topic
       |
       +--> SMS Service
       |
       +--> Audit Service
```