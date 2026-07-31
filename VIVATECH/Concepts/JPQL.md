# JPQL Notes

## What is JPQL?

JPQL (Java Persistence Query Language) is a query language used by JPA/Hibernate.

Unlike SQL, JPQL works with **Entities and their fields**, not directly with database tables and columns.

### SQL

```sql
SELECT * FROM salary_payment
```

### JPQL

```java
SELECT p FROM SalaryPayment p
```

Where:

- `SalaryPayment` = Entity class
    
- `p` = Alias
    

---

## Selecting Specific Fields

```java
SELECT p.msisdn
FROM SalaryPayment p
```

Equivalent SQL:

```sql
SELECT msisdn
FROM salary_payment
```

---

## Aggregate Functions

### Count

```java
SELECT COUNT(p)
FROM SalaryPayment p
```

### Sum

```java
SELECT SUM(p.amount)
FROM SalaryPayment p
```

### Distinct Count

```java
SELECT COUNT(DISTINCT p.msisdn)
FROM SalaryPayment p
```

Useful when an employee may have Salary, Bonus, and Stipend records but should only be counted once.

---

## JPQL Without Entity Relationships

If two entities are not connected using `@OneToMany`, `@ManyToOne`, etc., use:

```java
FROM SalaryPayment p, SalaryPaymentRequest r
WHERE p.bulkPaymentId = r.uuid
```

This is similar to:

```sql
FROM salary_payment p
INNER JOIN salary_payment_request r
ON p.bulk_payment_id = r.uuid
```

and works well in older Spring Boot / Hibernate projects.

---

## JPQL With Entity Relationships

If entities are mapped:

```java
@ManyToOne
private SalaryPaymentRequest request;
```

Then:

```java
SELECT p
FROM SalaryPayment p
JOIN p.request r
```

is preferred.

---

## JOIN ... ON Support

Modern Hibernate versions support:

```java
FROM SalaryPayment p
JOIN SalaryPaymentRequest r
ON p.bulkPaymentId = r.uuid
```

Older Hibernate/JPA versions may not.

For legacy Java 8 projects, prefer:

```java
FROM SalaryPayment p, SalaryPaymentRequest r
WHERE p.bulkPaymentId = r.uuid
```

for maximum compatibility.

---

## Common Dashboard Example

Total amount processed by a Third Party:

```java
@Query(
"SELECT SUM(p.amount.amount) " +
"FROM SalaryPayment p, SalaryPaymentRequest r " +
"WHERE p.bulkPaymentId = r.uuid " +
"AND r.thirdPartyAggregateId = :thirdPartyId " +
"AND p.status = 'PAID'"
)
```

Workforce paid last month:

```java
@Query(
"SELECT COUNT(DISTINCT p.msisdn) " +
"FROM SalaryPayment p, SalaryPaymentRequest r " +
"WHERE p.bulkPaymentId = r.uuid " +
"AND r.thirdPartyAggregateId = :thirdPartyId " +
"AND p.paymentMonth = :month " +
"AND p.status = 'PAID'"
)
```

---

## Key Rule

JPQL queries:

- Entity names, not table names.
    
- Entity fields, not column names.
    
- Relationships through entity mappings whenever possible.
    
- Use aggregate functions (`COUNT`, `SUM`, `AVG`) to let the database do the work instead of loading large datasets into Java memory.