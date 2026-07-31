[[Technical Terms]] 
[[Login Details]] 
[[Questions]] 
## Whole Project Structure
1. [[M-CASH — Complete High-Level System Guide]]
2. [[Project Setup]]
3. [[Swagger]]
## R&D
1. [[Transfer Type For Commission]] where agents are involved
2. [[Maker-Checker Flow Switch Wallet Deposit & Implement Same For Disbursement]] 
3. [[📄Thread Starvation & RabbitMQ Timeouts]]  
4. [[New Salary Payment]] [Transfer Controller]
5. GPay Account Transfer Service -> Event & RabbitMQ -> Next what trigger
6. TransferEvenListner
7. [[Switch Wallet Validation]] 
8. [[Switch Validation External API]]
9. [[Commission Disbursement]]

## Refactor & Code Explain
1. [[Salary Type Into Enum]]
## Doubt
1. [[Queries]] 
2. [[Maker Checker In Commission Disbursement]]
3. [[Third Party Dashboard Clarifications]] 
## Meeting
1. [[22-05-2026 Friday (Client Meeting - with Claude)]] 
2. [[12-07-2026-ClientMeeting]] 
## Tasks
1. [[Third Party or Bulk Payment Changes]] 
## Concepts
1. [[Mapping with Orika]] 
2. Premature Abstraction
3. Saga Design

## Users & Operations
1. [[Actors]]
2. [[Internal Agent & External Agent]]
3. [[Operations Per Actor]]
4. [[Flow For Each Operation Type]]
5. [[Permission Relationships]] 
6. [[Where Things Are In Code]] 
## Axon & Flow
1. [[Axon]] 
2. [[Flow of Event Driven API]] 
3. [[Core Concepts]] 
4. [[Projection]] 
## Transactions Flow
0. [[Overall Money Flow Architecture]] 
1. [[Subscriber Cash In]] 
2. [[Subscriber Cash Out]] 
3. [[Switch Account]] 
4. Switch Account Money
## Backend-Notes
1. [[BigMoneySerializer & BigMoneyDeserializer]] 
## [[Git M-CASH]] 
1. [[E-payroll-issue]] 
2. [[Check Git User]] 
3. [[Git Log]] 
4. [[Removing all the local changes and making local as it is as in remote]]
5. [[Switch branch with uncommit branch]] 
6. [[Why not use same branch for Multiple PRs]] 

## SQL Server 
1. [[SQL Server Restart]]

git checkout -b feature/profile-implementation-wallet

Dummy Data
[[User Data]] 

**Cash in** (Cash in should be placed under the cash out button than placed up)

If not mistaken, the requirement is to place the Cash-In option under Cash-Out in the left menu, currently it shows on the agent profile, will be taken care of.

