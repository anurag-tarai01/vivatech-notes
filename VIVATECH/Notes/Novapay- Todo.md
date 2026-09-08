### 9-07-2026:
1. System supports global USD and users, wallets, operations & curl to add local currency default wallets `Cmpleted`
2. Reconciliation Report - Support Multi Currency `Completed`
	- Just check convertRequestTimeZone - in reporting (Not support multi domain)
3. Domain Creation 
4. Wallets set for user when user is created, I have to check wallets creation saga code
5. If no FX rate present does it show the correct error message, you can send testing flag


# Issues to fix after MVP
1. Subscriber cash in (Internal agent)
	1. Internal agent if not have sufficient balance, should show correct error message, not whole response json
2. Pending wallet should not be show in dashboard(All actor)

3. Last 10 transaction to use target currency and aggregate id

4. Report fetch, if currency wallet not exist then it throw 500 error
	- e.g. SOS not exist for subscriber, but try to get report, it will through error





