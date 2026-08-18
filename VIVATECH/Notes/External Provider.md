# Technical Evaluation & Proposal: FX Rate Provider Integration for MFS Platform

**Scope:** Task 1.3 — Multi-Currency FX Engine Architecture

## 1. Executive Summary

As part of the Mobile Financial Services (MFS) Multi-Currency and Multi-Domain migration, the platform requires an accurate, reliable, and compliant Foreign Exchange (FX) rate feed to enable cross-currency wallet transactions (e.g., USD $\rightarrow$ SOS, INR, BDT).

### Key Proposal Takeaway

We propose a **Two-Tier Hybrid Strategy**:

1. **Development & Staging Environments:** Utilize **Frankfurter** (Open-Source / Self-Hosted Docker container) at **$0 cost** to avoid consuming API quotas during development and automated CI/CD testing.
    
2. **Production Environment:** Integrate a commercial provider (**CurrencyAPI Medium Tier** as primary candidate, with **Open Exchange Rates** as alternate) utilizing a **60-second asynchronous polling cache**.
    

_Total Estimated Production Cost:_ **~$40 – $50 / month**, requiring zero per-transaction latency overhead.

## 2. Evaluation Criteria for Financial Compliance

To qualify for an MFS/Fintech ecosystem, providers were evaluated against six mandatory operational pillars:

1. **Update Frequency:** Must provide near-real-time rates (at least every 60 seconds) to mitigate treasury currency fluctuation risk.
    
2. **Uptime & SLA:** Minimum 99.9% uptime guarantee with fault-tolerant global edge endpoints.
    
3. **Data Integrity & Traceability:** Immutable UTC timestamps for every rate tick to support financial audits and regulatory reconciliation.
    
4. **Currency Coverage:** Broad coverage for global fiat currencies (USD, EUR, GBP, INR, BDT, and regional African currencies such as SOS).
    
5. **Cost-to-Volume Efficiency:** Predictable flat-rate pricing that does not scale with end-user transaction volume.
    
6. **Developer Integration:** Clean REST/JSON APIs with standard HTTP headers for authentication and rate-limit tracking.
    

## 3. Provider Landscape Analysis

### Category A: Open-Source & Self-Hostable (Free)

| **Provider**                 | **Data Source**             | **Update Frequency** | **Pros**                                                                                          | **Cons**                                                                 | **Best Use Case**                                  |
| ---------------------------- | --------------------------- | -------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------- |
| **Frankfurter** _(Top Pick)_ | European Central Bank (ECB) | Daily (16:00 CET)    | • 100% Open source<br><br>  <br><br>• Zero API keys<br><br>  <br><br>• Dockerized / Self-hostable | • Daily updates only<br><br>  <br><br>• 32 major fiat currencies only    | Local Dev, CI/CD pipelines, Unit/Integration tests |
| **Fawaz Ahmed Currency API** | Central Banks via Open Repo | Daily                | • 200+ currencies<br><br>  <br><br>• CDN-backed (jsDelivr)<br><br>  <br><br>• Zero setup          | • Community maintained<br><br>  <br><br>• No SLA or commercial guarantee | Backup fallback in local test environments         |
| **VATcomply**                | ECB + Geolocation           | Daily                | • Open-source Python backend<br><br>  <br><br>• Docker compose ready                              | • Daily rates only<br><br>  <br><br>• Extra bloat (VAT/IP tools)         | Staging environments requiring IP metadata         |

### Category B: Commercial SaaS Providers (Mid-Tier Fintech)

| **Provider**                    | **Update Frequency**               | **Currency Count** | **Starting Cost**             | **Pros**                                                                                                                                   | **Cons**                                                                 |
| ------------------------------- | ---------------------------------- | ------------------ | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------ |
| **CurrencyAPI** _(Recommended)_ | **60 Seconds** (on Medium tier)    | 170+ Fiat & Crypto | **$39.99 / mo** (600k req/mo) | • 60-sec update window<br><br>  <br><br>• Sandbox API keys<br><br>  <br><br>• Base currency switching<br><br>  <br><br>• Header-based auth | • Monthly quotas apply on lower tiers                                    |
| **ExchangeRate-API**            | 5 Min (Pro) / Hourly               | 165+ Currencies    | **$10 – $30 / mo**            | • 15+ years track record<br><br>  <br><br>• Extremely stable JSON<br><br>  <br><br>• 99.99% AWS uptime                                     | • 60-sec real-time requires higher custom tier                           |
| **Open Exchange Rates**         | 60 Min (Free) / 5 Min (Enterprise) | 200+ Currencies    | **$12 – $47 / mo**            | • Industry standard format<br><br>  <br><br>• Comprehensive time-series data                                                               | • Free/low tiers lock base currency to USD only                          |
| **Fixer.io (APILayer)**         | 60 Seconds (Paid)                  | 170 Currencies     | **$10 – $40 / mo**            | • Backed by APILayer<br><br>  <br><br>• Standardized JSON format                                                                           | • Strict rate limiting<br><br>  <br><br>• Historical data is plan-locked |

### Category C: Tier-1 Enterprise & Institutional Providers

|**Provider**|**Data Source**|**Target Market**|**Estimated Cost**|**Evaluation Verdict**|
|---|---|---|---|---|
|**XE Currency Data API**|100+ Global Bank Feeds|Global Enterprises, SAP/Oracle ERPs|**$799 – $4,499+ / yr**|**Overkill for Phase 1.** High upfront annual commitment. Recommended once daily FX volume exceeds $500k.|
|**OANDA Rates API**|Market Maker Order Books|Hedge funds, Forex Brokers, Multinationals|**Custom Enterprise Quotes**|**Out of Scope.** Provides deep bid/ask liquidity data, but unnecessary for standard wallet transfer conversions.|

## 4. Comprehensive Comparison Matrix

|**Evaluation Parameter**|**Frankfurter (Open Source)**|**CurrencyAPI (Commercial SaaS)**|**XE Currency Data (Enterprise)**|
|---|---|---|---|
|**License / Type**|Open Source (MIT)|Commercial SaaS|Commercial Enterprise|
|**Monthly Cost**|**$0.00**|**$39.99 / month**|**~$375+ / month ($4.5k/yr)**|
|**Update Interval**|Once Daily|**Every 60 Seconds**|Real-time / Sub-minute|
|**Host Environment**|Internal Docker (AWS ECS)|Managed Third-Party SaaS|Managed Third-Party SaaS|
|**SLA Guarantee**|Internal Team Responsibility|99.9% Uptime SLA|99.99% Enterprise SLA with Dedicated Support|
|**Audit Compliance**|Reference data only|Full UTC Timestamps|Central Bank Regulatory Certified|
|**Recommended Role**|**Dev, QA, CI/CD Testbeds**|**Production Core Feed**|**Long-term Phase 2 Scale**|

## 5. Architectural Strategy: The Asynchronous Polling Engine

To ensure that the MFS platform remains decoupled from external provider latency, rate limits, and third-party outages, we will implement an **Asynchronous Local Ingestion Architecture**.

```
                           SCHEDULED WORKER
                     (Runs every 60s in background)
                                  │
                                  ▼
                   ┌──────────────────────────────┐
                   │  CurrencyAPI / Frankfurter   │
                   │      (GET /v3/latest)        │
                   └──────────────┬───────────────┘
                                  │
                                  ▼
                   ┌──────────────────────────────┐
                   │   MFS Local Rate Database    │
                   │   (ExchangeRate Table)       │
                   └──────────────┬───────────────┘
                                  │
                      Sub-millisecond Local Read
                                  │
 ┌────────────────────────────────┴────────────────────────────────┐
 │                     MFS TRANSFER ENGINE                         │
 │                                                                 │
 │  1. Fetches latest rate snapshot from local DB                  │
 │  2. Applies configurable business markup (e.g. +1.5%)           │
 │  3. Performs Banker's Rounding (HALF_EVEN)                      │
 │  4. Executes Atomic Wallet Movement (Source Debit/Target Credit)│
 └─────────────────────────────────────────────────────────────────┘
```

### Why this protects our business:

1. **Zero User Latency:** Cross-currency transfers execute against our local database in $< 2\text{ ms}$, rather than waiting $200\text{–}500\text{ ms}$ for an external HTTP round-trip.
    
2. **Deterministic Cost:** Regardless of whether our users perform 100 or 1,000,000 transfers per month, MFS makes exactly:
    
    $$\frac{1 \text{ request}}{\text{minute}} \times 60 \text{ minutes} \times 24 \text{ hours} \times 30 \text{ days} = \mathbf{43,200 \text{ \textbf{calls/month}}}$$
    
    _This uses less than 8% of the CurrencyAPI Medium Plan quota (600,000 requests/month)._
    
3. **Outage Resilience:** If the third-party provider experiences downtime, the MFS transfer engine continues operating using the local cache until the configured stale-rate threshold (e.g., 5 minutes) is reached.
    

## 6. Financial Risk & Fail-Safe Controls

|**Risk**|**Consequence**|**Engineering Control / Mitigation**|
|---|---|---|
|**Provider Downtime**|Transfers fail or block threads|**Circuit Breaker Pattern:** Fallback to last known local rate with a maximum 5-minute freshness window.|
|**Stale Rate Market Volatility**|FX losses due to sudden market shifts|**Rate Freshness Validator:** If local rate timestamp is $> 5\text{ minutes}$ old, reject FX transfers with `FX_RATE_STALE` error until resolved.|
|**Floating-Point Drift**|Financial discrepancies / leaky ledger|**Zero Floats Rule:** All rate mathematics enforced via `BigDecimal` and `BigMoney` using `RoundingMode.HALF_EVEN`.|
|**Provider Vendor Lock-in**|Cost escalation or policy changes|**Interface-Driven Design:** The domain logic interacts with an abstract `FxRateProviderClient` interface, allowing provider swap via configuration in under 1 hour.|

## 7. Recommended Implementation Timeline & Next Steps

- **Week 1:** Implement local `Frankfurter` Docker container; develop the internal `ExchangeRate` schema, caching engine, and unit test suites.
    
- **Week 2:** Subscribe to the `CurrencyAPI` Medium sandbox tier; implement `CurrencyApiProviderClient` with header security and circuit breaking.
    
- **Week 3:** Connect FX Engine snapshot logic to Task 1.4 (Atomic Transfer & Wallet Movement).
    

### Action Requested from Management

- **Approval of the Hybrid Architecture** (Open-Source for Dev / SaaS for Production).
    
- **Budget Approval** for **CurrencyAPI Medium Plan** ($39.99/month).