Same event.

---

## Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```
`spring-boot-starter-aop` provides the proxy/interception infrastructure that Spring uses for features such as `@Async`, `@Transactional`, `@Cacheable`, and custom `@Aspect` classes. Without proxying, Spring cannot intercept method calls to apply those behaviors.

Read More : [[spring-boot-starter-aop]] 

---

## Enable Async

```java
@SpringBootApplication
@EnableAsync
public class Application {
}
```

---

## Listener

```java
@Component
public class SmsListener {

    @Async
    @EventListener
    public void handle(
            PayrollCreatedEvent event) {

        System.out.println(
                Thread.currentThread()
        );

        sendSms();
    }
}
```

---

## Multiple Async Listeners

```java
@Component
public class AuditListener {

    @Async
    @EventListener
    public void handle(
            PayrollCreatedEvent event) {

        createAudit();
    }
}
```

```java
@Component
public class ReportListener {

    @Async
    @EventListener
    public void handle(
            PayrollCreatedEvent event) {

        generateReport();
    }
}
```

---

Flow:

```bash
PayrollCreatedEvent
       |
       +--> Thread-1 SMS
       |
       +--> Thread-2 Audit
       |
       +--> Thread-3 Report
```