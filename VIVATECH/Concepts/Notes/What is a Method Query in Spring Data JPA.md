A **Method Query** (also called a **Derived Query Method**) is a query that Spring Data JPA generates automatically from the repository method name.

You don't write `@Query` or SQL/JPQL manually.

---

## Example

### Entity

```
@Entitypublic class Employee {    @Id    private Long id;    private String msisdn;    private UserStatus status;}
```

### Repository

```
public interface EmployeeRepository extends JpaRepository<Employee, Long> {    List<Employee> findByStatus(UserStatus status);}
```

Spring automatically generates:

```
SELECT *FROM employeeWHERE status = ?
```

---

## Multiple Conditions

```
List<Employee> findByMsisdnAndStatus(        String msisdn,        UserStatus status);
```

Equivalent:

```
SELECT *FROM employeeWHERE msisdn = ?  AND status = ?
```

---

## In Your Project

You already have examples like:

```
vendorRepository.findByAccountNumberInAndThirdParty(        formattedAccountNumbers,        thirdParty);
```

Spring derives:

```
SELECT *FROM vendorWHERE account_number IN (...)  AND third_party = ?
```

without writing SQL or JPQL.

---

## Common Keywords

### Equals

```
findByStatus(status)
```

### And

```
findByMsisdnAndStatus(msisdn, status)
```

### Or

```
findByStatusOrRole(status, role)
```

### In

```
findByMsisdnIn(list)
```

### Not

```
findByStatusNot(status)
```

### Like

```
findByNameLike(name)
```

### Ignore Case

```
findByMsisdnIgnoreCase(msisdn)
```

### Order By

```
findByStatusOrderByCreatedDateDesc(status)
```

---

## Count Query

Method:

```
long countByStatus(SalaryStatus status);
```

Spring generates:

```
SELECT COUNT(*)FROM salary_paymentWHERE status = ?
```

---

## Exists Query

Method:

```
boolean existsByMsisdn(String msisdn);
```

Spring generates an efficient existence check.

---

## When Method Queries Are Good

Simple filters:

```
findByStatus(...)findByMsisdn(...)countByStatus(...)existsByMsisdn(...)
```

These are easy to read and maintain.

---

## When to Use `@Query`

When the query becomes complex:

```
@Query("SELECT COUNT(DISTINCT p.msisdn) " +"FROM SalaryPayment p, SalaryPaymentRequest r " +"WHERE p.bulkPaymentId = r.uuid " +"AND r.thirdPartyAggregateId = :thirdPartyId")
```

This cannot be expressed cleanly as a method name.

---

## Rule of Thumb

### Use Method Queries for:

```
findBy...countBy...existsBy...deleteBy...
```

with straightforward conditions.

### Use `@Query` for:

- `JOIN`
- `SUM`
- `COUNT(DISTINCT ...)`
- `GROUP BY`
- Complex filtering
- Dashboard/reporting queries

Your dashboard queries (`SUM`, `COUNT DISTINCT`, joining `SalaryPayment` and `SalaryPaymentRequest`) are a good use case for `@Query`, not method queries.