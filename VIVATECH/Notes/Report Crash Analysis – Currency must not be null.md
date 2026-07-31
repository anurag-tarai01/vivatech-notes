# Root Cause Analysis – `Currency must not be null`

## Error

```text
java.lang.NullPointerException: Currency must not be null
```

Triggered from:

```text
com.vivacom.mfs.wallet.query.hibernate.type.MoneyUserType.nullSafeGet(MoneyUserType.java:70)
```

Request path:

```text
/transaction-report/third-party/my-transaction-report
```

Main failure flow:

```text
ReportService.getFilteredSpecificUserTransactionReport()
    ->
pagingTypeQuery.getResultList()
    ->
Hibernate Entity Mapping
    ->
MoneyUserType.nullSafeGet()
    ->
CurrencyUnit.of(currencyString)
    ->
NullPointerException
```

---

## Observations

Logs confirm:

```java
CurrencyUnit.of(currencyString)
```

is being called with:

```java
currencyString = null
```

The exception occurs while Hibernate is mapping database rows into:

```java
CustomerTransactionReport
```

entities.

The following code triggers the failure:

```java
List<CustomerTransactionReport> resultList =
        pagingTypeQuery.getResultList();
```

---

## Likely Cause

`getFilteredSpecificUserTransactionReport()` does not perform aggregation queries directly. It loads full `CustomerTransactionReport` entities using Hibernate Criteria API.

Most likely cause:

One or more records contain NULL currency values for a field mapped using:

```java
MoneyUserType
```

Possible scenario:

|amount|currency|
|---|---|
|1254.00|NULL|

When Hibernate maps this row, `MoneyUserType.nullSafeGet()` executes:

```java
CurrencyUnit.of(currencyString)
```

which throws:

```text
NullPointerException: Currency must not be null
```

---

## Recommended Investigation

Inspect:

```java
CustomerTransactionReport
```

for fields using:

```java
@Type(type = "MoneyUserType")
```

or:

```java
BigMoney
Money
```

Then verify corresponding database columns for NULL currency values.

Example SQL:

```sql
SELECT *
FROM customer_transaction_report
WHERE currency IS NULL;
```

or check all money-related currency columns.

---

## Defensive Fix (Optional)

A defensive null check can be added inside:

```java
com.vivacom.mfs.wallet.query.hibernate.type.MoneyUserType
```

Example:

```java
@Value("${mfs.currency-code}")
private String currencyCode;

@Override
public Object nullSafeGet(ResultSet rs, String[] names,
        SessionImplementor session, Object owner)
        throws HibernateException, SQLException {

    BigDecimal amount = rs.getBigDecimal(names[0]);
    String currencyString = rs.getString(names[1]);

    if (amount == null || currencyString == null) {
        return BigMoney.zero(CurrencyUnit.of(currencyCode));
    }

    CurrencyUnit currency = CurrencyUnit.of(currencyString);

    return BigMoney.of(currency, amount);
}
```

Note:  
This may prevent crashes but could also hide invalid database records.

---

## Conclusion

Current evidence indicates:

- Request successfully reaches reporting layer
    
- Failure occurs during Hibernate entity mapping
    
- Root issue is null currency handling in `MoneyUserType`
    
- Most likely triggered by records containing NULL currency values