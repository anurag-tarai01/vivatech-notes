1.  Spread Rate vs FX Commission — which model do we use?
2. Commission rate config — flat or percentage?
3. Zero-commission case — is it valid?
	- If `fx_commission_rate` is not configured or is zero, should the FX transfer still proceed (just with no Leg 5), or should it be blocked?
4. Should FX commission be visible to the sender before transfer?
	- Before the sender confirms the transfer, should the API response show them the breakdown: `amount: 100 USD`, `fx_commission: 1 USD`, `total_deducted: 101 USD`, `receiver_gets: 55,987 SOS`? Or is the commission silent (no preview)?


Todo:
curl: /setup/fx-commission-wallet
FX_COMMISSION_USD, setup all wallet 

TransferEventDto:
fx_commission_rate - %2

No commissionInfo will generate

To withdraw from fx commission wallets: 
/withdraw-fx-commission


I have discussion with my senior and the following were point to follow:
1. We should remove spread, we actually don't need that and should only use rate as field and inverse rate. So we don't need effective rate, spread rate and market rate
2. /local-currency-wallets-so like this we have to create an new endpoint to create new separate wallets for  
	- /setup/fx-commission-wallet
	- FX_COMMISSION_USD, setup all currency in the system. And for the scinario we have disscuss, when add a new currency to system in that case, this wallet is not created for this currency. we have to discuss
3. TransferEventDto: fx_commission_rate - %2, it will config from admin side and fetch from the db admin config and no commission info will be saved, actully we don't need to save commission, as fx charges are system commission and no other user involved in it, we can track this from reports or wallet history


# To check
private String targetCurrency; - it have no use why added to TransferEventDto

in finish transaction  logic
if (dto.getFxRateSnapshotId() == null) { commissionInfoListener.saveCommissionData(dto);}
For example if its fx transfer but the service charges are deduct for other reason, for example for subscriber cash out, in that case this block will skipped



## Spread Rate vs FX Commission

### The Money Changer Analogy

Imagine a physical money changer on the street.

| Concept           | Money Changer Analogy                                                                                                                                                      | In Your System                                                                                          |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **Market Rate**   | The real USD/SOS rate from the news (`1 USD = 571.50 SOS`)                                                                                                                 | `ExchangeRate.currentRate` — fetched every 5 min                                                        |
| **Spread Rate**   | The money changer gives you a slightly worse rate to protect against rate movement (`1 USD = 559.87 SOS`). You never see this as a separate fee — it's baked into the rate | `fx_spread_rate = 2%` → `effectiveRate = marketRate × (1 - spread)`. Stored in `ExchangeRateSnapshot`   |
| **FX Commission** | The money changer also charges a visible `$1 service fee` on top, printed on the receipt                                                                                   | `fx_commission_rate = 1%` → `1 USD` deducted from sender, credited to AMT03. Stored as `CommissionInfo` |

---

### Why They Are Different (Key Technical Distinction)

||Spread Rate|FX Commission|
|---|---|---|
|**What it changes**|The exchange rate the receiver gets|Nothing about the rate — it's a separate fee|
|**Who pays**|The receiver (gets fewer SOS)|The sender (pays more USD)|
|**Denominated in**|Neither — it's a rate ratio (`%`)|Source currency (USD, SOS, etc.)|
|**Visible on receipt?**|No — it's implicit in the rate|Yes — explicit line item|
|**Stored where**|`exchange_rate_snapshot.spread_rate`|`commission_info` table|
|**Goes to**|Nobody — it adjusts the rate the platform quotes|AMT03 wallet (platform revenue)|
|**Purpose**|Protect platform from rate volatility in the 5-min polling window|Platform revenue / service charge|

---

### Simple Example to Show Your Senior

---

### Simple Example to Show Your Senior

**Transfer: 100 USD → SOS. Market rate: 1 USD = 571.50 SOS. Spread: 2%. Commission: 1%.**

Step 1 — Apply Spread (adjusts the rate):

  Effective Rate = 571.50 × (1 - 0.02) = 559.87 SOS per USD

  Receiver gets = 100 × 559.87 = 55,987 SOS  (not 57,150)

  ↑ The "missing" 1,163 SOS is the platform's rate margin

Step 2 — Apply FX Commission (separate fee from sender):

  Commission = 100 USD × 1% = 1 USD

  Sender is debited = 100 + 1 = 101 USD total

  AMT03 receives = 1 USD

**So the platform earns two things:**

1. **Spread margin** — embedded in the rate (receiver gets fewer SOS)
2. **FX Commission** — explicit deduction from sender (goes to AMT03 as revenue)

---

### When Talking to Your Senior, Use This Line

> _"The spread is a rate adjustment — it doesn't show up as a line item, it just means the receiver gets fewer units. The FX commission is an explicit service charge in the sender's currency that goes into AMT03, exactly like a cash-out fee goes into AMT03 for a normal transfer. Both protect the platform, but from different angles: spread protects against rate risk, commission is our explicit service revenue."_

