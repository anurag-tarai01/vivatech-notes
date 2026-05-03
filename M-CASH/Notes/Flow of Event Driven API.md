Instead of doing this (traditional way):
```java
Controller → Service → Repository → DB
```

Your system does:
```java
Command → Aggregate → Event → (multiple handlers react)
```

👉 **You don’t directly save data**  
👉 You **emit events**, and others react to them