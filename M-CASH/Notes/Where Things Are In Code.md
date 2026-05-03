
```
GATEWAY (port 3088) — request entry + auth enforcement
  com.vivacom.mfs.api.gateway.controller.
    AdminUserController.java       ← all admin CRUD + approve/reject/block
    Agentcontroller.java           ← agent create, approve, reject, unblock
    MerchantController.java        ← merchant create, approve, outlet block/close
    SubscriberController.java      ← subscriber create, approve, block, change-pincode
    UserController.java            ← bar/unbar, get-info, remarks, balance
    PublicApiController.java       ← /api/public/** (no auth)

BACKEND CORE (port 8090)
  Controllers (receive proxied requests):
    application/controller/AdminController.java      ← admin business logic
    application/controller/UserController.java       ← subscriber/agent/merchant logic
    application/controller/AgentController.java      ← get-child-agents
    application/controller/MerchantController.java   ← outlet block/unblock

  Commands + Events:
    core-api-kotlin/dto/AdminDto.java                ← DTO for all admin ops
    core-api-kotlin/dto/UserDto.java (implied)       ← DTO for user ops
    core-api/user/User.kt                            ← ALL command/event class definitions
                                                        (CreateSubscriberCommand, ApproveAgentCommand, etc.)

  Command Handlers:
    user/command/UserCommandHandler.java             ← handles all Create commands
    user/command/User.java                           ← handles all Update/Approve/Reject/Block
    user/command/Admin.java                          ← handles all Admin commands

  Aggregates (same files as handlers — Axon pattern):
    user/command/User.java                           ← User aggregate (Subscriber/Agent/Merchant)
    user/command/Admin.java                          ← Admin aggregate (all admin types)

  Sagas (multi-step orchestration):
    user/command/saga/SubscriberRegistrationSaga.java
    user/command/saga/AgentRegistrationSaga.java
    user/command/saga/MerchantRegistrationSaga.java
    user/command/saga/CreateCustomerCareSaga.java
    user/command/saga/CreateNetworkAdminSaga.java
    user/command/saga/SubscriberApproveSaga.java
    user/command/saga/AgentUpdateSaga.java
    user/command/saga/MerchantUpdateSaga.java
    user/command/saga/UserBarAsSenderSaga.java / UserUnBarAsSenderSaga.java

  Event Handlers (read model — where DB saves happen):
    user-query/listener/UserListener.java            ← THE central file. 87 KB.
                                                        Handles ALL user domain events.
                                                        All userRepository.save() calls.
                                                        All adminUserQueryRepository.save() calls.

  Read model repositories:
    user-query/repository/AdminUserQueryRepository.java
    user-query/repository/UserQueryRepository.java
    user-query/repository/MerchantQueryRepository.java (implied)

  Wallet creation (triggered post-approval):
    wallet/commandhandler/WalletCommandHandler.java  ← handles CreateXxxWalletCommand
    wallet/aggregate/Wallet.java                     ← Wallet aggregate
    wallet-query/listener/WalletListener.java        ← saves Wallet to DB
```