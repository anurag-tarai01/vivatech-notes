[[PREVIOUS Time sheet]]
# July
## 23-07-2026
1. Refactoring subscriber Cash-In(Customer care to subscriber) Transfer Flows to Support Multi-Currency Wallet Resolution
	- `TransferRequestBodyAdvice` fix to resolve wallets before request hit controller, only support TransferDto -// Yesterday worked
	- /subscriber-cash-in refactor -// Yesterday worked
	- Webclient, Reporting & Notification Microservices refactored to support new design
2. Refactoring  P2P Transfer Flows to Support Multi-Currency Wallet Resolution
	- /p2p-transfer (Started)
## 24-07-2026
1. Refactoring  P2P Transfer Flows to Support Multi-Currency Wallet Resolution
	- /p2p-transfer (Completed)
	- Reporting & notification refactor(Working on & facing some issues)
2. Refactored CachedWallet to store wallet aggregateId(account no.)
	- WalletListener
	- Updated Cache setup controller
## 27-07-2026
1. Report not showing p2p transfer, but stored in DB correctly (Fixed)
2. Raised PR for all changed until now
3. Customer Care (New Wallet Design for multi currency)
	- New wallet Design follow during customer care registration
	- Local Currency Wallet creation for Internal Agent 

## 28-07-2026
1. **Customer Care**
    -  local currency wallet creation for Internal Agents.(**Completed**)
2. **External Agents**
    -  refactoring the default wallet flow to the new multi-currency wallet design.(**Completed**)
3. **Merchant**
    - refactoring the default wallet flow to the new multi-currency wallet design.(**Completed**)
## 29-07-2026
1. **External Agents** (Distributor Agent, Resale Agent, Agent)
    - New local currency wallet creation (**Completed**)
2. **Merchant**
	- New local currency wallet creation. (**In progress**)
3. **OUTLET**
	- Default Wallet (**Completed**)
## 30-07-2026
1. **Merchant**
	- New local currency wallet creation. (**Completed**)
2. **Biller**
	- Default Wallet to follow new design(**Completed**)
	- New local currency wallet creation. (**Completed**)
3. **Third Party**
	- Default Wallet to follow new design (**Completed**)
	- New local currency wallet creation. (**Pending**)

## 31-07-2026
1.  **Third Party**
	- New local currency wallet creation.
2. DB rebuild & Recreate all the users and test all wallets created correctly
3. Float Generation(Account Transfer)
	- Resolve the wallets  of source and destination by new wallet design
	- User can select destination wallet currency, currencies fetched from destination wallets by aggregate id. BigMoney is created using that currency. e.g. USD, XAF or SOS and attached to TransferDto.
Now a single user hold multiple wallets. Before cross currency R&D. We have refactor all transaction one by  one. User can select currency wallet of destination wallet. Same currency transaction should work seamlessly SOS->SOS, USD->USD. Should check every single 

4. Float generation
5. Internal Agent Deposit
6. Subscriber Cash in
7. All other transaction

# August
## 03-08-2026
1. Float generation
	- Global & Local Currency Master Wallet
	- Reporting refactored to support multi-currency report generation and filter
	- Fixed reporting page to filter reports of Master Wallet by currency
2. Customer care deposit
	- Global USD transfer
	
## 04-08-2026
1. Customer Care Deposit (Super Admin Portal)
	- Provide drop down to select currency of destination wallet
	- Customer Care Deposit with multiple currency support
	- Report generation & Notification
## 05-08-2026
1. transfer/all (Super Admin Portal)
	- Issue fixed
2. Customer Care Total Balance report (Super Admin Portal)
	- Issue fixed
3. Customer Care Portal
	- Dashboard, Reports by currency, and Statements to support multi-currency wallets
4. Retesting & Reconciliation
	- Reconciled new wallet creation & deposit for customer care

**Now**:
- Float generation(Master Wallet Deposit)
- Customer care deposit
Flows migrated to new wallet design. Reporting, Reconciliation, & Notification working correctly.

**Next to work on**: Subscriber Cash In(Customer care to Subscriber)

## 06-08-2026
1. Customer care to subscriber(Subscriber cash in)
2. Both global and local currency transfer
3. UI changes
4. Report generation & Notification
## 07-08-2026
1. P2P Account Transfer
2. Both global and local currency transfer(USD->USD & SOS->SOS, .....)
3. Subscriber dashboard and reporting page to support multi currency
4. Code Push Code
## 10-08-2026
1. Latest PR code review with Javed Sir 
2. Subscriber Cash Out to Internal Agent
3. Subscriber Cash Out to External Agents
    - Account Transfer 
    - Report Generation & Fetch
## 11-08-2026
1. Subscriber Cash Out to External Agents
	- Notification 
2. External Agent Portal
	- Dashboard- Multiple wallet card
	- Agent to Subscriber Transfer
	- Agent to Resale Agent Transfer
	- Resale To Distributor Agent Transfer
	- Mini statements
	- Top 10 transactions
## 12-08-2026
1. Merchant Dashboard
2. Outlet Local Currency wallet implementation
## 13-08-2026
1. Subscriber to outlets(Merchant) Transactions
	- Single outlet 
	- Multiple outlet
	- Report generation
	- Reconciliation
## 14-08-2026
1. Merchant & Outlet transaction Reports issue fixes
2. Test Outlet Transaction and Reconciliation
3. Remove dependency of currency list from properties & fetch from active domains(webclient)
4. R&D implement new domain to the system
## 17-08-2026
1. Foreign Exchange(Forex/FX) R&D
	 - Understanding what is FX & How its calculated
	 - External API to get FX rate
## 18-08-2026
1. Foreign Exchange(Forex/FX) R&D
2. Document for evaluation of external FX provider(Open source or Paid)
3. Discussion with Javed Sir on FX Provider & Cross currency Account Transfer schema changes
4. R&D on Cross Currency Account Transfer schema changes, and Transfer Flow
## 19-08-2026
1. R&D on Implementation of Cross Currency Transfer
2. Prepare document of implementation and reviewed by Javed Sir
3. Start Implementation - Entities for FX service
## 20-08-2026
1. FX Engine set up - Entities, DTOs, Repository
2. Feign Client set up for calling external FX provider
3. Service Layer 
	- To Fetch The Market Rates & 
	- To convert the amount and generate snapshots
## 21-08-2026
1. FX Engine Implementation (**Completed**)
2. Testing FX Engine
3. Refactoring P2P Transfer for cross currency(**Started**)
## 24-08-2026
1. Atomic Bridge Orchestration (4-Legged Logic)
2. P2P FX transfer refactorization for FX transfer
## 25-08-2026
1. Subscriber mini statements refactored for FX transfer
2. Reporting Refactorization for FX transfer
	- refactored customer transaction report to store correct FX metadata
	- Fixed currency mismatch issue during report retrieve
## 27-08-2026
1. Notification microservice refactorization for FX transfer 
2. Refactored wallet update in wallet service to publish message to RabbitMQ for MoneySubtractedEvent & MoneyAddedEvent, for all 4 legs. So, reporting to generate wallet history.
3. Rebuild DB, And Perform P2P transactions, reconciliated reports
## 28-08-2026
1. Customer-care deposit refactor for FX transfer 
2. Customer care to Subscriber transfer refactor for FX transfer
## 31-08-2026

Refactored The following transfer for FX (cross currency) transaction
1. Subscriber to External Agent(Cash out)
2. External Agent to Subscriber(Cash In)
3. Agent to Resale-agent (cash out)
4. Resale agent to Dist agent (cash out)

# September

## 1-09-2026
External Agent Cash in
1. Dist. agent to RA (cash in)
2. RA to Agent (cash in)
3. Customer care to External Agent
4. Agent cash in(customer care to agent)

## 2-09-2026
1. Merchant cash in(customer care to merchant)
2. Outlet payment(Subscriber to outlet)
3. PR raised for all microservices

## 3-09-2026
1. Reporting Microservice
	- Refactoring of the **Jasper reporting** flow to support **FX transfers across all major actors**
	- Super admin, Subscriber, Internal Agent, External Agents
## 4-09-2026
1. Showing currency aggregate instead of unique wallet uulid,  both in all transfer page and report page
2. Showing transfer view page FX amount details(sent, received amount)
3. Fixing some bugs
	- Previous balance currency wrong showing
	- Wallet history not generating for 4th leg for some transaction
## 7-09-2026
1. Merchant and outlet transaction report download
2. Rebuild DB & started testing whole flow and 
	-  Done some fixes during that

## 8-09-2026
1. Refactored customer care deposit as recommended by Javed Sir.
2. Continued testing of the whole flow of the multi-currency implementation.
3. Fixed minor bugs there as well.

## 9-09-2026

I gave demo of Noavapay mulit currency to javed sir, it was mainly focused on super admin portal. There are 6 improvement and fixes listed out, which I haved implemented last working day.
1. NovaPay Multi-currency Demo Sync — 09-09-2026 | 10:30–11:00 AM IST with javed sir
**Focus:** Super Admin Portal
2. **Order of Account Balance Tab** in the Super Admin page — USD, followed by other currencies.
3. **Remittance Report Tab** to be hidden.
4. **All Transfers Page** — Core wallet name should be correctly resolved.
5. **Transfer Detail** — Wallet aggregate instead of wallet ID for both sender and receiver accounts.
6. **Internal Agent Total Balance Report** — Wallet aggregate instead of wallet ID.
7. **Internal Agent Report Filter** — Filter using wallet aggregate ID and currency instead of wallet ID.
## 10-09-2026
 MOM NovaPay Multi-currency Demo Sync — 10-09-2026

**Participants:** ⁠Md Javed, Anurag Tarai  
**Overview:** The demo has been successfully completed. The following fixes and action items were discussed:

1. Subscriber cash-in (internal agent) only active wallet currency should be shown. `Done`
2. If local currency system wallets do not exist or are not set up, the FX transaction should fail immediately instead of initiating. _(Note: This will be resolved when the separate currency table for multiple foreign currencies is implemented)
3. Top 10 transaction should show currency aggregate instead of unique wallet id `Done`

### UI changes:

1. Update the terminology to use "Source Currency" and "Destination Currency" in subscriber cash in (internal agent page) `Done`
2. transfer/all page search criteria section - there is typo in from date. And its trasnfer Id not account transfer id .  `Done`

3. **Last 10 Transactions (Mini Statements):** Update the UI to format and display amounts using exactly 2 decimal digits.

## 11-09-2026
1. All feedback refactorization completed
2. R&D on how to implement FX service charges, few points I need to discuss with javed sir before start implementation

## 14-09-2026
1. FX_COMMISSION admin wallet setup
2. Implemented Foreign Exchange commission logic

tomorrow:
Rebuild the db -> fx transfer without wallet setup & config also -> fx transfer without walletsetup -> do an subscriber cash-in test with service charge setup both non-fx, fx transfer -> setup the wallet -> perform the transfer again!

1. Testing of fx commission
2. Withdraw fx amount? - Should we implement it now or next? today is 15, today I will also working the following and also fix anything important bugs if found. And on 16 we will merge the code.
3. New domain to system.

>Float Generation - **Completed**
>Customer Care Deposit - **Completed**
> Customer Care To Subscriber - **Completed**
> Subscriber To Subscriber - **Completed**
> Subscriber To Customer Care - **P2P** - **Completed**
> Customer Care To Agent
> Agent to Customer Care

enum class WalletType {  
    SUBSCRIBER, `Done`
    CUSTOMER_CARE, `Done`
    AGENT, `Done`
    MERCHANT, `Done`
    OUTLET, `Not now`
    BILLER,`Done`
    THIRD_PARTY, `Done`
    
    ADMIN, 
    AMAL_EXPRESS, 
    
    INTERNAL_AGENT_COMMISSION // Legacy
    AGENT_COMMISION, // Legacy
    ADMIN_COMMISSION  // Legacy
}

