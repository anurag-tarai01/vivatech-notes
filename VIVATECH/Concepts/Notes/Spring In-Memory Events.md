Best for:
- Single application
- Monolith
- No microservices

---

## Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
```

Nothing extra needed.

---

## Event

```java
@Getter
@AllArgsConstructor
public class PayrollCreatedEvent {

    private Long payrollId;
    private String employeeName;
}
```

---

## Publisher

```java
@Service
@RequiredArgsConstructor
public class PayrollService {

    private final ApplicationEventPublisher publisher;

    public void createPayroll() {

        Long payrollId = 101L;

        publisher.publishEvent(
                new PayrollCreatedEvent(
                        payrollId,
                        "Anurag"
                )
        );
    }
}
```

---

## SMS Listener

```java
@Component
public class SmsListener {

    @EventListener
    public void handle(
            PayrollCreatedEvent event) {

        System.out.println(
                "SMS sent for payroll "
                + event.getPayrollId()
        );
    }
}
```

---

## Audit Listener

```java
@Component
public class AuditListener {

    @EventListener
    public void handle(
            PayrollCreatedEvent event) {

        System.out.println(
                "Audit log created"
        );
    }
}
```

---

## Flow

```bash
PayrollService
      |
      v
PayrollCreatedEvent
      |
      +--> SmsListener
      |
      +--> AuditListener
```