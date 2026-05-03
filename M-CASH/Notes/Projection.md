Projection = Read Model (a DB view built from events)

In simple words:

> A **projection listens to events and builds data in a format that your APIs can read easily**

This is your projection:
```java
@EventHandler
public void handle(AdminCreatedEvent event) {
    createAdmin(event.getAdminDto());
}
```

👉 This method is part of:

UserListener

✔️ So:

👉 **`UserListener` = Projection**

---
# Why do we need Projection?

Because:

👉 Your **Aggregate does NOT store data like a normal entity**

Instead:

Events stored:  
- AdminCreatedEvent  
- AdminApprovedEvent  
- AdminBlockedEvent

But your API needs:

```java
{  
  "name": "John",  
  "status": "ACTIVE"  
}
```

👉 That format does NOT exist in event store

---

# 🔄 So Projection does this:

Event → Projection → Database Table

---
## Flow
### 1. Event happens

```java
apply(new AdminCreatedEvent(...));
```

---

### 2. Projection listens

```java
@EventHandler  
public void handle(AdminCreatedEvent event)
```

---

### 3. Projection builds DB object

```java
Admin admin = new Admin();  
admin.setFirstName(...);
```

---

### 4. Projection saves to DB

```java
adminUserQueryRepository.save(admin);
```

---

# 🎯 Final Meaning

👉 Projection = **“Transform events into usable database records”**

|Layer|Purpose|
|---|---|
|Event Store|Full history (raw events)|
|Projection|Builds current state|
|DB (User table)|Used by APIs|
