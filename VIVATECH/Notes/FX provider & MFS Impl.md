```java
                 ┌─────────────────┐
                 │ External FX API  │
                 └────────┬────────┘
                          │
                    USD/XAF = 600
                          │
                          ▼
┌──────────────┐    ┌──────────────┐
│ MFS Service  │───▶│  FX Engine   │
└──────────────┘    └──────────────┘
                          │
                          ▼
                    conversion
```

### Why Not Call the External API Every Time?
Problems:

- API rate limits
- latency
- provider downtime
- network failures
- inconsistent rates
- transaction unpredictability
- increased cost
- inability to reproduce historical transactions

Therefore, you normally introduce an **FX rate storage/cache layer**.

### Basic Production Architecture

```java
                External FX Provider
                         │
                         │ rates
                         ▼
                 ┌───────────────┐
                 │ FX Rate Sync  │
                 │    Service    │
                 └───────┬───────┘
                         │
                         ▼
                 ┌───────────────┐
                 │ FX Rate Store │
                 └───────┬───────┘
                         │
                         ▼
                 ┌───────────────┐
                 │   FX Engine   │
                 └───────┬───────┘
                         │
                         ▼
                    MFS Services
```

The external API becomes your **rate source**.

Your database/cache becomes your **internal source of truth for the rate used by your transaction**.