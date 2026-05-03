Enforced at: `@PreAuthorize("@permissionService.check('PRIVILEGE_NAME')")` on every gateway endpoint — defined in `AdminUserController`, `Agentcontroller`, `MerchantController`, `SubscriberController` in the gateway.

|Who (has this privilege)|Can do what|
|---|---|
|`NETWORK_ADMIN_CREATE` (IT Manager, MyCash Manager)|Create Network Admin|
|`INTERNAL_AGENT_CREATE` (MyCash Manager)|Create Customer Care|
|`INTERNAL_AGENT_APPROVE` (MyCash Manager)|Approve/reject Customer Care|
|`AGENT_CREATE` (Internal Agent, Operation Manager)|Create Agent|
|`AGENT_APPROVE` (Internal Agent, Area Manager)|Approve/reject Agent|
|`MERCHANT_CREATE` (Internal Agent, Operation Manager)|Create Merchant|
|`MERCHANT_APPROVE` (Internal Agent)|Approve/reject Merchant, Outlet|
|`SUBSCRIBER_CREATE` (Agent, Internal Agent)|Create Subscriber|
|`SUBSCRIBER_APPROVE` (Internal Agent)|Approve Subscriber|
|`BARRING` (Operation Manager)|Bar any user as sender/receiver|
|`BLOCK_SUBSCRIBER` / `UNBLOCK_SUBSCRIBER` (Internal Agent)|Block/unblock subscriber|

---
