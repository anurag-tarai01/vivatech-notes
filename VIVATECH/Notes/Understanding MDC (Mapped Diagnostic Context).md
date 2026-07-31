# Understanding MDC (Mapped Diagnostic Context)

## What is MDC?

**MDC (Mapped Diagnostic Context)** is a feature provided by **SLF4J/Logback** that allows us to store contextual information (such as a trace ID or user ID) for the **current thread**.

The logger can automatically include this information in every log statement without us passing it around manually.

---

## Why do we need MDC?

Imagine a banking application processing hundreds of requests simultaneously.

```
User A
    │
    ▼
Controller
    │
    ▼
Service
    │
    ▼
Repository
```

Each layer writes logs:

```
Request received
Checking balance
Money transferred
SMS sent
```

Now imagine multiple users making requests at the same time.

Logs become:

```
Request received
Checking balance
Request received
Money transferred
SMS sent
Checking balance
```

It becomes impossible to know which logs belong to which request.

---

## The Solution: Request Tracing

Assign every request a unique ID.

Example:

```
traceId = abc123
```

Every log now includes this trace ID:

```
[abc123] Request received
[abc123] Checking balance
[abc123] Money transferred
[abc123] SMS sent
```

Another request gets a different ID:

```
[xyz789] Request received
[xyz789] Checking balance
```

Now it's easy to follow the lifecycle of a single request.

---

## Without MDC

Every method would need to receive the trace ID.

```java
controller(traceId)
```

↓

```java
service(traceId)
```

↓

```java
repository(traceId)
```

↓

```java
notification(traceId)
```

Method signatures become cluttered:

```java
public void sendSms(
    String msisdn,
    String body,
    String traceId
)
```

This is not ideal.

---

## With MDC

MDC stores contextual data in a **ThreadLocal**.

Think of every thread as having its own backpack.

```
Thread-1

Backpack
---------
traceId = abc123
userId = 1001
country = SO
```

Whenever a log statement executes,

```java
log.info("SMS sent");
```

the logger automatically reads the values from the backpack and prints:

```
[abc123] SMS sent
```

No need to pass `traceId` everywhere.

---

## Adding Data to MDC

Typically done at the beginning of request processing.

```java
MDC.put("traceid", UUID.randomUUID().toString());
```

Now every log on that thread automatically includes:

```
traceid=abc123
```

---

## Logging Configuration

Our project uses:

```properties
logging.pattern.console=...%X{traceid}...
```

`%X{traceid}` means:

> Print the MDC value stored under the key `traceid`.

If MDC contains:

```
traceid = abc123
```

Logs become:

```
INFO abc123 ---
```

If MDC is empty:

```
INFO ---
```

---

# Important: MDC is Thread-Local

MDC belongs to the **current thread only**.

Example:

```
Request Thread
--------------
traceId = abc123
```

Another thread starts:

```
Worker Thread
-------------
(empty)
```

The new thread does **not** inherit the MDC automatically.

---

# Why do we copy the MDC?

Our code does:

```java
Map<String, String> mdcContextMap = MDC.getCopyOfContextMap();
```

This creates a copy of the current thread's MDC.

Then an asynchronous task is started:

```java
taskExecutor.execute(...)
```

The worker thread starts with an empty MDC.

To preserve the logging context:

```java
MDC.setContextMap(mdcContextMap);
```

The worker thread now has the same trace ID.

```
Request Thread
--------------
traceId = abc123

        │ Copy
        ▼

Worker Thread
-------------
traceId = abc123
```

Now logs from both threads share the same trace ID.

---

# Why did the NullPointerException occur?

Sometimes:

```java
MDC.getCopyOfContextMap()
```

returns:

```java
null
```

instead of a map.

Why?

Because the current thread's MDC is already empty.

Example:

```
Current Thread

(empty)
```

So:

```java
Map<String, String> mdcContextMap = null;
```

Later:

```java
MDC.setContextMap(mdcContextMap);
```

becomes:

```java
MDC.setContextMap(null);
```

Internally, Logback performs something similar to:

```java
context.putAll(map);
```

which becomes:

```java
putAll(null);
```

Result:

```
NullPointerException
```

---

# The Immediate Fix

Instead of:

```java
MDC.setContextMap(mdcContextMap);
```

use:

```java
if (mdcContextMap != null) {
    MDC.setContextMap(mdcContextMap);
}
```

Now:

### Case 1

```
mdcContextMap != null
```

Restore the MDC.

Everything works.

### Case 2

```
mdcContextMap == null
```

Skip restoring.

No exception occurs.

---

# Why call `MDC.clear()`?

Thread pool threads are reused.

Example:

### Task 1

```
Worker Thread

traceId = ABC123
```

Task finishes.

The thread returns to the pool.

---

### Task 2

The same worker thread is reused.

Without clearing MDC:

```
Worker Thread

traceId = ABC123
```

The new request incorrectly inherits the previous request's trace ID.

This is called **MDC leakage**.

To prevent this:

```java
finally {
    MDC.clear();
}
```

The worker thread returns to the pool with an empty MDC.

---

# Does the null check hide the real problem?

No.

The null check prevents the application from crashing.

However, it does **not** explain why the MDC was empty.

There are two separate concerns:

1. **Defensive Programming**
   - Prevent `NullPointerException`
   - Use:
     ```java
     if (mdcContextMap != null) {
         MDC.setContextMap(mdcContextMap);
     }
     ```

2. **Root Cause Analysis**
   - Investigate why the MDC is empty.
   - Search for:
     ```java
     MDC.put(...)
     ```
   - Determine where the application initializes the `traceid`.
   - Verify whether that initialization is skipped for certain execution paths (e.g., RabbitMQ consumers, scheduled jobs, async tasks).

---

# Key Takeaways

- MDC stores contextual information (trace ID, user ID, etc.) for the current thread.
- It allows loggers to automatically include contextual information in every log statement.
- MDC is implemented using `ThreadLocal`, so each thread has its own independent context.
- Asynchronous threads do not inherit the parent's MDC automatically.
- `MDC.getCopyOfContextMap()` copies the current thread's context.
- `MDC.setContextMap()` restores that context in another thread.
- Calling `MDC.setContextMap(null)` causes a `NullPointerException`.
- Always check for `null` before calling `setContextMap()`.
- Always call `MDC.clear()` when a pooled thread finishes work.
- The null check prevents crashes, but the missing MDC should still be investigated if request tracing is expected.