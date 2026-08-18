This is one of the biggest concepts you need to understand.

Suppose external API gives:

```
USD/XAF = 600
```

Your company might not want to give customers exactly 600.

Maybe you want a 1% markup.

Then:

```
Market rate = 600
```

Customer rate:

```
600 × 0.99
= 594
```

So customer receives:

```
1 USD = 594 XAF
```

The external API's rate is therefore **not necessarily the rate used by your transaction**.

You can have:
```java
External Market Rate
        ↓
   FX Pricing
        ↓
Customer/Transaction Rate
        ↓
    Conversion
```