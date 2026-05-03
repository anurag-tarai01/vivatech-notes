**Axon Framework** is a Java framework that helps you build systems using:

- **CQRS** (Command Query Responsibility Segregation)
- **Event-Driven Architecture**
- **Event Sourcing**

```java
Axon = a framework that enforces:
"Commands → Events → System reacts"
```

# What problem does Axon solve?

In normal apps:

```java
Controller → Service → Repository → DB
```

Problems:

- Tight coupling ❌
- No history ❌
- Hard to extend ❌
- Complex workflows messy ❌

---

Axon changes it to:

```java
Command → Aggregate → Event → Handlers → DB
```

Benefits:

- Loose coupling ✅
- Full history (audit) ✅
- Easy to extend ✅
- Handles complex flows (via Saga) ✅