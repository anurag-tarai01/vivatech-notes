#### 1. Provider Comparison Matrix

| **Provider**                         | **Recommended Plan** | **Price / Month** | **Update Interval** | **Monthly Request Limit** | **Key Pros & Cons**                                                                                                                                                                                                      |
| ------------------------------------ | -------------------- | ----------------- | ------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **ExchangeRate-API** _(Recommended)_ | **Pro Plan**         | **$10.00**        | **5 Minutes**       | 100,000 requests          | **Pros:** Extremely high historical uptime (99.99%), low cost, perfectly aligns with our 5-minute polling architecture.  <br>  <br>**Cons:** 5-minute delay (not 60-second, though irrelevant for our polling interval). |
| **CurrencyAPI**                      | **Medium Plan**      | **$39.99**        | **60 Seconds**      | 600,000 requests          | **Pros:** 60-sec refresh, 10+ decimal precision, standard base-switching.  <br>  <br>**Cons:** Higher cost. Overpowered unless we increase our polling frequency to every 60 seconds.                                    |
| **Open Exchange Rates**              | **Enterprise**       | **$47.00+**       | **5 Minutes**       | 200,000 requests          | **Pros:** Industry-standard JSON format.  <br>  <br>**Cons:** Free/low plans restrict base currency to USD only. Most expensive option for 5-minute updates.                                                             |

#### 2. Selected Recommendation: ExchangeRate-API (Pro Plan)

**Why ExchangeRate-API is the Best Fit:**

1. **Architectural Alignment:** Our FX Engine design utilizes a scheduled Quartz job to poll for rates every 5 minutes (saving API calls and internal DB writes). ExchangeRate-API's 5-minute update interval perfectly matches our internal polling frequency. Paying $30 more for CurrencyAPI's 60-second updates yields no benefit if we only poll every 5 minutes.
    
2. **Payload Structure & Auditing:** Returns high-precision rates alongside immutable UTC timestamps (e.g., `time_last_update_utc`), which is critical for parsing into Java `BigDecimal` and maintaining exact audit trails for every transaction.
    
3. **Usage Economics:** With our scheduled 5-minute polling strategy (1 call/5 minutes = 8,640 calls/month), we consume less than **8.6%** of the 100,000 request limit, giving us massive headroom for scaling without needing to upgrade tiers.
    
4. **Testing Strategy:** ExchangeRate-API offers a free tier (250 req/mo) which is sufficient for basic manual sandbox verification. For local development and automated CI pipelines, we will use the free open-source Frankfurter API to ensure zero quota consumption.
    

#### 3. Budget Request & Decision Summary

- **Proposed Vendor:** ExchangeRate-API
    
- **Plan:** Pro Tier
    
- **Monthly Commitment:** **$10.00 / month**
    
- **Action Required:** Approval to acquire API keys for development integration.