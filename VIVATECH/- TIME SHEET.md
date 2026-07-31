
# July 2026
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

Extras:

> Customer Care Deposit 
> Customer Care To Subscriber
> Subscriber To Subscriber - **Completed**
> Subscriber To Customer Care
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

