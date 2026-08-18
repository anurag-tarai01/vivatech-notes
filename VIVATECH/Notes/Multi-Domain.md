## Operational Runbook: Adding a New Domain
	Once the 4 refactoring steps above are done, this is the exact sequence:

```
Step 1: POST /geography/create-domain
        { name, shortName, localCurrency, foreignCurrency, isPrimary: false }

Step 2: POST /geography/approve-create-domain
        { id: <domainId> }
        → Domain status becomes ACTIVE

Step 3: POST /geography/create-zone  (existing API)
Step 4: POST /geography/create-area  (existing API)
        → Build the geographic hierarchy for the new domain

Step 5: GET /setup/setup-domain-wallets?domainShortName=<shortName>
        → Provisions AMT01_<currency>, AMT06_<currency>, RESELLER_<currency>,
          AMAL_EXPRESS_COMMISION_<currency> system wallets

Step 6: (Optional) POST /geography/set-primary-domain  { domainId: <id> }
        → Only if this new domain should become the new default context
```

---

## Summary of Code Changes

| File                            | Change                                                        | Effort  |
| ------------------------------- | ------------------------------------------------------------- | ------- |
| `Domain.java` (geography-query) | Add `isPrimary` boolean field                                 | Trivial |
| DB migration script             | Add `is_primary` column, mark existing domain as primary      | Trivial |
| `GlobalContextInitializer.java` | Load from `isPrimary` domain instead of `get(0)`              | Trivial |
| `DomainDto.java`                | Add `isPrimary` field                                         | Trivial |
| `DomainEventDto.java`           | Add `isPrimary` field                                         | Trivial |
| `SetupController.java`          | Parameterize `setupLocalCurrencyWallets` by domain short name | Small   |
| `DomainController.java`         | New `set-primary-domain` endpoint                             | Small   |

**Estimated effort: 1–2 days total.** No existing flows are broken.

---

IMPORTANT

The existing primary domain (`SO`) must be manually set `is_primary = true` in the DB **before** deploying this change. Otherwise the fallback `get(0)` kicks in and behavior is unchanged — safe but not deterministic.

NOTE

`ConfigConstants.CURRENCY_CODE` and `COUNTRY_CODE` continue to represent the **primary domain** only. Secondary domain users will inherit primary domain defaults for phone formatting and default currency. Full per-user domain-scoped context is a separate, larger workstream (full multi-currency migration).