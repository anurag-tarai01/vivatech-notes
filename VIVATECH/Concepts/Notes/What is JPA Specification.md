

**JPA (Java Persistence API)** is a **specification**, not an implementation.

Think of it like this:

```
JPA = Rules / ContractHibernate = Implementation of those rules
```

Similar to:

```
List = InterfaceArrayList = Implementation
```

or

```
Servlet = SpecificationTomcat = Implementation
```

---

## Why does JPA exist?

Before JPA, every ORM framework had its own API.

Examples:

- Hibernate
- TopLink
- OpenJPA

Code written for one ORM was difficult to move to another.

JPA defined a standard way to:

- Map entities
- Persist objects
- Query data
- Manage relationships

so developers could write:

```
@Entitypublic class Employee {}
```

instead of Hibernate-specific code.

---

## JPA vs Hibernate

### JPA

Defines:

```
@Entity@Table@Id@OneToMany@ManyToOne
```

and

```
EntityManagerJPQL
```

But JPA itself contains no database logic.

---

### Hibernate

Actually executes:

```
employeeRepository.save(employee);
```

Generates:

```
INSERT INTO employee ...
```

Talks to:

- SQL Server
- MySQL
- PostgreSQL
- Oracle

---

## Example

### JPA Code

```
@Entitypublic class Employee {    @Id    private Long id;    private String name;}
```

### Repository

```
public interface EmployeeRepository        extends JpaRepository<Employee, Long> {}
```

This works because:

```
Spring Data JPA        ↓Hibernate        ↓SQL Server
```

---

## JPQL and JPA

JPQL is part of the JPA specification.

JPA says:

```
SELECT e FROM Employee e
```

must work.

Hibernate translates it into:

```
SELECT *FROM employee
```

---

## Why does this matter for your query?

You wrote:

```
FROM SalaryPayment pJOIN SalaryPaymentRequest rON p.bulkPaymentId = r.uuid
```

The important question is:

> Does the JPA specification version used by my project support JOIN ... ON between unrelated entities?

### JPA 2.0 (older)

Usually:

```
FROM SalaryPayment p, SalaryPaymentRequest rWHERE p.bulkPaymentId = r.uuid
```

works.

### JPA 2.1+

Added:

```
JOIN ... ON
```

support.

But support varies depending on:

- Hibernate version
- Spring Boot version

---

## How to know what your project uses?

Check:

### Maven

```
<parent>    <groupId>org.springframework.boot</groupId>    <artifactId>spring-boot-starter-parent</artifactId>    <version>2.1.0.RELEASE</version></parent>
```

or

```
<hibernate.version>...</hibernate.version>
```

---

## Practical rule for legacy projects

For your Java 8 + older Spring Boot project:

```
FROM SalaryPayment p, SalaryPaymentRequest rWHERE p.bulkPaymentId = r.uuid
```

is usually the safest JPQL.

It works across almost all JPA/Hibernate versions and doesn't require entity relationships.

### Quick Interview Answer

> JPA (Java Persistence API) is a Java specification that defines standard APIs and annotations for ORM and database persistence. Hibernate is the most popular implementation of the JPA specification. JPQL is the query language defined by JPA and operates on entities rather than database tables.