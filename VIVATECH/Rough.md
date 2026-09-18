### **Demo Script: Multi-Currency and Multi-Domain Architecture**

#### **Phase 1: Initial System State (Single Domain & Currency)**

1. **System Initialization:** We begin with the system configured for a single geographical domain (Somalia), which was established during the initial setup. The designated foreign currency is USD, and the local currency is SOS.
2. **Currency Activation:** At present, USD is the sole active currency, while SOS remains inactive.
3. **Current Operating Mode:** Effectively, the application is currently operating in a single-domain, single-currency environment.

#### **Phase 2: Standard Single-Currency Operations (USD)**

4. **Transaction Demonstration:** We will now execute a series of standard transactions within this single-currency framework to establish a baseline before transitioning to multi-currency operations.
5. **Float Generation:** First, we generate the initial e-money float for the USD core master wallet.
6. **Internal Agent Onboarding:** Next, we create and onboard a new Internal Agent into the system.
7. **Agent Float Allocation:** We execute a B2B deposit (USD to USD) from the core master wallet to fund the Internal Agent's wallet.
8. **Subscriber Onboarding:** We then create and onboard a new Subscriber.
9. **Subscriber Cash-In:** The Internal Agent performs a standard Cash-In transaction to fund the Subscriber's USD wallet.
10. **Subscriber Cash-Out:** The Subscriber then performs a standard Cash-Out transaction.
11. **Secondary Subscriber Creation:** A second Subscriber is onboarded to demonstrate peer-to-peer capabilities.
12. **P2P Transfer:** We execute a standard Peer-to-Peer (P2P) USD transfer between the two subscribers.

#### **Phase 3: Multi-Currency Activation & Cross-Currency Transactions**

13. **Local Currency Activation (SOS):** We will now activate the local currency (SOS) for the Somalia domain. _(Note: The technical onboarding and backend configuration of new currencies and domains is handled entirely by the development team)._
14. **Core System Readiness:** The SOS currency onboarding is now complete, and SOS wallets have been successfully provisioned across all core system accounts.
15. **User Wallet Provisioning:** We will now provision SOS wallets for our existing Internal Agent and Subscribers.
16. **Agent Float Allocation (SOS):** We allocate SOS funds to the Internal Agent from the core master wallet.
17. **Cross-Currency Cash-In:** We will demonstrate a cross-currency Cash-In transaction, converting funds natively during the deposit.
18. **FX Commission Configuration:** We will now configure the Foreign Exchange (FX) commission charges in the system.
19. **Cross-Currency Cash-In (with FX):** We execute another cross-currency Cash-In, this time demonstrating the system's automated deduction and routing of the FX commission.
20. **Cross-Currency Cash-Out:** Next, the Subscriber performs a cross-currency Cash-Out transaction.
21. **Service Charge Configuration:** We will now configure standard transaction service charges for Cash-Out operations.
22. **Cash-Out (with Service Charges):** We repeat the Cash-Out transaction to demonstrate the automated calculation and deduction of the newly applied service charges.

#### **Phase 4: Multi-Domain Expansion & Cross-Border P2P**

23. **New Domain Onboarding:** We will now introduce a completely new geographical domain to the system, along with its respective currencies. _(Again, this backend setup is fully managed by the development team)._
24. **Cross-Domain Subscriber Onboarding:** We onboard a new Subscriber in this new domain and provision a CDF (Congolese Franc) wallet.
25. **Cross-Border P2P Transfer (SOS to CDF):** We will now demonstrate an international, cross-currency Peer-to-Peer transfer. Subscriber 1 will send funds from their local SOS wallet directly to the new Subscriber's local CDF wallet.

#### **Phase 5: System Reconciliation & Conclusion**

26. **Continuous Reconciliation:** Throughout this demonstration, the system has been continuously performing automated financial reconciliations in the background to ensure ledger integrity across all currencies.
27. **Final Reconciliation Report:** To conclude, we will generate and review the final system-wide reconciliation report. This verifies that all generated floats, user balances, FX commissions, and service charges balance perfectly across all domains and currencies.
28. **End of Demonstration:** This concludes the demonstration of our Multi-Currency and Multi-Domain architecture. Thank you.