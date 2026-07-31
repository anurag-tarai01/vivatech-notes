For the **Async Event Driven** example, I included:

```
<dependency>    <groupId>org.springframework.boot</groupId>    <artifactId>spring-boot-starter-aop</artifactId></dependency>
```

The reason is that features like:

```
@Async@Transactional@Cacheable@Retryable
```

are implemented by Spring using **AOP (Aspect-Oriented Programming) proxies**.

---

## What happens without AOP?

When Spring sees:

```
@Asyncpublic void sendSms() {    ...}
```

it doesn't magically create a new thread.

Instead, Spring creates a **proxy object** around your bean.

```
Caller   |   vSpring Proxy   |   +--> Create Thread   |   +--> Call Actual Method
```

Without the proxy mechanism, Spring cannot intercept the method call and execute it asynchronously.

---

## Example

Your code:

```
@Servicepublic class SmsService {    @Async    public void sendSms() {        System.out.println(            Thread.currentThread().getName()        );    }}
```

What Spring effectively creates:

```
SmsServiceProxy        |        +--> ExecutorService.submit(...)                |                v          SmsService.sendSms()
```

The proxy logic comes from Spring's AOP infrastructure.

---

## Why does it sometimes work without explicitly adding AOP?

If you use:

```
spring-boot-starter-web
```

or some other starters, Spring may already bring AOP-related libraries transitively.

For example:

```
spring-boot-starter
```

already includes core Spring libraries.

In many Spring Boot 3 projects:

```
@EnableAsync
```

works without explicitly adding:

```
spring-boot-starter-aop
```

because the required proxy infrastructure is already present.

---

## When is `spring-boot-starter-aop` definitely needed?

When you use:

```
@Aspect@Before@After@Around
```

Example:

```
@Aspect@Componentpublic class LoggingAspect {    @Before(      "execution(* com.company.service.*.*(..))"    )    public void log() {        System.out.println("Called");    }}
```

Without:

```
spring-boot-starter-aop
```

Spring cannot create and process these aspects.

---

## For `@Async`, should you add it?

### Option 1 (Common)

```
spring-boot-starter-aop
```

Add it explicitly.

Pros:

- Clear dependency
- No surprises
- Works consistently

### Option 2

Rely on transitive dependencies.

Pros:

- Fewer dependencies

Cons:

- Future dependency changes may break behavior

---

## In your backend

If you have code like:

```
@EnableAsync
```

and

```
@Async@EventListenerpublic void handle(...) {}
```

check your dependency tree:

```
mvn dependency:tree
```

You may find AOP libraries already included transitively.

That's why many projects appear to use `@Async` successfully even without explicitly declaring:

```
<artifactId>spring-boot-starter-aop</artifactId>
```

The short answer:

> `spring-boot-starter-aop` provides the proxy/interception infrastructure that Spring uses for features such as `@Async`, `@Transactional`, `@Cacheable`, and custom `@Aspect` classes. Without proxying, Spring cannot intercept method calls to apply those behaviors.