This document outlines the evaluation between Open Source (Frankfurter) and Commercial (e.g., ExchangeRate-API) providers for our FX Engine integration.

## Comparison Matrix

| **Parameter**          | **Frankfurter (Open Source)**               | **Commercial API (e.g., ExchangeRate-API)** |
| ---------------------- | ------------------------------------------- | ------------------------------------------- |
| **Cost**               | **$0 (Free / Self-hostable)**               | **~10–10–40 / month**                       |
| **Update Frequency**   | **Once Daily** (published ~16:00 CET)       | **Every 5 Minutes or 60 Seconds**           |
| **Data Source**        | Central Banks (ECB reference rates)         | Commercial Banks & FX Liquidity Feeds       |
| **Timestamp Accuracy** | Calendar Date only (`"date": "2026-08-17"`) | Exact UTC Tick (`"2026-08-17T23:59:59Z"`)   |
| **Production SLA**     | None (Community / Internal DevOps)          | 99.9% Uptime Guarantee                      |

## Frankfurter (Open Source) Analysis

### Pros for Development & QA

- **Zero Cost:** No API keys required, completely free.
- **No Rate Limits:** Ideal for automated CI/CD pipelines and high-volume local testing where rate limits would block execution.
- **Predictable Data:** ECB rates are stable, making assertion testing in lower environments straightforward.

### Cons & Risks for Production

- **Intraday Financial Loss (Arbitrage Risk):** Because rates only update once every 24 hours, market fluctuations during the day will cause the platform to execute conversions at outdated prices, creating direct treasury losses.
- **No Real-Time Auditing:** The API returns a date (`YYYY-MM-DD`), not a precise millisecond timestamp. A daily date cannot prove which exact moment a rate was locked for a disputed transaction. If a user challenges a transfer at 14:30, a date of "2026-08-17" provides no legal defense. A UTC timestamp like `2026-08-17T14:27:33Z` is mandatory for financial auditing.

## Verdict & Recommended Strategy

- **Development, QA & CI/CD:** Use **Frankfurter**. It costs $0, avoids hitting rate limits during automated testing, and supports all required base/quote currency conversions for structural testing.
- **Production:** Use a **Paid Commercial API**. A 24-hour delayed feed is a financial vulnerability for a live wallet platform. Real-time money movement requires at least 5-minute updates and precise UTC timestamps for audit defense.