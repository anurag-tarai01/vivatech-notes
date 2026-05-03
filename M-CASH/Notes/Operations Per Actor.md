#### Admin (Customer Care + Network Admin)

| Operation             | Gateway endpoint                                | Command dispatched                                                          |
| --------------------- | ----------------------------------------------- | --------------------------------------------------------------------------- |
| Create Customer Care  | `POST /admin/register-customer-care`            | `CreateAdminCommand`                                                        |
| Create Network Admin  | `POST /admin/create-network-admin`              | `CreateNetworkAdminCommand`                                                 |
| Read — pending CC     | `POST /admin/pending-customer-care`             | query only                                                                  |
| Read — active CC      | `POST /admin/active-customer-care`              | query only                                                                  |
| Read — network admins | `POST /admin/network-admins`                    | query only                                                                  |
| Read — single info    | `GET /user/get-admin-info/{id}`                 | query only                                                                  |
| Update CC profile     | `POST /admin/update-customer-care`              | `AdminUpdateCommand`                                                        |
| Update CC password    | `POST /admin/update-customer-care-password`     | `AdminPasswordUpdateCommand`                                                |
| Update Network Admin  | `POST /admin/update-network-admin`              | `UpdateNetworkAdminCommand`                                                 |
| Delete                | —                                               | No hard delete. `RejectAdminCommand` deletes PENDING admin from DB entirely |
| Approve CC            | `POST /admin/approve-customer-care`             | `ApproveAdminCommand`                                                       |
| Reject CC             | `POST /admin/reject-customer-care`              | `RejectAdminCommand`                                                        |
| Reject CC update      | `POST /admin/reject-updated-customer-care`      | `AdminUpdateRejectedCommand`                                                |
| Block                 | `POST /admin/block`                             | `AdminBlockCommand`                                                         |
| Unblock               | `POST /admin/unblock`                           | `AdminUnblockCommand`                                                       |
| Role update request   | `POST /admin/create-admin-role-update-request`  | `CreateAdminRoleUpdateRequestCommand`                                       |
| Approve role update   | `POST /admin/approve-admin-role-update-request` | `ApproveAdminRoleUpdateRequestCommand`                                      |
| Suspend commission    | `POST /admin/suspend-commission-payment`        | `SuspendAdminCommissionPaymentCommand2`                                     |
| Resume commission     | `POST /admin/resume-commission-payment`         | `ResumeAdminCommissionPaymentCommand2`                                      |

---

#### Subscriber

| Operation              | Gateway endpoint                           | Command dispatched                                  |
| ---------------------- | ------------------------------------------ | --------------------------------------------------- |
| Create (by admin)      | `POST /user/register-subscriber`           | `CreateSubscriberCommand`                           |
| Create (self-register) | `POST /api/public/subscriber-registration` | `CreateSubscriberCommand`                           |
| Read — pending         | `POST /user/pending-subscriber`            | query only                                          |
| Read — active          | `POST /user/active-subscriber`             | query only                                          |
| Read — single info     | `POST /user/get-subscriber-info`           | query only                                          |
| Update                 | `POST /user/update-subscriber`             | `SubscriberUpdateCommand`                           |
| Approve                | `POST /user/approve-subscriber`            | `ApproveSubscriberCommand`                          |
| Reject                 | `POST /user/reject-subscriber`             | `RejectSubscriberCommand`                           |
| Approve update         | `POST /user/approve-subscriber-update`     | `SubscriberUpdateApprovedCommand`                   |
| Reject update          | `POST /user/reject-updated-subscriber`     | `SubscriberUpdateRejectedCommand`                   |
| Unblock                | `POST /user/unblock-subscriber`            | `UserUnblockCommand`                                |
| Change pincode         | `POST /user/change-pincode`                | `ChangePincodeCommand`                              |
| Change phone number    | `POST /user/change-phone-number-by-admin`  | `CreatePhoneNumberUpdateCommand`                    |
| Bar as sender          | `POST /user/bar`                           | `BarUserSenderCommand`                              |
| Bar as receiver        | `POST /user/bar`                           | `BarUserReceiverCommand`                            |
| Unbar                  | `POST /user/unbar`                         | `UserUnBarAsSenderSaga` / `UserUnBarAsReceiverSaga` |
| Assign account manager | `POST /user/assign-account-manager`        | `UpdateAccountManagerCommand2`                      |

---

#### Agent

| Operation           | Gateway endpoint                  | Command dispatched                               |
| ------------------- | --------------------------------- | ------------------------------------------------ |
| Create              | `POST /user/register-agent`       | `CreateAgentCommand`                             |
| Read — pending      | `POST /user/pending-agent`        | query only                                       |
| Read — active       | `POST /user/active-agent`         | query only                                       |
| Read — child agents | `POST /agent/get-child-agents`    | query only                                       |
| Update              | `POST /user/update-agent`         | `AgentUpdateCommand`                             |
| Approve             | `POST /user/approve-agent`        | `ApproveAgentCommand`                            |
| Reject              | `POST /user/reject-agent`         | `RejectAgentCommand`                             |
| Reject update       | `POST /user/reject-updated-agent` | `AgentUpdateRejectedCommand`                     |
| Unblock             | `POST /user/unblock-agent`        | `UserUnblockCommand`                             |
| Bar / Unbar         | `POST /user/bar` / `/user/unbar`  | `BarUserSenderCommand` / `UserUnBarAsSenderSaga` |

---

#### Merchant + Outlet

| Operation        | Gateway endpoint                     | Command dispatched                       |
| ---------------- | ------------------------------------ | ---------------------------------------- |
| Create merchant  | `POST /user/register-merchant`       | `CreateMerchantCommand`                  |
| Read — pending   | `POST /user/pending-merchant`        | query only                               |
| Read — active    | `POST /user/active-merchant`         | query only                               |
| Update           | `POST /user/update-merchant`         | `MerchantUpdateCommand`                  |
| Approve merchant | `POST /user/approve-merchant`        | `ApproveMerchantCommand`                 |
| Reject merchant  | `POST /user/reject-merchant`         | `RejectMerchantCommand`                  |
| Reject update    | `POST /user/reject-updated-merchant` | `MerchantUpdateRejectedCommand`          |
| Approve outlet   | `POST /user/approve-outlet`          | `ApproveMerchantCommand` (per outlet)    |
| Reject outlet    | `POST /user/reject-outlet`           | `RejectMerchantCommand`                  |
| Block outlet     | `POST /user/merchant/outlet/block`   | direct DB update in `MerchantController` |
| Unblock outlet   | `POST /user/merchant/outlet/unblock` | direct DB update                         |
| Close outlet     | `POST /user/close-outlet`            | `OutletCloseSaga`                        |
| Unblock merchant | `POST /user/unblock-merchant`        | `UserUnblockCommand`                     |