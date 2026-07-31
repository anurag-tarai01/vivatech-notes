## Why do we use BigMoneySerializer and BigMoneyDeserializer?

`BigMoney` is a complex Java object from the Joda-Money library.  
Frontend applications and APIs communicate using JSON, which only understands simple data types like strings and numbers.

So, Serializer and Deserializer act as translators between:

```
Java Object  <->  JSON
```

---

# BigMoneySerializer (Java → JSON)

`BigMoneySerializer` is used when sending API responses to the frontend.

It converts a complex `BigMoney` object into clean JSON.

Example:

## Java Object

```
BigMoney.of(CurrencyUnit.of("XAF"), 1000)
```

## Converted JSON

```
{  "amount": 1000.0,  "currency": "XAF",  "symbol": "FCFA",  "pretty": "FCFA1000.0000"}
```

Without a custom serializer, Jackson may expose unnecessary internal object details or fail to serialize properly.

---

# BigMoneyDeserializer (JSON → Java)

`BigMoneyDeserializer` is used when frontend sends JSON to backend.

It converts incoming JSON into a proper `BigMoney` Java object.

Example:

## Incoming JSON

```
{  "amount": 1000.0,  "currency": "XAF"}
```

## Converted Java Object

```
BigMoney.of(CurrencyUnit.of("XAF"), 1000)
```

This allows backend business logic to safely perform monetary calculations.

---

# Important Understanding

Serializer/Deserializer only work during:

```
API Request / API Response
```

They are triggered only when data enters or leaves the application through JSON.

---

# Why Serializer Changes Did Not Fix Database NULL Issue

The issue was not happening at the API layer.

It was happening during database persistence.

---

# Actual Flow

## 1. Request Processing

Controller/service created:

```
SalaryPaymentRequest
```

---

## 2. Invalid Flow Happened

Example:

```
No active vendor found
```

or

```
No active employee found
```

In this flow:

```
currentBalancetotalSalary
```

were never assigned.

So they remained:

```
null
```

---

## 3. Entity Saved Directly To Database

Code executed:

```
repository.save(entity)
```

Hibernate directly persisted:

```
NULL
```

into database columns.

At this stage:

```
Jackson Serializer was NEVER called
```

because no JSON conversion happened.

---

# Why Application Crashed Later

Later, dashboard/statistics API tried reading those rows.

Hibernate used:

```
MoneyUserType
```

to convert DB columns back into `BigMoney`.

But DB contained:

```
NULL currencyNULL amount
```

So Hibernate failed while reconstructing the object and threw:

```
NullPointerException
```

before API response generation.

Because of that:

```
BigMoneySerializer never got a chance to execute
```

---

# Core Lesson

|Layer|Responsibility|
|---|---|
|Serializer / Deserializer|Protect API boundary|
|Hibernate UserType|Protect database mapping|
|Service / Controller|Ensure valid entity state before save|

---

# Correct Fix

The proper fix was:

## 1. Keep Serializer/Deserializer null-safe

(for API safety)

AND

## 2. Initialize default BigMoney values before saving entity

Example:

```
.totalSalary(mfsUtils.getMoney(0)).currentBalance(currentBalance)
```

This prevents NULL values from entering the database.

---

# Final Conclusion

The root issue was not JSON serialization.

The actual problem was:

- invalid business flow
- partially initialized entity
- NULL money values persisted into DB