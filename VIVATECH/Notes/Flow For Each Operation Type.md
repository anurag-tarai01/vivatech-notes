## Create flow
```java
Gateway Controller (AdminUserController / Agentcontroller / MerchantController)
    → validates DTO, enriches fields (status=PENDING, encode password)
    → commandBus.dispatch(CreateXxxCommand)
        → UserCommandHandler.handleCreateXxx()
           (repo.newInstance(() -> new User(dto)) OR new Admin(id, dto))
            → Aggregate constructor: apply(XxxCreatedEvent)
                → Event stored in Axon event store (SQL Server via JDBC)
                → @EventSourcingHandler on(XxxCreatedEvent) sets aggregate state
                → UserListener.handle(XxxCreatedEvent)
                   → userRepository.save(user)   ← read model updated
                → XxxRegistrationSaga.on(XxxCreatedEvent) starts, waits...
```

**Concrete classes:** `AdminController.createInternalAgent()` → `CreateAdminCommand` → `UserCommandHandler.handleCreateAdmin()` → `Admin(adminId, adminDto)` → `AdminCreatedEvent` → `UserListener.handle(AdminCreatedEvent)` → `adminUserQueryRepository.save(admin)`

## Approve flow
```java
Gateway Controller
    → validates approver exists and is correct type
    → commandBus.dispatch(ApproveXxxCommand)
        → User/Admin Aggregate.handle(ApproveXxxCommand)
            → apply(XxxApproveEvent)
                → UserListener.handle(XxxApproveEvent)
                   → user.setStatus(ACTIVE)
                   → userRepository.save(user)
                → XxxRegistrationSaga.on(XxxApproveEvent) resumes
                   → dispatch(CreateXxxWalletCommand)
                       → WalletCommandHandler creates Wallet aggregate
                           → WalletListener.on(XxxWalletCreateEvent)
                              → accountQueryRepository.save(wallet)
                   → dispatch(CreateXxxCommissionWalletCommand)   [if applicable]
                   → @EndSaga
```
**Concrete classes:** `AdminController.approveAgent()` → `ApproveAdminCommand` → `Admin.handle(ApproveAdminCommand)` → `AdminApprovedEvent` → `UserListener.handle(AdminApprovedEvent)` → `admin.setStatus(ACTIVE)` → `CreateCustomerCareSaga.on(AdminApprovedEvent)` → `CreateCustomerCareWalletCommand` → `WalletCommandHandler` → `CustomerCareWalletCreateEvent` → `WalletListener.on(CustomerCareWalletCreateEvent)`

## Update flow
```java
Controller → dispatch(UpdateXxxCommand)
    → Aggregate.handle(UpdateXxxCommand) → apply(XxxUpdatedEvent)
        → UserListener.on(XxxUpdatedEvent)
           → user.setUpdatedData(JSON of new data)
           → user.setStatus(PENDING)
           → userRepository.save(user)
→ [Second admin] Controller → dispatch(ApproveUpdateCommand / RejectUpdateCommand)
    → Aggregate → XxxUpdateApprovedEvent / XxxUpdateRejectedEvent
        → UserListener applies diff from updatedData OR reverts updatedData=null
        → user.setStatus(ACTIVE)
```

## Reject flow (creation)
```java
Controller → dispatch(RejectXxxCommand)
    → Aggregate → apply(XxxRejectEvent)
        → UserListener.handle(XxxRejectEvent)
           → userService.deleteUser(id)          ← hard deletes from DB
           → userService.deleteUserRoles(id)
           → userService.deleteUserInfoFromCache()
        → XxxRegistrationSaga.on(XxxRejectEvent)
           → @EndSaga  (no wallets ever created)
```
