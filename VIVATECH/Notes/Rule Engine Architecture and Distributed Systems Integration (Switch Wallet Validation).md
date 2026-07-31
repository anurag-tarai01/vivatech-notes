### 1. Why did you choose to implement the validation using a rule-based engine instead of putting the validation logic inside the `TransferController` or a transactional application service?

- **Focus:** Decoupling, Single Responsibility Principle (SRP), and cyclomatic complexity.
    

### 2. The design pattern used for your validation engine resembles a combination of the Strategy Pattern and the Chain of Responsibility. Can you explain how these patterns apply to your `supports()` and execution flow?

- **Focus:** Design pattern articulation and structural awareness.
    

## State Management & Distributed Systems

### 3. Your `SwitchAccountValidationRule` checks if the `TransferStatus` is null or `STARTED`. Why is tracking the state of the DTO crucial when dealing with asynchronous webhook callbacks from third-party APIs?

- **Focus:** Idempotency, network safety, and preventing duplicate processing.
    

### 4. What would happen if a callback request hit the `SwitchWalletHookController` while the initial transaction's database commit was still in-flight, and how does your architecture prevent race conditions?

- **Focus:** Concurrency, distributed systems, and transactional boundaries.
    

## Data Mapping & Infrastructure

### 5. We observed that during the webhook callback phase, the `TransferEventDto` was missing the original metadata payload because the mapper did not fetch child records from `AccountTransferMetadata`. How would you fix this mapping layer permanently without relying solely on the status check?

- **Focus:** Hibernate/JPA fetch strategies (Eager vs. Lazy), DTO mapping configurations, and object graph navigation.
    

### 6. Why is it considered an anti-pattern to modify or mutate the state of an incoming `AccountTransfer` entity directly inside a read-only metadata logging method like `saveCallbackMetadata`?

- **Focus:** Side-effects, persistence context management, and clean code principles.
    

## Network & Security Integration

### 7. In your Feign Client setup, you removed manual OAuth token generation because a global `RequestInterceptor` was already active. How does a Feign `RequestInterceptor` work, and what are the risks of header collisions if manual tokens are also injected?

- **Focus:** HTTP/Spring Cloud routing, interceptor lifecycles, and security frameworks.
    

### 8. You decoupled the base URLs for standard transactions and validation routing using dynamic URI injection in Feign. Why is this a safer approach than routing validation queries through the primary write-transaction endpoint?

- **Focus:** Separation of concerns at the network layer and system safety.
    

## Enterprise Readiness & Resilience

### 9. Your configuration utilizes a fallback default value inside the `@Value` annotation (`:true`). Why is this pattern essential when working within a multi-module Maven architecture (`common`, `wallet`, `reporting`)?

- **Focus:** Build tools, Spring Bean initialization lifecycle, and cross-module classpath safety.
    

### 10. If the Switch network goes down completely (Timeout / 504 Gateway Error) during a bulk payroll run, how will your validation rule handle the exception, and what is the impact on the rest of the unvalidated batch?

- **Focus:** Fault tolerance, circuit breakers, and batch processing resilience.