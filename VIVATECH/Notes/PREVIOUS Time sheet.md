25-june-2026
|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|New Module KT from Javed Sir: Novapay Multi Domain|Development|09:00|10:00|01 Hrs 00 Min|||
|2|Novapay Project Set UP Backend & Reporting. For Multi Domain|Development|10:00|13:00|03 Hrs 00 Min|||
|3|Preparing Clear Document For Commission Disbursement Flow|Development|15:00|18:00|03 Hrs 00 Min|||
|4|Preparing and Going Through Queries for Client Meeting at 3 p.m IST|Development|14:00|15:00|01 Hrs 00 Min|||

29-june
|Task ID|Task Name|Category|Start Time|End Time|Duration|Status|Action|
|---|---|---|---|---|---|---|---|
|1|List of Switch APIs and preparing for client meeting, for queries|Development|09:00|10:00|01 Hrs 00 Min|||
|2|Setting Up Novapay Microservices for Multi domain & Multi Currency implementation|Development|10:00|13:00|03 Hrs 00 Min|||
|3|Setting Up Nova Pay Project|Development|14:00|15:00|01 Hrs 00 Min|||
|4|Client Meeting|Development|15:00|16:00|01 Hrs 00 Min|||
|5|Geography Set Up FirstSetUp in Set UP controller - Foreign & Local Currency|Development|16:00|18:00|02 Hrs 00 Min|||

30-june
|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Admin & Wallet Set Up For Multi Domain|Development|09:00|11:00|02 Hrs 00 Min|||
|2|Nova Pay Multi Domain: Removing Dependency from properties files|Development|11:00|13:00|02 Hrs 00 Min|||
|3|R&D for making Novapay into single currency global USD as first step for Multi Domain|Development|14:00|18:00|04 Hrs 00 Min|||

1-jul
|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Nova pay multi domain: Migrating to Single Global Currency USD, from default domain in db|Development|09:00|13:00|04 Hrs 00 Min|||
|2|Nova pay multi domain: Migrating to Single Global Currency USD, from default domain in db|

2-jul
|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Novapay Multi Domain: Single global currency USD, fetching domain context from db default domain. Testing again with rebuilding DB.|Development|09:00|10:00|01 Hrs 00 Min|||
|2|Review & Push the code: backend, webclient & api-gateway|Development|10:00|11:00|01 Hrs 00 Min|||
|3|Setting a lightweight DB, alternative of SSMS|Development|11:00|12:00|01 Hrs 00 Min|||
|4|Started Backend Refactor remove every single dependency of currency and country code from properties.|Development|12:00|13:00|01 Hrs 00 Min|||
|5|Noava Pay: Replaced mfs.country-code and mfs.currency-code @Value field injections across the codebase with centralized ConfigConstants.COUNTRY_CODE and ConfigConstants.CURRENCY_CODE usages.|Development|14:00|17:00|03 Hrs 00 Min|||
|6|Testing rebuilding DB again, added internal agent and perform Account Transfer|

3-jul
|Task ID|Task Name|Category|Start Time|End Time|Duration|Status|Action|
|---|---|---|---|---|---|---|---|
|1|Refactoring mfs-webclient, removing dependency of currency and country code from config|Development|09:00|10:00|01 Hrs 00 Min|||
|2|Meeting with Javed Sir|Development|10:00|10:15|00 Hrs 15 Min|||
|3|Creating Agent & Other Users and Checking, doing some transaction. Re-buidling db again|Development|10:15|13:00|02 Hrs 45 Min|||
|4|Working On Reporting, removing dependency from config for currency and country code. Use Postconstruct to load the domain from backend.|Development|14:00|16:00|02 Hrs 00 Min|||
|5|Notification, removing migrate code as well. Use ConfigConstant shared from bakend.|Development|16:00|17:00|01 Hrs 00 Min|||
|6|Reconstruct DB again, and done few transactions.|

4-jul
|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Customer Care Deposit (Super Admin Portal): Provide drop down to select currency of destination wallet.|Development|09:00|12:00|03 Hrs 00 Min|||
|2|Customer Care Deposit with multiple currency support|Development|12:00|13:00|01 Hrs 00 Min|||
|3|Report generation & Notification|Development|14:00|16:00|02 Hrs 00 Min|||
|4|Reconciliation & Testing: Customer Care Deposit (Super Admin Portal) multi currency support|Development|16:00|18:00|02 Hrs 00 Min|||

6-jul
|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Multi Domain Single Global Currency Testing All Transactions|Development|09:00|11:00|02 Hrs 00 Min|||
|2|Setup the M-CASH Again and fixes for deployment|Development|11:00|13:00|02 Hrs 00 Min|||
|3|Nova Pay Single Global Currency Testing All Transactions & Reconsolidate|

7-jul
|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Nova Pay Single Global Currency, Review the code and pushed the code to git. 1. Backend 2. Reporting 3. Webclient 4. Reporting|Development|09:00|11:00|02 Hrs 00 Min|||
|2|Fixing Notification Related Issues during transactions|Development|11:00|13:00|02 Hrs 00 Min|||
|3|All Default Wallets Setup api for local currency for a domain|


8-jul
|   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|
|1|Rebuild DB again and Test curl localhost:8090/setup/local-currency-wallets-so api for set up default wallets for local currency of SO|Development|09:00|10:00|01 Hrs 00 Min|||
|2|Daily Sync with Javed Sir|Development|10:00|10:15|00 Hrs 15 Min|||
|3|Removing reconciliation report dependency from TotalAccountBalanceFlat and use TotalAccountBalance to support multi currency|Development|10:15|13:00|02 Hrs 45 Min|||
|4|Update DailyReconciliationReportingJob, to use TotalAccountBalance to generate the report & Done reconciliation|

9-jul
|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Tested job scheduler which generate daily reconciliation report snapshots|Development|09:00|11:00|02 Hrs 00 Min|||
|2|Reconciliation report to support multiple currency filter|Development|11:00|13:00|02 Hrs 00 Min|||
|3|Refactored download report(Excel + Pdf) to support multi currency|Development|14:00|16:00|02 Hrs 00 Min|||
|4|Testing & Reconciliation of the reports with dummy data|Development|16:00|18:00|02 Hrs 00 Min|

10-jul
|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Review the code and push reconciliation report to support multi currency for all microservices|Development|09:00|10:00|01 Hrs 00 Min|||
|2|Studying docs and next what to do for multi currency and preparing for meeting at 11am|Development|10:00|11:00|01 Hrs 00 Min|||
|3|Meeting with Syed sir & Javed sir on Novapay|Development|11:00|11:30|00 Hrs 30 Min|||
|4|Currency code from properties instead of enum|Development|11:30|13:00|01 Hrs 30 Min|||
|5|Super Admin Dashboard to support multi currency|Development|14:00|17:00|03 Hrs 00 Min|||
|6|R&D on User to hold multiple wallets based on currency|Development|17:00|


13-jul
|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Multi domain Code review and Merge & R&D on how to implement new wallet creation for internal Agent|Development|09:00|13:00|04 Hrs 00 Min|||
|2|R&D on multi wallet for single user, implementation plan|Development|14:00|15:00|01 Hrs 00 Min|||
|3|ThirdParty Module Client Feedback and Demo Meeting|Meeting|15:00|16:30|01 Hrs 30 Min|||
|4|Running MFS backend, and preparing updates of todays Client meeting|

14-jul
|Task ID|Task Name|Category|Start Time|End Time|Duration|Status|Action|
|---|---|---|---|---|---|---|---|
|1|Fix status when approve the payment, switch marked as PAID even external API still processing it|Development|09:00|13:00|04 Hrs 00 Min|||
|2|Payment template Excel sheet Modification|Development|14:00|15:00|01 Hrs 00 Min|||
|3|Employee Incentive Payments: Commission, Bonus, Stipend.|Development|15:00|18:00|03 Hrs 00 Min|||

15-jul(Worked on GPAY not NovaPay)
|   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|
|1|During Incentive payments, instead of validation the sheet accounts with DB vendors or employees. These data may not exist in system. We need to validate direct Wallet & Switch from sheet|Development|09:00|11:00|02 Hrs 00 Min|||
|2|Incentive payments completed both for employee & vendor|Development|11:00|13:00|02 Hrs 00 Min|||
|3|Create Employee with bulk upload sheet. This is Legacy code which don't handle both Wallet & Switch Account. To do completely new implementation.|Development|14:00|17:00|03 Hrs 00 Min|||
|4|Create Vendor with bulk upload: New API & Page.|Development|17:00|19:30|02 Hrs 30 Min|||


16-jul
|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Completed Third Party Modification & Fixes & Pushed the code|Development|09:00|11:00|02 Hrs 00 Min|||
|2|Bug fix after Deployed by Javed Sir, webclient all account transfer|Development|11:00|13:00|02 Hrs 00 Min|||
|3|R&D on in Novapay to implement User to hold multiple wallet by currency|R&D|14:00|18:00|04 Hrs 00 Min|

17-jul
|Task ID|Task Name|Category|Start Time|End Time|Duration|Status|Action|
|---|---|---|---|---|---|---|---|
|1|Subscriber Multi Wallet Creation|Development|09:00|13:00|04 Hrs 00 Min|||
|2|Subscriber new currency wallet creation|Development|14:00|18:00|04 Hrs 00 Min|||

20-jul
|Task ID|Task Name|Category|Start Time|End Time|Duration|Status|Action|
|---|---|---|---|---|---|---|---|
|1|Subscriber Multi Wallet Maker Checker|Development|09:00|13:00|04 Hrs 00 Min|||
|2|Subscriber Multi Currency Wallet Maker Checker|Development|14:00|1|

21-jul
|Task ID|Task Name|Category|Start Time|End Time|Duration|Status|Action|
|---|---|---|---|---|---|---|---|
|1|Subscriber multi wallet creation maker checker completed|Development|09:00|10:00|01 Hrs 00 Min|||
|2|Using GpayAccountTransferService of Gpay in Novapay for All Account Transfers|Development|10:00|13:00|03 Hrs 00 Min|||
|3|Use GpayAccountTransferService of Gpay in Novapay for All Account Transfers|Development|14:00|18:00|04 Hrs 00 Min|||

22-jul
|Task ID|Task Name|Category|Start Time|End Time|Duration|Status|Action|
|---|---|---|---|---|---|---|---|
|1|Refactoring subscriber Cash-In(Customer care to subscriber) Transfer Flows to Support Multi-Currency Wallet Resolution|Development|09:00|13:00|04 Hrs 00 Min|||
|2|TransferRequestBodyAdvice fix to resolve wallets before request hit controller, only support TransferDto|Development|14:00|16:00|02 Hrs 00 Min|||
|3|/subscriber-cash-in refactor endpoint|Development|16:00|18:00|02 Hrs 00 Min|||
