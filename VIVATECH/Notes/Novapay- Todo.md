
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


## MOM NovaPay Multi-currency Demo Sync — 10-09-2026

**Participants:** ⁠Md Javed, Anurag Tarai  
**Overview:** The demo has been successfully completed. The following fixes and action items were discussed:

1. Subscriber cash-in (internal agent) only active wallet currency should be shown.
2. If local currency system wallets do not exist or are not set up, the FX transaction should fail immediately instead of initiating. _(Note: This will be resolved when the separate currency table for multiple foreign currencies is implemented)
3. Top 10 transaction should show currency aggregate instead of unique wallet id

### UI changes:

1. Update the terminology to use "Source Currency" and "Destination Currency" in subscriber cash in (internal agent page)
2. transfer/all page search criteria section - there is typo in from date. And its trasnfer Id not account transfer id .  
    3. **Last 10 Transactions (Mini Statements):** Update the UI to format and display amounts using exactly 2 decimal digits.

Thank you


Thanks > **Anurag Tarai**
> 
> MOM NovaPay Multi-currency Demo Sync — 09-09-2026 | 10:30–11:00 AM IST Participants: Md Javed, Anurag Tarai Focus: Super Admin Portal Following fixes were discussed: Order of Account Balance Tab in the Super Admin page — USD, followed by other currencies.Remittance Report Tab to be hidden.All…

Good Morning Md Javed sir, I have fixed these changes on the last working day. Please just let me know when you're available. We can proceed with demo of next module (Customer care portal). In the meantime, I am rebuilding the DB so we can start fresh with a clean DB.

Thanks












