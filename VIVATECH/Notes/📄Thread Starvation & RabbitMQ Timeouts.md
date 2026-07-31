
## 1. What is the Issue?

During the Maker-Checker approval flow (`/approve`), the application experienced severe performance degradation leading to failed transactions.

The application logs revealed two critical cascading errors:

1. **HikariPool Thread Starvation:** `WARN : HikariPool-1 - Thread starvation or clock leap detected (housekeeper delta=58s...)`
    
2. **RabbitMQ Connection Timeout:** `ERROR : org.springframework.amqp.AmqpTimeoutException: java.util.concurrent.TimeoutException`
    

Interestingly, the exact same business logic executed perfectly when triggered via the `/execute` endpoint. The failures were exclusively isolated to the `/approve` endpoint.

## 2. Why it Happened: The Root Cause

The root cause of this failure is an architectural anti-pattern known as **Transaction Scope Bloat**.

The issue originated from placing a `@Transactional` annotation on the outer wrapper method (`approveAndExecuteRequest`) that contained a heavy, long-running iteration loop (`processDisbursement`).

### The Chain of Events:

1. **The Lock-Up:** When the Checker clicked "Approve", Spring Boot opened a master database transaction and checked out exactly one database connection from the HikariPool.
    
2. **The Hostage Situation:** Because the `@Transactional` annotation was at the top level, that single database connection was held "hostage" for the _entire_ duration of the method. It could not be returned to the pool until every single commission in the batch was processed.
    
3. **The Starvation:** As the heavy loop slowly processed the batch, the Hikari Connection Pool ran completely dry.
    
4. **The Collision:** At the same time, scheduled background tasks (like `SwitchWalletReceiveMoneyStatusCheckJob`) woke up and tried to borrow database connections. Because the pool was empty, these threads froze, causing a complete system bottleneck.
    
5. **The Timeout:** With the application's threads completely deadlocked waiting for database resources, the RabbitMQ driver lost patience waiting for a connection/acknowledgment to publish the transfer events, resulting in the `AmqpTimeoutException`.
    

**Why did `/execute` work?** The `/execute` endpoint called the loop directly _without_ an outer `@Transactional` annotation. This allowed the loop to rapidly borrow and return database connections for each individual payout, keeping the connection pool healthy and flowing.

## 3. How to Fix It

To resolve this issue permanently, you must address both the code architecture and the infrastructure configuration.

### A. The Code Fix (Primary Solution)

Remove the outer transaction boundary from long-running batch processes.

- **DO NOT** put `@Transactional` on methods that loop over large datasets or make multiple sequential third-party network calls.
    
- **DO** allow the inner methods (e.g., `gPayAccountTransferService.processTransaction`) to manage their own short-lived, isolated transactions.
    

Java

```
// BAD: Holds the DB connection open during the entire loop
@Transactional(rollbackFor = Exception.class) 
public BaseResponseDto approveAndExecuteRequest(...) {
    // ...
    processDisbursement(disbursementTransferType); // Heavy loop
    // ...
}

// GOOD: Connection pool can breathe. Inner methods handle their own transactions.
public BaseResponseDto approveAndExecuteRequest(...) {
    // ...
    processDisbursement(disbursementTransferType); // Heavy loop
    // ...
}
```

### B. The Infrastructure Safety Nets (Secondary Solutions)

To make the application more resilient to future traffic spikes, apply the following optimizations to your `application.properties`:

1. **Optimize HikariPool:** Balance the pool to handle concurrent jobs and approval batches.
    
    Properties
    
    ```
    spring.datasource.hikari.maximum-pool-size=20
    spring.datasource.hikari.minimum-idle=5
    # Drop idle connections quickly to prevent stale socket errors
    spring.datasource.hikari.idle-timeout=30000 
    ```
    
2. **Increase RabbitMQ Tolerance:** Give the message broker slightly more time to establish connections during heavy CPU loads.
    
    Properties
    
    ```
    spring.rabbitmq.connection-timeout=60000
    ```
    
3. **Audit Background Jobs:** Ensure that scheduled Quartz jobs (like `SwitchWalletReceiveMoneyStatusCheckJob`) use highly optimized, indexed database queries. If a background job performs a full table scan while an Admin is approving a batch, it will instantly recreate the thread starvation issue.