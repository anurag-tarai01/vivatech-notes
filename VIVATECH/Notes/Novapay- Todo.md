
# Issues to fix after MVP
1. Pending wallet should not be show in dashboard(All actor)

2. Last 10 transaction to use target currency and aggregate id

3. Report fetch, if currency wallet not exist then it throw 500 error
	- e.g. SOS not exist for subscriber, but try to get report, it will through error

## Fixes - 09-09-2026
1. **Order of Account Balance Tab** in the Super Admin page — USD, followed by other currencies. `Completed`
2. **Remittance Report Tab** to be hidden. `Completed`
3. **All Transfers Page** — Core wallet name should be correctly resolved. `Completed`
4. **Transfer Detail** — Wallet aggregate instead of wallet ID for both sender and receiver accounts. `Completed`
5. **Internal Agent Total Balance Report** — Wallet aggregate instead of wallet ID. `Completed`
6. **Internal Agent Report Filter** — Filter using wallet aggregate ID and currency instead of wallet ID. `Completed`

## Fixes
Super Admin
1. Transfer list filter using to Account and from account to support aggregate id
2. Reconciliation report - not able to get report for specific dates, report snap shot generate every day so, but why not yesterdays?
3. 












