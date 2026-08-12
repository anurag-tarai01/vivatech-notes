
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
1. Merchant Outlet Local wallet implementation
2. Merchant Dashboard





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

