
A clean design would be:

**Input**

- `fromCurrency`
- `toCurrency`
- `sourceAmount`
- transaction/context identifier

**FX Engine**

1. Validate currency pair.
2. Find applicable exchange rate.
3. Determine rate validity/effective time.
4. Calculate destination amount.
5. Return a **rate snapshot** that the transaction can persist.

**Output**
```
sourceAmount       = 100 SOS
fromCurrency       = SOS
toCurrency         = USD
exchangeRate       = 0.0012
destinationAmount  = 0.12 USD
rateId             = FX-123
```