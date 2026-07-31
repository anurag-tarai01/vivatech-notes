## Subscriber Cash-Out — Complete Flow (Webclient to Response)

> **Scenario:** A subscriber logs into the webclient portal and withdraws 5,000 XAF from an agent (Customer Care) account.

There are **two execution paths** in the backend for `SUBSCRIBER_CASH_OUT`:

- **Path A** — `toWallet` is a `CUSTOMER_CARE` wallet → stored as a pending approval request (agent must approve on their portal)
- **Path B** — `toWallet` is an `AGENT` wallet → processed immediately, money moves instantly

The code explicitly branches on this. Both paths are traced below.

---

### Microservices Involved

```
┌──────────────────────┐
│   mfs-webclient      │  Port: (internal, serves browser UI)
│   (Spring MVC + Thymeleaf)
└─────────┬────────────┘
          │ HTTP (RestTemplate)
┌─────────▼────────────┐
│   mfs-api-gateway    │  Port: 3088
│   (JWT auth + proxy) │
└─────────┬────────────┘
          │ HTTP (RestTemplate)
┌─────────▼────────────┐
│   mfs-backend-new    │  Port: 8090
│   (business logic)   │
└────┬──────────┬───────┘
     │          │ RabbitMQ
     │   ┌──────▼──────────┐
     │   │ mfs-reporting   │  Port: 9090
     │   │ (async)         │
     │   └─────────────────┘
     │ HTTP (RestTemplate — only on CUSTOMER_CARE path)
┌────▼─────────────────┐
│ mfs-notification     │  Port: 8989
│ (push/SMS notify)    │
└──────────────────────┘
```

---

### Phase 0 — Subscriber Login (Webclient → Gateway → Backend)

Before any cash-out can happen, the subscriber must be logged in. The webclient stores the JWT token in the HTTP session.

**Step 0.1 — Subscriber navigates to login page**

Browser hits `GET /admin/subscriber-login` → `LoginController.subscriberLogin()` (webclient) → renders `subscriber-login.html` — a form with msisdn + OTP fields

**Step 0.2 — Request OTP**

Browser submits msisdn, webclient calls gateway `POST /request-otp`. Gateway's `SecurityController` proxies it to `mfs-notification` at `POST /user/request-otp`. Notification service generates OTP (hardcoded `9999` in test), saves it, sends SMS.

**Step 0.3 — Verify OTP and get token**

Browser submits OTP. Webclient calls `POST /admin/get-otp-login-token`.

`LoginController.getOtpToken()` (webclient):

java

```java
String url = apiGatewayAddress + "/get-otp-login-token";
OtpLoginRequest otpLoginRequest = new OtpLoginRequest();
otpLoginRequest.setDeviceId(deviceId);
otpLoginRequest.setMsisdn(msisdn);
otpLoginRequest.setPincode(otp);          // "9999"
OtpTokenResponse res = restTemplate.postForObject(url, otpLoginRequest, OtpTokenResponse.class);
```

Gateway `SecurityController.getOtpLoginToken()`:

- Verifies OTP against DB
- Checks user status is `ACTIVE`
- Issues JWE token via `TokenService.generateToken()` (RSA-OAEP-512 + A256CBC-HS512)
- Returns `OtpTokenResponse { token, userId, userType=SUBSCRIBER, walletId, ... }`

Back in `LoginController.getOtpToken()`:

java

```java
SessionHelper.setToken(res.getToken());             // JWT stored in HTTP session
SessionHelper.setAdminId(res.getUserId());          // subscriber DB id in session
SessionHelper.setSubscriberWalletId(dto.getWalletId()); // subscriber wallet aggregateId in session
SessionHelper.setUserInfoDto(dto);                  // full user info in session
```

Browser is redirected to `redirect:dashboard`.

---

### Phase 1 — Open Cash-Out Form (Webclient GET)

Subscriber clicks "Withdraw Cash from Agent" in the dashboard.

**Browser hits:** `GET /admin/deposit/subscriber-to-agent-deposit/create`

**`DepositController.subscriberToCashOutCreate()`** (webclient):

java

```java
int userId = SessionHelper.getAdminId();             // from session
SubscriberInfoDto subscriberInfo = userService.getSubscriberInfo(userId);
                                // → REST GET to gateway → backend → returns subscriber info + balance
model.addAttribute("subscriberInfo", subscriberInfo);
return "admin/deposit/subscriber/subscriber-to-agent-create";
```

**What the HTML renders** (`subscriber-to-agent-create.html`):

- Shows subscriber's own phone number and current balance (from `subscriberInfo`)
- Dropdown "To Agent" — populated via AJAX `POST /admin/auto-search/get-internal-external-agents-by-parts` as the user types (Select2 library)
- Amount input field
- Pincode field (4-digit)
- Save button which calls `submitSubscriberCashOutForm()` in `agent.js`

---

### Phase 2 — Subscriber Submits the Form (Browser → Webclient)

Subscriber selects an agent, enters amount (e.g., 5000) and pincode (e.g., 1234), clicks "Save".

**`submitSubscriberCashOutForm()` in `agent.js`** executes first:

javascript

```javascript
function submitSubscriberCashOutForm() {
    if($("#subscriberCashOutForm").valid()) {
        // shows SweetAlert confirmation popup:
        Swal.fire({
            title: "Are you sure?",
            html: "<h4>Cash out from AgentName</h4><br>Amount: <b>5000</b>",
            showCancelButton: true,
            confirmButtonText: "Confirm"
        }).then((result) => {
            if (result.value == true) {
                $("#subscriberCashOutForm").submit();  // only submits after confirmation
            }
        });
    }
}
```

After subscriber clicks "Confirm" in the popup, the form POSTs to: `POST /admin/deposit/subscriber-to-agent-deposit/store`

**`DepositController.storeSubscriberToCashOutStore()`** (webclient):

java

```java
String url = apiGatewayAddress + "/transfer/subscriber-cash-out";
double amount = Double.parseDouble(formData.getFirst("amount"));     // "5000" → 5000.0
String receiverId = formData.getFirst("toAccountId");                // agent's wallet aggregateId
String pinCode = formData.getFirst("pincode");                       // "1234"
BigMoney bigMoney = BigMoney.of(CurrencyUnit.of("XAF"), 5000.0);
```

**⚠️ Local pincode check in webclient (client-side double check):**

java

```java
UserInfoDto userInfoDto = SessionHelper.getUserInfoDto();  // from session
if (!passwordEncoder.matches(pinCode, userInfoDto.getPincode())) {
    // pincode mismatch → immediately return error, never call gateway
    redirectAttributes.addFlashAttribute("errorMessage", "Wrong Pin");
    return "redirect:/admin/dashboard";
}
```

This is a **pre-flight local check** — the webclient validates the pincode against what was stored in the session during login. If it fails here, the request never leaves the webclient.

If pincode matches locally, builds the request:

java

```java
TransferDto transferDto = new TransferDto();
transferDto.setFromAccountId(SessionHelper.getSubscriberWalletId()); // from session e.g. "SWALLET-SUB-001"
transferDto.setToAccountId(receiverId);                              // agent wallet aggregateId
transferDto.setAmount(bigMoney);                                     // XAF 5000
transferDto.setPincode(pinCode);                                     // "1234" (raw, for backend re-validation)
transferDto.setTransferInitiedById(SessionHelper.getAdminId());      // subscriber DB id
transferDto.setTransferInitietedByType(TransferInitiatedByType.SUBSCRIBER);
transferDto.setChannelType(ChannelType.WEB);
transferDto.setAsync(false);
```

Sends it:

java

```java
restTemplate.setInterceptors(SessionHelper.getInterceptor());
// getInterceptor() adds:
//   "Authorization: Bearer <jwt_token from session>"
//   "x-trace-id: <MDC trace UUID>"
//   "X-Forwarded-For: <client IP>"
TransferResponseDto responseTransferDto = restTemplate.postForObject(url, transferDto, TransferResponseDto.class);
```

---

### Phase 3 — API Gateway Receives Request

**`TransferController.SubscriberCashOut()`** (gateway) **File:** `mfs-api-gateway/controller/TransferController.java` line 125

java

```java
@RequestMapping(value = "/subscriber-cash-out", method = RequestMethod.POST)
@PreAuthorize("@permissionService.check('DEPOSIT_SUBSCRIBER_CASH_OUT')")
public Object SubscriberCashOut(@RequestBody @Valid TransferDto requestDto) {
```

**Step 3.1 — JWT validation (before method executes)**

`ApiAuthorizationFilter` (gateway Spring Security filter) intercepts:

- Extracts Bearer token from `Authorization` header
- Calls `TokenService.validateToken()` — decrypts JWE with private RSA key
- Extracts claims: `sub` (userId), `scope` (roles), `deviceId`
- Validates `deviceId` in token matches device in DB
- Sets `SecurityContextHolder.setContext(authentication)` with subscriber's roles

**Step 3.2 — `@PreAuthorize` executes**

`permissionService.check('DEPOSIT_SUBSCRIBER_CASH_OUT')` — checks if subscriber's roles contain this privilege.

> **Note from code:** The privilege check here is `DEPOSIT_SUBSCRIBER_CASH_OUT`. For a subscriber doing self cash-out via the webclient, this privilege must be assigned to the subscriber role. Looking at the code, the old admin-driven cash-out endpoint was commented out with this same privilege check. The active path (`subscriber-to-agent-deposit`) uses `@PreAuthorize("@permissionService.check('ROLE_SUBSCRIBER_USER')")` in the webclient but the gateway endpoint still requires `DEPOSIT_SUBSCRIBER_CASH_OUT`. Both must pass.

**Step 3.3 — Forward to backend**

java

```java
String uri = coreServerAddress + "/transfer/subscriber-cash-out";
ResponseEntity<Object> out = getPOSTApiResponse(restTemplate, uri, requestDto, Object.class);
```

`getPOSTApiResponse()` (gateway `BaseController`):

- Adds `X-User-ID: <userId>` header
- Adds `X-Trace-ID: <traceId>` header
- Adds `X-Forwarded-For: <clientIP>` header
- POSTs `requestDto` (the full `TransferDto`) to backend

**Step 3.4 — Gateway checks response and calls notification**

java

```java
LinkedHashMap<String, String> result = (LinkedHashMap<String, String>) out.getBody();
String status = result.get("status");
if (status.equalsIgnoreCase("Success")) {
    // Only on success — call notification service
    String notificationUri = notificationServerAddress + "user/send-cashout-approval-notification";
    getPOSTApiResponse(restTemplate, notificationUri, requestDto, Object.class);
}
return out;
```

This notification call happens **after** the backend responds and only if status is `"Success"`.

---

### Phase 4 — Backend Core Processing

**`TransferController.SubscriberCashOut()`** (backend) **File:** `application/controller/TransferController.java` line 588

java

```java
TransferEventDto dto = mapper.getMapperFacade().map(requestDto, TransferEventDto.class);
dto.setCreatedAt(new Date());
dto.setUpdatedAt(new Date());
dto.setTransferType(TransferType.SUBSCRIBER_CASH_OUT);             // ← forced here
String transferId = utils.getAccountTransferId(dto.getTransferType());
// → generates e.g. "SCO260427.1045.X9Y8Z7"  (SCO = Subscriber Cash Out prefix)
dto.setTransferAggregateId(transferId);
dto.setTransferMetadata("");
```

**Step 4.1 — Load wallet info**

java

```java
WalletInfo fromWallet = walletService.getWalletInfo(dto.getFromAccountAggregateId());
// fromWallet.type = SUBSCRIBER, fromWallet.userId = subscriber's DB id

WalletInfo toWallet = walletService.getWalletInfo(dto.getToAccountAggregateId());
// toWallet.type = AGENT OR CUSTOMER_CARE

UserInfo toUser = userService.getUserInfoFromId(toWallet.getUserId());
UserInfo fromUser = userService.getUserInfoFromId(fromWallet.getUserId());
```

**Step 4.2 — Run validation**

java

```java
validateTransferRequestNew(dto);
// → TransferValidationService.checkTransactionValidations(dto)
// → TransferValidatorService.validate(dto)
```

**`TransferValidatorService.validate()`** fetches both wallets from DB and runs all rules:

**Rule 1 — `AccountsVerificationRule.handleSubscriberCashOut()`:**

java

```java
List<String> validWalletTypes = Arrays.asList(
    WalletType.AGENT.toString(), WalletType.CUSTOMER_CARE.toString()
);
if (!validWalletTypes.contains(toAccountInfo.getAccountWalletType().toString())) {
    return new ValidationResult(false, "account X is not a valid agent account");
}
if (fromAccountInfo.getAccountWalletType() != WalletType.SUBSCRIBER) {
    return new ValidationResult(false, "account X is not a valid subscriber account");
}
```

Also runs `validateAccountStatus()`:

- `fromWallet` (subscriber) must be `ACTIVE` or `BARRED_AS_RECEIVER` (not BARRED_AS_SENDER or BLOCKED)
- `toWallet` (agent) must be `ACTIVE` or `BARRED_AS_SENDER` (not BARRED_AS_RECEIVER or BLOCKED)

**Rule 2 — `PincodeVerificationRule`:** BCrypt-matches `dto.getPincode()` ("1234") against subscriber's stored pincode. This is the **backend re-validation** after the webclient's local check.

**Rule 3 — `AccountBalanceLimitValidaionRule`:** Confirms subscriber balance ≥ amount + service charge.

**Rule 4 — `ThresholdValidationRule`:** Checks amount is within subscriber's grade-based transaction limits (min/max per transaction, daily/monthly caps).

**Rule 5 — `MaxMinBalanceLimitValidationRule`:** Checks subscriber balance after deduction won't go below their minimum allowed balance.

**Rule 6 — `CoolDownValidationRule`:** Checks subscriber isn't in a cooldown period.

**Step 4.3 — Branch on toWallet type**

java

```java
if (WalletType.CUSTOMER_CARE.equals(toWallet.getWalletType())) {
    // PATH A — Customer Care (Internal Agent) cashout
    boolean status = internalAgentTransferRequestsService
        .addNewInternalAgentTransferRequests(dto, fromUser, fromWallet, toUser, toWallet);
    if (status) {
        return BaseResponseDto { status="Success", statusCode=PROCESSING,
                                  message="Please wait while the agent approves the transfer" };
    }
}
// PATH B — Regular Agent cashout
BaseResponseDto responseDto = newAccountTransferService.processTransaction(dto);
return responseDto;
```

---

### Path A — Customer Care (Internal Agent) Cash-Out

**`InternalAgentTransferRequestsService.addNewInternalAgentTransferRequests()`:**

java

```java
InternalAgentTransferRequests request = InternalAgentTransferRequests.builder()
    .transferId("SCO260427.1045.X9Y8Z7")
    .amount(XAF 5000)
    .sender(subscriberEntity)              // from userQueryRepository
    .fromWallet("SWALLET-SUB-001")
    .receiver(customerCareAdminEntity)     // from adminUserQueryRepository
    .toWallet("CWALLET-CC-001")
    .status(UpdateStatus.PENDING)
    .transferData(utils.serializeMetadata(dto))  // full dto JSON
    .submittedById(subscriberDbId)
    .createdAt(new Date())
    .expiryTime(new Date(now + TRANSFER_REQUEST_VALIDITY))  // 60 minutes validity
    .build();
internalAgentTransferRequestsQueryRepository.save(request);  // writes to internal_agent_transfer_requests table
return true;
```

**No wallet balance changes. No accounting entries. Just a pending request in DB.**

**What happens next (Customer Care approves on their portal):**

Customer Care sees a notification on their portal (from the notification service) and goes to their pending cash-out requests list. They click "Approve".

The webclient calls the backend `POST /transfer/approve-cash-out-request` → loads the serialized `dto` from `InternalAgentTransferRequests.transferData` → calls `newAccountTransferService.processTransaction(dto)` — which runs the **same path as Path B** (full wallet debit/credit flow). Until this approval, zero money has moved.

---

### Path B — Regular Agent Cash-Out (Immediate)

**`NewAccountTransferService.processTransaction(dto)`**

`SUBSCRIBER_CASH_OUT` is NOT in the bypass list at the top of `processTransaction()`, so the full flow runs:

**Step B.1 — Scale amount**

java

```java
BigMoney amount = dto.getAmount();
dto.setAmount(MfsUtils.getScaledMoney(amount));   // rounds to 4 decimal places: 5000.0000 XAF
```

**Step B.2 — Re-validate (second validation pass)**

java

```java
dto = transferValidationService.checkTransactionValidations(dto);
// Same rules run again — pincode re-checked, balance re-checked against current live DB values
```

**Step B.3 — Calculate service charge + commission**

java

```java
dto = serviceAndCommissionChargeService.calculateServiceAndCommissionCharge(dto);
// Reads SUBSCRIBER_CASH_OUT service charge profile from DB
// Example: 1% of 5000 = 50 XAF service charge
// Commission for agent commission wallet: e.g., 25 XAF
// dto.setServiceCharge(XAF 50)
// dto.setCommissionInfos([{agentCommissionWalletId, XAF 25}])
```

**Step B.4 — Check for error after commission calc**

java

```java
if (!StringUtils.isEmpty(dto.getErrorReason())) {
    dto = storeFailedTransaction(dto);
    dto.setEventType("AccountTransferFailedEvent");
    rabbitMQProducer.rabbitProducer(dto);
    return FAILED response;
}
```

**Step B.5 — Save initial DB record**

java

```java
accountTransferListener.initiateTransaction(dto);
```

**`AccountTransferListener.initiateTransaction()`:**

- Creates `AccountTransfer` entity:
    - `id = "SCO260427.1045.X9Y8Z7"`
    - `fromAccountId = "SWALLET-SUB-001"` (subscriber wallet)
    - `toAccountId = "AWALLET-AGT-001"` (agent wallet)
    - `amount = XAF 5000`
    - `transferType = SUBSCRIBER_CASH_OUT`
    - `transferStatus = STARTED`
    - `createdAt = now`
    - `serviceCharge = XAF 50`
- Creates `AccountTransferMetadata` (empty string for cash-out)
- `queryrepository.saveAndFlush(accountTransfer)` → **first DB write**

**Step B.6 — Publish to RabbitMQ**

java

```java
dto.setEventType("AccountTransferCreateEvent");
rabbitMQProducer.rabbitProducer(dto);
// → RabbitTemplate.convertAndSend("mfs-core-exchange", "mfs-core-key", messageMap)
// → mfs-reporting picks this up for initial audit trail
```

**Step B.7 — Update wallet balances**

java

```java
dto = newWalletService.updateWalletBalances(dto);
```

**`NewWalletService.updateWalletBalances()` for `SUBSCRIBER_CASH_OUT`:**

This is NOT one of the special cases at the top of the method, so goes to the default else branch:

java

```java
totalWithdrawAmount = MfsUtils.getScaledMoney(totalWithdrawAmount.plus(serviceCharge));
// totalWithdrawAmount = 5000 + 50 = 5050 XAF
```

**DEBIT subscriber wallet:**

java

```java
BigMoney withdrawWalletCurrentBalance = getCurrentWalletBalance("SWALLET-SUB-001");
// → walletQueryRepository.findByAggregateId("SWALLET-SUB-001").getBalance()
// → returns 10,000 XAF

dto.setWithDrawBalanceData(withdrawWalletCurrentBalance, totalWithdrawAmount);
// stores "balance before=10000, debit=5050" in dto for reporting

String oldTransferId = getLastTransferId("SWALLET-SUB-001");
// reads wallet.transfer_id field = "SCI-PREV-001" (last successful transfer)

int rowAffected = walletQueryRepository.withdrawFromWallet(
    "SWALLET-SUB-001",      // walletId
    5050,                   // debitAmount (BigDecimal)
    new Date(),             // updatedAt
    "SCO260427.1045.X9Y8Z7", // newTransferId
    "SCI-PREV-001",          // oldTransferId (concurrency guard)
    lastTransferId           // fallback transfer ID
);
// SQL:
// UPDATE Wallet SET amount = amount - 5050, transfer_id = 'SCO260427.1045.X9Y8Z7'
// WHERE aggregate_id = 'SWALLET-SUB-001'
// AND (transfer_id = 'SCI-PREV-001' OR transfer_id = lastTransferId)

if (rowAffected != 1) throw new MFSException("There was some error, Please try again.");
// Subscriber wallet balance: 10,000 - 5,050 = 4,950 XAF
```

Publishes `MoneySubtractedEvent` to RabbitMQ for reporting wallet history.

**CREDIT agent wallet:**

java

```java
totalDepositAmount = dto.getAmount();
// = 5000 XAF (not 5050 — subscriber pays fee, agent receives face value)

BigMoney depositWalletCurrentBalance = getCurrentWalletBalance("AWALLET-AGT-001");
// = 20,000 XAF

dto.setDepositBalanceData(depositWalletCurrentBalance, totalDepositAmount);

walletQueryRepository.depositToWallet("AWALLET-AGT-001", 5000, new Date());
// SQL:
// UPDATE Wallet SET amount = amount + 5000 WHERE aggregate_id = 'AWALLET-AGT-001'
// Agent wallet balance: 20,000 + 5,000 = 25,000 XAF
```

Publishes `MoneyAddedEvent` to RabbitMQ for reporting wallet history.

**CREDIT agent commission wallet:**

java

```java
handleCommissions(dto, ...);
// dto.getCommissionInfos() = [{agentCommissionWalletId="AWALLET-COMM-001", amount=25}]
walletQueryRepository.depositToWallet("AWALLET-COMM-001", 25, new Date());
// SQL:
// UPDATE Wallet SET amount = amount + 25 WHERE aggregate_id = 'AWALLET-COMM-001'
// Agent commission wallet: += 25 XAF
```

Publishes `MoneyAddedEvent` for commission to RabbitMQ.

**Step B.8 — Third party call**

java

```java
dto = newThirdPartyService.handleThirdPartyService(dto);
// For SUBSCRIBER_CASH_OUT: no third-party provider found → no-op, returns dto unchanged
```

**Step B.9 — Finish transaction**

java

```java
finishTransaction(dto);
```

**`finishTransaction()` calls in sequence:**

1. `accountTransferListener.processTransaction(dto)`:
    - Creates `AccountTransferOperation` rows:
        - WITHDRAW: `{transferId, walletId="SWALLET-SUB-001", type=WITHDRAW, amount=5050}`
        - DEPOSIT: `{transferId, walletId="AWALLET-AGT-001", type=DEPOSIT, amount=5000}`
    - Updates `AccountTransfer.transferStatus = DONE`
    - Saves all via JPA repositories
2. `accountTransferListener.updateApprovalDetail(dto)`:
    - Sets `AccountTransfer.transferApprovedByAggregateId` if an approver was involved
3. `accountTransferListener.updateTransactionStatus(dto)`:
    - Updates `AccountTransfer.transferStatus` based on transfer type final state
4. `accountTransferListener.completeTransaction(dto)`:
    - For `SUBSCRIBER_CASH_OUT`: no special case — no-op
5. `commissionInfoListener.saveCommissionData(dto)`:
    - Creates `CommissionInfo` entity for the agent's 25 XAF commission
    - Saves to `commission_info` table
6. `promotionListener.handlePromotionalOffer(dto)`:
    - Checks if any promotional offer applies to this transaction
    - If yes, credits promotional reward
7. `fixTransferMetadataProcessor.handleMetadataForCompleteTransaction(dto)`:
    - Cleans up and finalizes transfer metadata

**Step B.10 — Publish AccountTransferDoneEvent**

java

```java
// SUBSCRIBER_CASH_OUT is NOT excluded from notification event
dto.setEventType("AccountTransferDoneEvent");
rabbitMQProducer.rabbitProducer(dto);
// → mfs-notification service picks this up
// → sends SMS/push to subscriber: "You withdrew 5000 XAF"
// → sends SMS/push to agent: "You received 5000 XAF"
```

**Step B.11 — Publish AccountTransferCompletedEvent**

java

```java
dto.setEventType("AccountTransferCompletedEvent");
rabbitMQProducer.rabbitProducer(dto);
// → mfs-reporting picks this up for CustomerTransactionReport
```

**Step B.12 — Return response**

java

```java
responseDto.setStatusCode(StatusCode.SUCCESS);
responseDto.setStatus("Success");
responseDto.setMessage("Transaction Success,Transfer Id: SCO260427.1045.X9Y8Z7");
return responseDto;
```

---

### Phase 5 — Gateway Receives Backend Response

Back in `TransferController.SubscriberCashOut()` (gateway):

java

```java
LinkedHashMap<String, String> result = (LinkedHashMap<String, String>) out.getBody();
String status = result.get("status");   // "Success"
if (status.equalsIgnoreCase("Success")) {
    String notificationUri = notificationServerAddress + "user/send-cashout-approval-notification";
    getPOSTApiResponse(restTemplate, notificationUri, requestDto, Object.class);
    // This is for the CUSTOMER_CARE path — sends a push notification to the agent
    // asking them to approve the request.
    // For Path B (regular agent) this still fires but the notification service
    // checks if agentInfo.getUserType() == ADMIN — if not ADMIN, it does nothing.
}
return out;  // returns the backend response body to the webclient
```

**`send-cashout-approval-notification` in `mfs-notification`:**

`UserController.sendCashoutApprovalNotification()` (notification):

java

```java
UserInfo agentInfo = walletService.getUserInfoFromWalletId(agentWallet);
if (agentInfo.getUserType() == UserType.ADMIN) {
    // Only for CUSTOMER_CARE agents (UserType.ADMIN in the admin table)
    userNotificationProcessor.sendCashoutApprovalNotification(
        agentInfo, subscriberName, subscriberPhone, userType, amount
    );
    // Sends push notification to Customer Care agent's device:
    // "Subscriber John Doe (237611111111) wants to withdraw 5000 XAF. Please approve."
}
// For regular AGENT — agentInfo.getUserType() is AGENT, not ADMIN → no notification sent
```

---

### Phase 6 — Notification via RabbitMQ (Async)

Simultaneously with Phase 5, `mfs-notification` also consumes from RabbitMQ:

`RabbitMQConsumer.rabbitConsumer()` (notification service) receives `AccountTransferDoneEvent`:

- Routes to `notificationService.handleAccountTransferDoneEvent(dto)`
- Identifies transfer type `SUBSCRIBER_CASH_OUT`
- Sends **SMS to subscriber**: "Dear John, you have withdrawn 5000 XAF. Your balance: 4950 XAF."
- Sends **SMS to agent**: "You have received 5000 XAF from subscriber John. Your balance: 25000 XAF."
- Sends **push notification** if both parties have registered push tokens

---

### Phase 7 — Reporting (Async via RabbitMQ)

`RabbitMQConsumer.rabbitConsumer()` (reporting service `mfs-reporting`):

**`AccountTransferCreateEvent`** → `newTransactionAuditReportService.accountTransferCreateEvent()`:

- Creates initial audit record in reporting DB

**`MoneySubtractedEvent` / `MoneyAddedEvent`** → `newTransactionReportingService.handleMoneySubtractedEvent()` / `handleMoneyAddedEvent()`:

- Updates wallet history in reporting DB for both subscriber and agent wallets

**`AccountTransferCompletedEvent`** → `reportProcessingService.accountTransferCompletedEvent()`:

- Creates `CustomerTransactionReport` entity:
    - `transferType = SUBSCRIBER_CASH_OUT`
    - Payer: subscriber (name, msisdn, zone, area, grade, balance before/after)
    - Payee: agent (name, msisdn, zone, area)
    - `amount = 5000`, `serviceCharge = 50`, commission info
    - `status = SUCCESS`
- `CustomerTransactionReportRepository.save(report)` → **reporting DB write**
- Updates `RecentTransaction` for subscriber:

java

```java
case SUBSCRIBER_CASH_OUT:
    recentTransaction = recentTransactionRepository
        .findByUserIdAndDestinationAndTransferType(subscriberId, agentWalletId, SUBSCRIBER_CASH_OUT);
    if (recentTransaction != null) {
        recentTransaction.setLastTransaction(dto.getCreatedAt());
        recentTransactionRepository.save(recentTransaction);
    } else {
        // create new RecentTransaction record for this subscriber-agent pair
        recentTransaction = RecentTransaction.builder()
            .destination(agentWalletId)
            .destinationName(agentWalletName)
            .transferType(SUBSCRIBER_CASH_OUT)
            .userId(subscriberId)
            .lastTransaction(now).build();
        recentTransactionRepository.save(recentTransaction);
    }
```

---

### Phase 8 — Webclient Renders Response

Back in `DepositController.storeSubscriberToCashOutStore()` (webclient):

java

```java
TransferResponseDto responseTransferDto = restTemplate.postForObject(url, transferDto, TransferResponseDto.class);

if (responseTransferDto.getStatusCode() == StatusCode.SUCCESS
        || responseTransferDto.getStatusCode() == StatusCode.PROCESSING) {
    redirectAttributes.addFlashAttribute("successMessage", responseTransferDto.getMessage());
    return "redirect:/admin/dashboard";
} else {
    redirectAttributes.addFlashAttribute("errorMessage", responseTransferDto.getMessage());
    return "redirect:create";  // back to the cash-out form
}
```

Browser follows the redirect to `/admin/dashboard`. The dashboard template reads `successMessage` from flash attributes and shows a green success alert:

> **"Transaction Success, Transfer Id: SCO260427.1045.X9Y8Z7"** or **"Please wait while the agent approves the transfer"** (Path A)

---

### Final Balance Summary

```
Before:
  Subscriber wallet (SWALLET-SUB-001):    10,000 XAF
  Agent wallet (AWALLET-AGT-001):         20,000 XAF
  Agent commission wallet (AWALLET-COMM): 2,000 XAF

After (Path B — instant):
  Subscriber wallet:   4,950 XAF  (−5,050 = amount 5000 + fee 50)
  Agent wallet:       25,000 XAF  (+5,000)
  Agent commission:    2,025 XAF  (+25 commission)

After (Path A — pending, no change until CC approves):
  All wallets:        unchanged until Customer Care approves
```

---

### Complete End-to-End Diagram

```
BROWSER
  │ User clicks "Withdraw Cash from Agent"
  ▼
mfs-webclient: GET /admin/deposit/subscriber-to-agent-deposit/create
  │ DepositController.subscriberToCashOutCreate()
  │ → fetches subscriber info from gateway → renders HTML form
  ▼
BROWSER shows form: Agent dropdown, Amount, Pincode

  │ User fills form, clicks Save
  │ agent.js: submitSubscriberCashOutForm() → SweetAlert confirm popup
  │ User clicks Confirm
  ▼
mfs-webclient: POST /admin/deposit/subscriber-to-agent-deposit/store
  │ DepositController.storeSubscriberToCashOutStore()
  │ ① LOCAL pincode check: passwordEncoder.matches(pinCode, session.pincode)
  │   FAIL → "Wrong Pin" → redirect dashboard (never leaves webclient)
  │   PASS ↓
  │ ② Builds TransferDto:
  │    fromAccountId = session.subscriberWalletId
  │    toAccountId = selected agent walletId
  │    amount = 5000 XAF
  │    pincode = "1234"
  │    transferInitiedById = session.userId
  │    channelType = WEB
  │ ③ restTemplate.postForObject(gateway + "/transfer/subscriber-cash-out", dto)
  │    Interceptors inject: "Authorization: Bearer <jwt>", x-trace-id, X-Forwarded-For
  ▼
mfs-api-gateway: POST /transfer/subscriber-cash-out
  │ ① ApiAuthorizationFilter: decrypt JWE → extract userId + roles → set SecurityContext
  │ ② @PreAuthorize: permissionService.check('DEPOSIT_SUBSCRIBER_CASH_OUT') → pass
  │ ③ TransferController.SubscriberCashOut()
  │    getPOSTApiResponse(restTemplate, backend + "/transfer/subscriber-cash-out", dto)
  │    Injects: X-User-ID, X-Trace-ID headers
  ▼
mfs-backend-new: POST /transfer/subscriber-cash-out
  │ TransferController.SubscriberCashOut()
  │ ① Map TransferDto → TransferEventDto
  │ ② Force transferType = SUBSCRIBER_CASH_OUT
  │ ③ Generate transferId = "SCO260427.1045.X9Y8Z7"
  │ ④ Load fromWallet (SUBSCRIBER) + toWallet (AGENT or CUSTOMER_CARE)
  │ ⑤ validateTransferRequestNew(dto)
  │    → AccountsVerificationRule: from=SUBSCRIBER ✓, to=AGENT/CUSTOMER_CARE ✓
  │    → PincodeVerificationRule: BCrypt.matches("1234", stored) ✓
  │    → AccountBalanceLimitValidaionRule: balance 10000 ≥ 5050 ✓
  │    → ThresholdValidationRule: 5000 within grade limits ✓
  │    → MaxMinBalanceLimitValidationRule: 4950 ≥ minBalance ✓
  │    → CoolDownValidationRule: not in cooldown ✓
  │
  ├── PATH A: toWallet.type == CUSTOMER_CARE
  │    InternalAgentTransferRequestsService.addNewInternalAgentTransferRequests()
  │    → Save to internal_agent_transfer_requests table (status=PENDING, expiry=+60min)
  │    → return "Success/PROCESSING" response
  │
  └── PATH B: toWallet.type == AGENT
       NewAccountTransferService.processTransaction(dto)
       ① Scale amount → 5000.0000 XAF
       ② Re-validate (second pass)
       ③ ServiceAndCommissionChargeService.calculateServiceAndCommissionCharge()
          → serviceCharge = 50 XAF, commission = [{AWALLET-COMM-001, 25 XAF}]
       ④ AccountTransferListener.initiateTransaction()
          → AccountTransfer saved: status=STARTED → DB write #1
       ⑤ RabbitMQ → "AccountTransferCreateEvent"
       ⑥ NewWalletService.updateWalletBalances()
          DEBIT subscriber: 5050 XAF
          → SQL: UPDATE Wallet SET amount=amount-5050, transfer_id='SCO...'
                 WHERE aggregate_id='SWALLET-SUB-001' AND transfer_id='SCI-PREV-001'
          → rowAffected check → DB write #2
          → RabbitMQ "MoneySubtractedEvent"
          CREDIT agent: 5000 XAF
          → SQL: UPDATE Wallet SET amount=amount+5000 WHERE aggregate_id='AWALLET-AGT-001'
          → DB write #3
          → RabbitMQ "MoneyAddedEvent"
          CREDIT commission: 25 XAF
          → SQL: UPDATE Wallet SET amount=amount+25 WHERE aggregate_id='AWALLET-COMM-001'
          → DB write #4
          → RabbitMQ "MoneyAddedEvent"
       ⑦ NewThirdPartyService.handleThirdPartyService() → no-op for SUBSCRIBER_CASH_OUT
       ⑧ finishTransaction(dto)
          → processTransaction(): save AccountTransferOperation rows (WITHDRAW+DEPOSIT) → DB write #5
          → updateApprovalDetail(): update AccountTransfer approver fields → DB write #6
          → updateTransactionStatus(): status = DONE → DB write #7
          → completeTransaction(): no-op for SUBSCRIBER_CASH_OUT
          → commissionInfoListener.saveCommissionData(): save CommissionInfo → DB write #8
          → promotionListener: check/apply promotion
          → fixTransferMetadataProcessor: finalize metadata
       ⑨ RabbitMQ → "AccountTransferDoneEvent"   (for notification)
       ⑩ RabbitMQ → "AccountTransferCompletedEvent" (for reporting)
       ⑪ return BaseResponseDto { status="Success", transferId="SCO260427.1045.X9Y8Z7" }
  ▼
mfs-api-gateway receives backend response
  │ status == "Success"
  │ → POST notification/user/send-cashout-approval-notification (for CUSTOMER_CARE path only)
  │ return response to webclient
  ▼
mfs-webclient: receives response
  │ statusCode == SUCCESS → flash "Transaction Success" → redirect /admin/dashboard
  ▼
BROWSER: Dashboard shows green success banner

══════════════════ ASYNC (via RabbitMQ) ══════════════════

mfs-notification consumes "AccountTransferDoneEvent"
  └─ Send SMS to subscriber: "You withdrew 5000 XAF, balance: 4950 XAF"
  └─ Send SMS/push to agent: "Received 5000 XAF from subscriber"

mfs-reporting consumes:
  "AccountTransferCreateEvent"   → newTransactionAuditReportService.accountTransferCreateEvent()
  "MoneySubtractedEvent"        → newTransactionReportingService.handleMoneySubtractedEvent()
  "MoneyAddedEvent" (x2)        → newTransactionReportingService.handleMoneyAddedEvent()
  "AccountTransferCompletedEvent" → reportProcessingService.accountTransferCompletedEvent()
     → CustomerTransactionReport saved to reporting DB
     → RecentTransaction for SUBSCRIBER_CASH_OUT updated for this subscriber-agent pair
```