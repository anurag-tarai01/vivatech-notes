Role: Business Lead fintech.
Context: We are having a wallet based transaction system where the transaction happens between subscriber to agent and vice versa. When the subscriber do the SUBSCRIBER_CASH_OUT transaction then small money is deposited in the commission wallet which is then disbursed at a certain date manually by the authorized person. The issue here is the subscriber send money from his SOS wallet and agent wallet receive money in his USD wallet and the commission deducted from the subscriber wallet is deposited into the SOS Commission_wallet. 
This is the transaction snippet 
Subscriber: SOS -> USD Wallet
Agent: USD Wallet
Subscriber (1000 SOS ) -> Agent(USD) , let 2% is service charge which is 20 SOS which is deposited to Commission_wallet
Condition 1: The commission disbursement done from the authorised person, and agent doesn't have the SOS wallet then what can we do, cancel the commission disbursement or add that money in the suspense wallet.
Condition 2: The transaction will not start nor failed nor the money will be added to the suspense wallet, it will stays in the commission_wallet when the Agent has the SOS wallet then only the money will be disbursed to his wallet.
Task: Find the best scenario for this. Reasoning instruction (Tree of Thought):
- Propose atleast 3 different solution approach.
- Evaluate pros/cons of each.
- Select the best solution and justify.
Constraint:
- The system handles 1000 transaction daily with the subscriber base of 200K.
- Solution must be production scalable.
Output Format:
- possible solution.
- comparision table.
- Final descision.
- Implementation plan.