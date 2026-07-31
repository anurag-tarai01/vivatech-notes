**MFS PLATFORM**

**Switch Account Validation (Name Enquiry)**

_End-to-End Implementation Plan_

Creation Phase & Payment Execution Phase

|   |   |
|---|---|
|Prepared for|Architecture Reference|
|Vivacom MFS Engineering Team|mfs-backend-new / mfs-api-gateway|

  

# **Executive Summary**

This document provides a comprehensive, production-ready implementation plan for integrating Third-Party Switch Account Validation (Name Enquiry) into the Vivacom MFS platform. Validation is enforced at two distinct lifecycle phases:

|   |   |
|---|---|
|**Phase 1**|Creation Phase — validated in the WebClient UI before an Employee or Vendor record is persisted to the database, and re-validated server-side inside saveEmployee() / saveVendor() as a hard gate.|
|**Phase 2**|Payment Execution Phase — validated server-side inside ThirdPartyController just before SalaryPaymentRequest approval dispatches the TransferEventDto to the GPayAccountTransferService event bus.|

|   |
|---|
|**Key Finding from Codebase Analysis:**<br><br>A dedicated proxy endpoint already exists in the backend: POST /third-party/validate-switch-account. This plan wires it correctly end-to-end: applies the correct CMR / XAF payload structure, enforces the orgCode rule (present for Bank "B", absent for Mobile "M"), and surfaces the validated name in the UI before final form submission.|

  

# **Part 1: Cross-Microservice Flow Analysis**

## **1.1  System Architecture Overview**

The MFS platform consists of three primary services relevant to this integration:

|   |   |
|---|---|
|**Service**|Role|
|mfs-api-gateway|JWT-protected reverse-proxy. All WebClient traffic enters here. Controllers call the backend via RestTemplate (coreServerAddress).|
|mfs-backend-new|Core business logic — ThirdPartyController, ThirdPartyService, GPayAccountTransferService, SwitchWalletTokenService, ThirdPartyValidationController.|
|External Switch|api.g-payment.net — OAuth2 token endpoint + transfer/validation endpoint. Accessed only from the backend, never from the frontend.|

## **1.2  Creation Phase Flow**

### **Sequence: WebClient → Gateway → Backend → Switch → DB**

1. User fills in Employee/Vendor form (account number, type, orgCode if Bank).

2. WebClient calls the INTERNAL proxy: POST /third-party/validate-switch-account (Gateway).

3. Gateway proxies to: POST /third-party/validate-switch-account on the backend (no new code — route already exists via ThirdPartyController chain).

4. Backend ThirdPartyValidationController authenticates against the Switch OAuth endpoint and calls transfer/validation?intent=disburse.

5. Switch returns {status: 0, name: "TCHEUTCHOUA FRIDE"} (Bank) or {status: 0, name: "N A null DUCAS..."} (Mobile Money).

6. If status != 0, the Gateway returns a 422 error and the UI blocks submission.

7. If status == 0, the UI auto-populates and locks the "Verified Name" field, then enables the Submit button.

8. User confirms and submits. Backend ThirdPartyService.saveEmployee() / saveVendor() performs a second server-side validation call before calling employeeRepository.save().

## **1.3  Payment Execution Phase Flow**

### **Sequence: Maker submits payroll → Checker approves → Backend validates each payee before firing event bus**

9. Maker calls POST /third-party/create-payroll-for-employee-in-bulk (or create-employee-payroll), creating a SalaryPaymentRequest with FileStatus.PENDING.

10. Checker calls POST /third-party/approve-payroll (via Gateway). This is the critical injection point.

11. Before updating FileStatus to APPROVED and iterating over SalaryPayment rows, a new validatePayeesBeforeApproval() method is called.

12. For each SalaryPayment where paymentMode == SWITCH, a Name Enquiry call is made to the Switch API.

13. If any payee fails validation, the approval is aborted and the Checker receives a structured error listing the failed payees.

14. If all payees pass, the original approval flow continues: status → APPROVED, then GPayAccountTransferService.processTransaction() fires the TransferEventDto per payee.

## **1.4  Frontend State Management (WebClient)**

The UI must enforce a strict state machine to prevent form submission before validation completes:

|   |   |   |
|---|---|---|
|**State**|**Submit Button**|**Verified Name Field**|
|**IDLE (no input)**|DISABLED|Empty|
|**TYPING (user edits account number)**|DISABLED — reset validation|Cleared|
|**VALIDATING (API call in-flight)**|DISABLED — show spinner|"Validating..."|
|**VALID (status == 0)**|ENABLED (green)|Populated + locked (read-only)|
|**INVALID (status != 0)**|DISABLED|Error message shown|

|   |
|---|
|**Frontend UX Rule:**<br><br>CRITICAL: Use debounce (500ms) on the account number input to avoid firing the validation API on every keystroke. Also debounce the orgCode field for Bank accounts — the validation must wait for both number AND orgCode to be filled.|

  

# **Part 2: The API Contract**

## **2.1  OAuth Token Handshake**

### **POST https://api.g-payment.net/switch/api/enterprise/oauth/token**

|   |
|---|
|**REQUEST**|
|# Production cURL<br><br>curl -X POST 'https://api.g-payment.net/switch/api/enterprise/oauth/token' \<br><br>  -H 'Content-Type: application/json' \<br><br>  -d '{<br><br>    "client_id":     "GPayTest",<br><br>    "client_secret": "9Zl31oy<~}1a",<br><br>    "grant_type":    "partner",<br><br>    "scope":         "GP"<br><br>  }'|

|   |
|---|
|**EXPECTED RESPONSE**|
|{<br><br>  "message":      "Going on well",<br><br>  "status":       0,<br><br>  "access_token": "eyJhbGciOiJIUzI1NiJ9.<payload>.<sig>",<br><br>  "token_type":   "Bearer",<br><br>  "expires_in":   3600<br><br>}|

|   |
|---|
|**CRITICAL BUG TO FIX:**<br><br>grant_type must be "partner" (NOT "client_credentials"). scope must be "GP". The existing ThirdPartyValidationController has a bug: it passes grant_type: "client_credentials" — this must be corrected to "partner".|

## **2.2  Bank Account Name Enquiry (type: "B")**

### **POST https://api.g-payment.net/switch/api/enterprise/transfer/validation?intent=disburse**

Bank accounts MUST include orgCode. The account object structure is:

|   |
|---|
|**BANK ACCOUNT REQUEST (type: "B")**|
|curl -X POST 'https://api.g-payment.net/switch/api/enterprise/transfer/validation?intent=disburse' \<br><br>  -H 'Content-Type: application/json' \<br><br>  -H 'Authorization: Bearer <access_token>' \<br><br>  -d '{<br><br>    "account": {<br><br>      "orgCode":      "CCA",<br><br>      "countryCode":  "CMR",<br><br>      "number":       "00856171601",<br><br>      "type":         "B",<br><br>      "amount":       1000,<br><br>      "currencyCode": "XAF"<br><br>    }<br><br>  }'|

|   |
|---|
|**EXPECTED RESPONSE**|
|{<br><br>  "message":         "WOW",<br><br>  "status":          0,<br><br>  "name":            "TCHEUTCHOUA FRIDE",<br><br>  "amount":          1000,<br><br>  "expectedFee":     0,<br><br>  "amountBreakdown": []<br><br>}|

## **2.3  Mobile Money Name Enquiry (type: "M")**

Mobile Money accounts MUST OMIT orgCode entirely. Including it will cause the Switch API to reject the request or return an error.

|   |
|---|
|**MOBILE MONEY REQUEST (type: "M") — NO orgCode**|
|curl -X POST 'https://api.g-payment.net/switch/api/enterprise/transfer/validation?intent=disburse' \<br><br>  -H 'Content-Type: application/json' \<br><br>  -H 'Authorization: Bearer <access_token>' \<br><br>  -d '{<br><br>    "account": {<br><br>      "countryCode":  "CMR",<br><br>      "number":       "677061344",<br><br>      "type":         "M",<br><br>      "amount":       1000,<br><br>      "currencyCode": "XAF"<br><br>    }<br><br>  }'|

|   |
|---|
|**EXPECTED RESPONSE**|
|{<br><br>  "message":         "Excellent",<br><br>  "status":          0,<br><br>  "name":            "N A null DUCAS FUEN EPOUSE WANYEH DUCAS FUEN EPOUSE WANYEH",<br><br>  "amount":          1000,<br><br>  "expectedFee":     0,<br><br>  "amountBreakdown": []<br><br>}|

|   |   |   |   |   |
|---|---|---|---|---|
|**Field**|**Bank (B)**|**Mobile (M)**|**Required**|**Notes**|
|**orgCode**|✅ REQUIRED|❌ MUST OMIT|Conditional|Omitting from B or adding to M causes failure|
|**countryCode**|"CMR"|"CMR"|Always|Hardcoded|
|**number**|Bank Acc No|Phone No|Always||
|**type**|"B"|"M"|Always|Drives orgCode rule|
|**amount**|1000|1000|Always|Use 1000 XAF for validation calls|
|**currencyCode**|"XAF"|"XAF"|Always|Hardcoded|

  

# **Part 3: Implementation Strategy**

## **3.1  Fix: ThirdPartyValidationController (Backend)**

File: mfs-backend-new/application/src/main/java/com/vivacom/mfs/application/controller/ThirdPartyValidationController.java

Two critical corrections are required in the existing controller:

• grant_type must be "partner" (currently "client_credentials" — will always fail in production).

• The SwitchValidationRequest must pass countryCode and currencyCode inside the account object (currently passing currency at root level as "GHS" — must be "XAF" inside account).

• The @RequestBody annotation on validateTransaction must be removed (Feign does not use Spring's @RequestBody — the body is already inferred from the parameter type).

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **ThirdPartyValidationController.java (CORRECTED)**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| // FILE: ThirdPartyValidationController.java<br><br>// PATH: .../application/controller/ThirdPartyValidationController.java<br><br>@RestController<br><br>@RequestMapping("/third-party")<br><br>public class ThirdPartyValidationController {<br><br>    private final SwitchWalletClient switchWalletClient;<br><br>    @Value("${switch.api.test-mode}")<br><br>    private boolean isTestMode;<br><br>    @Value("${switch.api.test-client-id}")<br><br>    private String testClientId;<br><br>    @Value("${switch.api.test-client-secret}")<br><br>    private String testClientSecret;<br><br>    @Value("${switch.api.prod-client-id}")<br><br>    private String prodClientId;<br><br>    @Value("${switch.api.prod-client-secret}")<br><br>    private String prodClientSecret;<br><br>    public ThirdPartyValidationController(SwitchWalletClient switchWalletClient) {<br><br>        this.switchWalletClient = switchWalletClient;<br><br>    }<br><br>    @PostMapping("/validate-switch-account")<br><br>    public ResponseEntity<SwitchNameEnquiryResponseDto> validateSwitchAccount(<br><br>            @RequestBody SwitchNameEnquiryRequestDto requestDto) {<br><br>        // Step 1: Authenticate<br><br>        Map<String, String> authBody = new HashMap<>();<br><br>        authBody.put("client_id",     isTestMode ? testClientId : prodClientId);<br><br>        authBody.put("client_secret", isTestMode ? testClientSecret : prodClientSecret);<br><br>        authBody.put("grant_type",    "partner");    // MUST be "partner"<br><br>        authBody.put("scope",         "GP");         // MUST be "GP"<br><br>        SwitchAuthResponse authResponse = switchWalletClient.authenticate(authBody);<br><br>        if (authResponse == null \| authResponse.getAccess_token() == null) {<br><br>            return ResponseEntity.status(503).body(<br><br>                SwitchNameEnquiryResponseDto.builder()<br><br>                    .status(-1).message("Switch authentication failed").build());<br><br>        }<br><br>        String bearerToken = "Bearer " + authResponse.getAccess_token();<br><br>        // Step 2: Build account object — orgCode only for type "B"<br><br>        SwitchAccount.SwitchAccountBuilder acctBuilder = SwitchAccount.builder()<br><br>            .countryCode("CMR")<br><br>            .number(requestDto.getNumber())<br><br>            .type(requestDto.getType())<br><br>            .amount(requestDto.getAmount() != null ? requestDto.getAmount() : "1000")<br><br>            .currencyCode("XAF");<br><br>        if ("B".equalsIgnoreCase(requestDto.getType())) {<br><br>            if (requestDto.getOrgCode() == null \| requestDto.getOrgCode().isBlank()) {<br><br>                return ResponseEntity.badRequest().body(<br><br>                    SwitchNameEnquiryResponseDto.builder()<br><br>                        .status(-1).message("orgCode is required for Bank accounts").build());<br><br>            }<br><br>            acctBuilder.orgCode(requestDto.getOrgCode());<br><br>        }<br><br>        // For type "M": do NOT set orgCode — @JsonInclude(NON_NULL) handles serialization<br><br>        SwitchValidationRequest validationRequest = SwitchValidationRequest.builder()<br><br>            .account(acctBuilder.build())<br><br>            .build();<br><br>        // Step 3: Call validation<br><br>        SwitchValidationResponse response =<br><br>            switchWalletClient.validateTransaction(bearerToken, validationRequest);<br><br>        if (response == null) {<br><br>            return ResponseEntity.status(502).body(<br><br>                SwitchNameEnquiryResponseDto.builder()<br><br>                    .status(-1).message("No response from Switch").build());<br><br>        }<br><br>        int status = response.getStatus() != null ?<br><br>            Integer.parseInt(response.getStatus()) : -1;<br><br>        return ResponseEntity.ok(<br><br>            SwitchNameEnquiryResponseDto.builder()<br><br>                .status(status)<br><br>                .message(response.getMessage())<br><br>                .verifiedName(status == 0 ? response.getName() : null)<br><br>                .build());<br><br>    }<br><br>} |

## **3.2  New DTOs Required**

### **SwitchNameEnquiryRequestDto.java**

File: mfs-backend-new/core-api/src/main/java/com/vivacom/mfs/common/dto/remittance/SwitchNameEnquiryRequestDto.java

|   |
|---|
|**SwitchNameEnquiryRequestDto.java**|
|@Data @Builder @NoArgsConstructor @AllArgsConstructor<br><br>@JsonInclude(JsonInclude.Include.NON_NULL)<br><br>public class SwitchNameEnquiryRequestDto {<br><br>    @NotBlank(message = "account number is required")<br><br>    private String number;<br><br>    @NotBlank(message = "type must be B or M")<br><br>    @Pattern(regexp = "^[BM]$", message = "type must be B or M")<br><br>    private String type;<br><br>    private String orgCode;    // Required when type=B, must be absent when type=M<br><br>    private String amount;     // Defaults to "1000" in controller if null<br><br>}|

### **SwitchNameEnquiryResponseDto.java**

|   |
|---|
|**SwitchNameEnquiryResponseDto.java**|
|@Data @Builder @NoArgsConstructor @AllArgsConstructor<br><br>public class SwitchNameEnquiryResponseDto {<br><br>    private int     status;<br><br>    private String  message;<br><br>    private String  verifiedName;   // Only populated on status == 0<br><br>}|

### **SwitchAccount.java — Add missing fields**

File: mfs-backend-new/core-api/.../remittance/SwitchAccount.java

The existing SwitchAccount is missing countryCode, amount, and currencyCode. These fields must be added:

|   |
|---|
|**SwitchAccount.java (UPDATED)**|
|@Data @Builder<br><br>@JsonInclude(JsonInclude.Include.NON_NULL)<br><br>public class SwitchAccount {<br><br>    private String orgCode;       // Bank only — @JsonInclude handles null exclusion<br><br>    private String countryCode;   // Always "CMR"<br><br>    private String number;<br><br>    private String type;          // "B" or "M"<br><br>    private String amount;<br><br>    private String currencyCode;  // Always "XAF"<br><br>    private String callbackURL;   // Used for disburse operations, not validation<br><br>}|

## **3.3  Fix SwitchWalletClient Feign Interface**

File: mfs-backend-new/core-api/.../remittance/SwitchWalletClient.java

The validateTransaction method incorrectly uses @RequestBody (Spring MVC annotation). Feign resolves the request body from the method parameter type — remove it:

|   |
|---|
|**SwitchWalletClient.java (CORRECTED)**|
|public interface SwitchWalletClient {<br><br>    // ... existing methods ...<br><br>    @RequestLine("POST /switch/api/enterprise/oauth/token")<br><br>    @Headers("Content-Type: application/json")<br><br>    SwitchAuthResponse authenticate(Map<String, String> body);<br><br>    // Changed from Map<String, ?> to Map<String, String> for type safety<br><br>    @RequestLine("POST /switch/api/enterprise/transfer/validation?intent=disburse")<br><br>    @Headers({<br><br>        "Content-Type: application/json",<br><br>        "Authorization: {token}"<br><br>    })<br><br>    SwitchValidationResponse validateTransaction(<br><br>        @Param("token") String token,<br><br>        SwitchValidationRequest request    // REMOVED @RequestBody<br><br>    );<br><br>}|

## **3.4  Application Properties Update**

File: mfs-backend-new/application/src/main/resources/application.properties

|   |
|---|
|**application.properties**|
|# Switch Name Enquiry / Account Validation<br><br>switch.api.test-mode=true<br><br>switch.api.test-client-id=GPayTest<br><br>switch.api.test-client-secret=9Zl31oy<~}1a<br><br>switch.api.prod-client-id=<br><br>switch.api.prod-client-secret=<br><br># Switch base URL (already exists — keep as-is):<br><br># mfs.switch_wallet.api.baseurl=https://sandbox.g-payment.net  (test)<br><br># mfs.switch_wallet.api.baseurl=https://api.g-payment.net      (prod)|

## **3.5  Creation Phase: ThirdPartyService Injection Points**

### **File: ThirdPartyService.java — saveEmployee() method**

Inject SwitchNameEnquiryService (a new thin service wrapping the validation call) as a dependency, and call it before employeeRepository.save():

|   |
|---|
|**ThirdPartyService.java — saveEmployee() (MODIFIED)**|
|// FILE: ThirdPartyService.java<br><br>// Inject the new validation service<br><br>@Autowired<br><br>private SwitchNameEnquiryService switchNameEnquiryService;<br><br>public BaseResponseDto saveEmployee(CreateEmployeeDto dto) {<br><br>    // ... existing WALLET mode checks ...<br><br>    // ─── SWITCH mode: perform Name Enquiry before saving ───────────<br><br>    if (dto.getPaymentMode() == PaymentMode.SWITCH) {<br><br>        SwitchNameEnquiryResponseDto enquiry =<br><br>            switchNameEnquiryService.validateAccount(<br><br>                dto.getAccountNumber(),<br><br>                dto.getSwitchPaymentType(),   // "B" or "M"<br><br>                dto.getOrgCode()              // null for Mobile, required for Bank<br><br>            );<br><br>        if (enquiry.getStatus() != 0) {<br><br>            log.warn("Employee account validation failed: {}", enquiry.getMessage());<br><br>            return BaseResponseDto.builder()<br><br>                .statusCode(StatusCode.FAILED)<br><br>                .message("Account validation failed: " + enquiry.getMessage())<br><br>                .build();<br><br>        }<br><br>        // Persist the verified name returned by the Switch<br><br>        dto.setName(enquiry.getVerifiedName());<br><br>        log.info("Employee account validated. Verified name: {}", enquiry.getVerifiedName());<br><br>    }<br><br>    // ─── end validation ─────────────────────────────────────────────<br><br>    employee.setName(dto.getName());<br><br>    // ... rest of existing save logic ...<br><br>    employeeRepository.save(employee);<br><br>    return BaseResponseDto.builder()<br><br>        .statusCode(StatusCode.SUCCESS)<br><br>        .message("Successfully created new employee")<br><br>        .build();<br><br>}|

Apply the exact same pattern to saveVendor():

|   |
|---|
|**ThirdPartyService.java — saveVendor() (MODIFIED)**|
|public BaseResponseDto saveVendor(CreateVendorDto dto) {<br><br>    // ...<br><br>    if (dto.getPaymentMode() == PaymentMode.SWITCH) {<br><br>        SwitchNameEnquiryResponseDto enquiry =<br><br>            switchNameEnquiryService.validateAccount(<br><br>                dto.getAccountNumber(),<br><br>                dto.getSwitchPaymentType(),<br><br>                dto.getOrgCode()<br><br>            );<br><br>        if (enquiry.getStatus() != 0) {<br><br>            return BaseResponseDto.builder()<br><br>                .statusCode(StatusCode.FAILED)<br><br>                .message("Vendor account validation failed: " + enquiry.getMessage())<br><br>                .build();<br><br>        }<br><br>        dto.setName(enquiry.getVerifiedName());<br><br>    }<br><br>    // ... rest of vendor save ...<br><br>}|

## **3.6  New Service: SwitchNameEnquiryService**

File: mfs-backend-new/application/src/main/java/com/vivacom/mfs/application/service/SwitchNameEnquiryService.java

This service is the single reusable point for all Name Enquiry calls across both phases. It encapsulates authentication, payload construction, and error handling:

|   |
|---|
|**SwitchNameEnquiryService.java (NEW)**|
|@Service<br><br>@Slf4j<br><br>public class SwitchNameEnquiryService {<br><br>    @Autowired<br><br>    private SwitchWalletClient switchWalletClient;<br><br>    @Value("${switch.api.test-mode}")<br><br>    private boolean testMode;<br><br>    @Value("${switch.api.test-client-id}")<br><br>    private String testClientId;<br><br>    @Value("${switch.api.test-client-secret}")<br><br>    private String testClientSecret;<br><br>    @Value("${switch.api.prod-client-id}")<br><br>    private String prodClientId;<br><br>    @Value("${switch.api.prod-client-secret}")<br><br>    private String prodClientSecret;<br><br>    /**<br><br>     * Performs a Switch Name Enquiry (Account Validation).<br><br>     *<br><br>     * @param accountNumber  the account number or mobile phone number<br><br>     * @param type           "B" for Bank, "M" for Mobile Money<br><br>     * @param orgCode        required for Bank (type=B), must be null for Mobile (type=M)<br><br>     * @return SwitchNameEnquiryResponseDto with status, message, and verifiedName<br><br>     */<br><br>    public SwitchNameEnquiryResponseDto validateAccount(<br><br>            String accountNumber, String type, String orgCode) {<br><br>        log.info("[SwitchNameEnquiry] Validating account: {} type: {}", accountNumber, type);<br><br>        try {<br><br>            // 1. Obtain Bearer token<br><br>            Map<String, String> authBody = Map.of(<br><br>                "client_id",     testMode ? testClientId : prodClientId,<br><br>                "client_secret", testMode ? testClientSecret : prodClientSecret,<br><br>                "grant_type",    "partner",<br><br>                "scope",         "GP"<br><br>            );<br><br>            SwitchAuthResponse auth = switchWalletClient.authenticate(authBody);<br><br>            if (auth == null \| auth.getAccess_token() == null) {<br><br>                throw new MFSException("Switch token fetch failed");<br><br>            }<br><br>            // 2. Build account payload — strict structural rules<br><br>            SwitchAccount.SwitchAccountBuilder acct = SwitchAccount.builder()<br><br>                .countryCode("CMR")<br><br>                .number(accountNumber)<br><br>                .type(type)<br><br>                .amount("1000")<br><br>                .currencyCode("XAF");<br><br>            if ("B".equalsIgnoreCase(type)) {<br><br>                if (orgCode == null \| orgCode.isBlank()) {<br><br>                    return SwitchNameEnquiryResponseDto.builder()<br><br>                        .status(-1)<br><br>                        .message("orgCode required for Bank accounts")<br><br>                        .build();<br><br>                }<br><br>                acct.orgCode(orgCode);<br><br>                // type M: orgCode is NOT set — @JsonInclude(NON_NULL) omits it<br><br>            }<br><br>            SwitchValidationRequest req = SwitchValidationRequest.builder()<br><br>                .account(acct.build())<br><br>                .build();<br><br>            // 3. Call validation<br><br>            SwitchValidationResponse resp =<br><br>                switchWalletClient.validateTransaction(<br><br>                    "Bearer " + auth.getAccess_token(), req);<br><br>            int status = resp != null && resp.getStatus() != null<br><br>                ? Integer.parseInt(resp.getStatus()) : -1;<br><br>            log.info("[SwitchNameEnquiry] Result: status={} name={}",<br><br>                status, resp != null ? resp.getName() : "null");<br><br>            return SwitchNameEnquiryResponseDto.builder()<br><br>                .status(status)<br><br>                .message(resp != null ? resp.getMessage() : "No response")<br><br>                .verifiedName(status == 0 ? resp.getName() : null)<br><br>                .build();<br><br>        } catch (Exception e) {<br><br>            log.error("[SwitchNameEnquiry] Validation error: {}", e.getMessage(), e);<br><br>            return SwitchNameEnquiryResponseDto.builder()<br><br>                .status(-1)<br><br>                .message("Validation service unavailable: " + e.getMessage())<br><br>                .build();<br><br>        }<br><br>    }<br><br>}|

## **3.7  Payment Execution Phase: Injection into approve-payroll**

### **File: ThirdPartyController.java — approvePayroll() method**

The approve-payroll endpoint currently calls thirdPartyService.approvePayroll(uuid, submittedById) which immediately sets FileStatus.APPROVED. The validation gate is injected BEFORE this call:

|   |
|---|
|**ThirdPartyController.java — approvePayroll() (MODIFIED)**|
|// FILE: ThirdPartyController.java<br><br>@Autowired<br><br>private SwitchNameEnquiryService switchNameEnquiryService;<br><br>@RequestMapping(value = "/approve-payroll", method = RequestMethod.POST)<br><br>public Object approvePayroll(<br><br>        @RequestBody PayrollVerificationRequestDto requestDto) throws Exception {<br><br>    String uuid = requestDto.getUuid();<br><br>    SalaryPaymentRequest payroll = salaryPaymentRequestRepository.findByUuid(uuid);<br><br>    if (payroll == null) {<br><br>        return BaseResponseDto.builder()<br><br>            .statusCode(StatusCode.FAILED).message("Payroll not found").build();<br><br>    }<br><br>    if (!payroll.getStatus().equals(FileStatus.PENDING)) {<br><br>        return BaseResponseDto.builder()<br><br>            .statusCode(StatusCode.FAILED)<br><br>            .message("Payroll is not in PENDING state").build();<br><br>    }<br><br>    // ─── PAYMENT PHASE VALIDATION: Validate all SWITCH payees ────────<br><br>    List<SalaryPayment> payments = salaryPaymentRepository.findAllByBulkPaymentId(uuid);<br><br>    List<String> validationFailures = new ArrayList<>();<br><br>    for (SalaryPayment payment : payments) {<br><br>        // Only validate SWITCH payment modes (WALLET is internal, no Switch call needed)<br><br>        if (PaymentMode.SWITCH.equals(payment.getPaymentMode())) {<br><br>            SwitchNameEnquiryResponseDto enquiry =<br><br>                switchNameEnquiryService.validateAccount(<br><br>                    payment.getMsisdn(),<br><br>                    payment.getSwitchPaymentType(),   // "B" or "M"<br><br>                    payment.getOrgCode()              // null for Mobile<br><br>                );<br><br>            if (enquiry.getStatus() != 0) {<br><br>                validationFailures.add(String.format(<br><br>                    "[%s - %s]: %s",<br><br>                    payment.getName(), payment.getMsisdn(), enquiry.getMessage()<br><br>                ));<br><br>                log.warn("[PaymentPhase] Payee validation failed: {} - {}",<br><br>                    payment.getMsisdn(), enquiry.getMessage());<br><br>            }<br><br>        }<br><br>    }<br><br>    if (!validationFailures.isEmpty()) {<br><br>        String errorDetail = String.join("; ", validationFailures);<br><br>        log.error("[PaymentPhase] Approval blocked. {} payee(s) failed: {}",<br><br>            validationFailures.size(), errorDetail);<br><br>        return BaseResponseDto.builder()<br><br>            .statusCode(StatusCode.FAILED)<br><br>            .message("Payroll approval blocked. " + validationFailures.size() +<br><br>                     " payee(s) failed validation: " + errorDetail)<br><br>            .build();<br><br>    }<br><br>    // ─── end validation ───────────────────────────────────────────────<br><br>    // Proceed with original approval logic<br><br>    thirdPartyService.approvePayroll(uuid, requestDto.getSubmittedById());<br><br>    // Continue firing TransferEventDto per payee...<br><br>    // (existing logic unchanged below this point)<br><br>    return BaseResponseDto.builder()<br><br>        .statusCode(StatusCode.SUCCESS).message("Payroll approved successfully").build();<br><br>}|

|   |
|---|
|**Performance Consideration:**<br><br>For bulk payrolls with many SWITCH payees, validation introduces N sequential HTTP calls to the Switch API. If the payroll has >20 SWITCH payees, consider parallelizing the validation using CompletableFuture.allOf() with a bounded ExecutorService (e.g., 5 threads) to keep approval latency under 5 seconds.|

### **SalaryPayment Entity — Ensure SWITCH fields are present**

File: mfs-backend-new/wallet-query/src/main/java/com/vivacom/mfs/wallet/query/entity/SalaryPayment.java

Confirm these fields exist on the SalaryPayment entity (they should already exist based on the codebase, but verify):

|   |
|---|
|**SalaryPayment.java — Required fields**|
|// These fields must exist on SalaryPayment for the payment-phase validation:<br><br>private PaymentMode paymentMode;      // WALLET or SWITCH<br><br>private String switchPaymentType;     // "B" or "M" — from employee.switchPaymentType<br><br>private String orgCode;               // Bank org code — from employee.orgCode|

## **3.8  API Gateway: Route the Validate Endpoint**

File: mfs-api-gateway/.../controllers/ThirdPartyController.java

The gateway's ThirdPartyController must proxy /third-party/validate-switch-account to the backend. Add this method:

|   |
|---|
|**Gateway ThirdPartyController.java — add route**|
|// FILE: mfs-api-gateway ThirdPartyController.java<br><br>@RequestMapping(value = "/validate-switch-account", method = RequestMethod.POST)<br><br>public Object validateSwitchAccount(@RequestBody Object requestDto) {<br><br>    String uri = coreServerAddress + "third-party/validate-switch-account";<br><br>    return getPOSTApiResponse(restTemplate, uri, requestDto, Object.class);<br><br>}|

|   |
|---|
|**Security Note:**<br><br>Ensure SecurityConfig in the gateway does NOT block /third-party/validate-switch-account. It should require authentication (ThirdParty Admin JWT) but not a specific RBAC permission, as it is called interactively during form fill.|

## **3.9  WebClient UI Integration**

### **Endpoint to call from the WebClient**

|   |
|---|
|**WebClient API Call**|
|POST {gatewayBaseUrl}/third-party/validate-switch-account<br><br>Authorization: Bearer <thirdPartyAdminJwt><br><br>Content-Type: application/json<br><br>// BANK account:<br><br>{<br><br>  "number":  "00856171601",<br><br>  "type":    "B",<br><br>  "orgCode": "CCA"<br><br>}<br><br>// MOBILE MONEY account:<br><br>{<br><br>  "number": "677061344",<br><br>  "type":   "M"<br><br>}|

### **JavaScript/TypeScript: Validation hook (React / Angular / Vue compatible)**

|   |
|---|
|**WebClient — Validation Hook (React example)**|
|// switchValidation.service.ts (or .js)<br><br>interface ValidationResult {<br><br>  status: number;<br><br>  message: string;<br><br>  verifiedName: string \| null;<br><br>}<br><br>async function validateSwitchAccount(<br><br>  number: string,<br><br>  type: 'B' \| 'M',<br><br>  orgCode?: string<br><br>): Promise<ValidationResult> {<br><br>  const payload: Record<string, string> = { number, type };<br><br>  if (type === 'B') {<br><br>    if (!orgCode) throw new Error('orgCode required for Bank accounts');<br><br>    payload.orgCode = orgCode;<br><br>  }<br><br>  // type=M: orgCode intentionally omitted<br><br>  const response = await fetch(`${API_BASE}/third-party/validate-switch-account`, {<br><br>    method: 'POST',<br><br>    headers: {<br><br>      'Content-Type': 'application/json',<br><br>      'Authorization': `Bearer ${getToken()}`<br><br>    },<br><br>    body: JSON.stringify(payload)<br><br>  });<br><br>  if (!response.ok) {<br><br>    throw new Error(`Validation request failed: ${response.status}`);<br><br>  }<br><br>  return response.json();<br><br>}<br><br>// ─── Usage in a React component ────────────────────────────────────<br><br>const [verifiedName, setVerifiedName]   = useState('');<br><br>const [isValidating, setIsValidating]   = useState(false);<br><br>const [isValidated,  setIsValidated]    = useState(false);<br><br>const [validationError, setValidationError] = useState('');<br><br>// Debounced handler — fire after 500ms of no typing<br><br>const debouncedValidate = useMemo(<br><br>  () => debounce(async (number, type, orgCode) => {<br><br>    setIsValidating(true);<br><br>    setIsValidated(false);<br><br>    setVerifiedName('');<br><br>    setValidationError('');<br><br>    try {<br><br>      const result = await validateSwitchAccount(number, type, orgCode);<br><br>      if (result.status === 0) {<br><br>        setVerifiedName(result.verifiedName ?? '');<br><br>        setIsValidated(true);<br><br>      } else {<br><br>        setValidationError(result.message);<br><br>      }<br><br>    } catch (e) {<br><br>      setValidationError('Validation service unavailable.');<br><br>    } finally {<br><br>      setIsValidating(false);<br><br>    }<br><br>  }, 500),<br><br>  []<br><br>);<br><br>// On account number or orgCode change:<br><br>useEffect(() => {<br><br>  setIsValidated(false);       // reset on every change<br><br>  if (accountNumber.length > 5) {<br><br>    if (paymentType === 'B' && !orgCode) return;  // wait for orgCode<br><br>    debouncedValidate(accountNumber, paymentType, orgCode);<br><br>  }<br><br>}, [accountNumber, orgCode, paymentType]);<br><br>// Submit button: only enabled when validated<br><br><button disabled={!isValidated \| isValidating} onClick={handleSubmit}><br><br>  {isValidating ? 'Validating...' : 'Submit'}<br><br></button><br><br>// Verified name field: read-only, auto-populated<br><br><input<br><br>  type="text"<br><br>  value={isValidating ? 'Validating...' : verifiedName}<br><br>  readOnly={isValidated}<br><br>  placeholder={validationError \| 'Will auto-populate after validation'}<br><br>  style={{ borderColor: isValidated ? 'green' : validationError ? 'red' : undefined }}<br><br>/>|

  

# **Part 4: Implementation Checklist & File Summary**

## **4.1  Files to Create (New)**

|   |   |
|---|---|
|**File**|**Purpose**|
|SwitchNameEnquiryService.java|Reusable service — auth + payload construction + validation call|
|SwitchNameEnquiryRequestDto.java|Inbound DTO for the proxy endpoint (number, type, orgCode)|
|SwitchNameEnquiryResponseDto.java|Outbound DTO (status, message, verifiedName)|

## **4.2  Files to Modify (Existing)**

|   |   |
|---|---|
|**File**|**Change Required**|
|ThirdPartyValidationController.java|Fix grant_type, fix account DTO, remove @RequestBody from Feign call|
|SwitchWalletClient.java|Remove @RequestBody from validateTransaction; fix Map type|
|SwitchAccount.java|Add countryCode, amount, currencyCode fields|
|SwitchValidationRequest.java|Remove unused fields (amount, currency, reference, description at root)|
|ThirdPartyService.java|Inject SwitchNameEnquiryService; add validation gate in saveEmployee() and saveVendor()|
|ThirdPartyController.java (backend)|Add approvePayroll() validation loop; inject SwitchNameEnquiryService|
|ThirdPartyController.java (gateway)|Add /validate-switch-account proxy route|
|application.properties (backend)|Add switch.api.test-client-id and switch.api.test-client-secret|

## **4.3  Critical Invariants — Never Violate**

|   |
|---|
|**IMMUTABLE API RULES — FROM SWITCH NETWORK CONTRACT:**<br><br>1. orgCode MUST be present in the account object when type="B" (Bank). 2. orgCode MUST be absent from the account object when type="M" (Mobile Money). @JsonInclude(NON_NULL) on SwitchAccount guarantees this if orgCode field is left null. 3. countryCode is always "CMR". currencyCode is always "XAF". 4. grant_type is always "partner". scope is always "GP". 5. The Switch validation endpoint path is always: /switch/api/enterprise/transfer/validation?intent=disburse. 6. A status value of 0 in the Switch response means SUCCESS. Any other value (including null) is FAILURE.|

  

# **Appendix: SwitchValidationRequest Clean-up**

The current SwitchValidationRequest has extra fields that are not part of the Switch validation payload (amount, currency, reference, description at root level). These should be removed to avoid polluting the JSON sent to the Switch API. The validation payload is exclusively determined by the nested account object:

|   |
|---|
|**SwitchValidationRequest.java (CORRECTED)**|
|// FILE: SwitchValidationRequest.java (CORRECTED)<br><br>@Data @Builder<br><br>@JsonInclude(JsonInclude.Include.NON_NULL)<br><br>public class SwitchValidationRequest {<br><br>    private SwitchAccount account;<br><br>    // Root-level amount, currency, reference, description REMOVED.<br><br>    // All validation parameters are inside the account object.<br><br>}|

## **Token Caching Recommendation**

The Switch access_token has a 3600-second (1 hour) TTL. To avoid one auth call per validation, implement a simple in-memory token cache in SwitchNameEnquiryService using a volatile field with an expiry timestamp:

|   |
|---|
|**Token Caching (Recommended Optimization)**|
|// Inside SwitchNameEnquiryService — optional but recommended for bulk payrolls<br><br>private volatile String cachedToken;<br><br>private volatile long   tokenExpiresAt = 0L;<br><br>private String getBearerToken() {<br><br>    long now = System.currentTimeMillis();<br><br>    if (cachedToken != null && now < tokenExpiresAt - 60_000L) {<br><br>        return cachedToken;  // reuse if >60s remaining<br><br>    }<br><br>    Map<String, String> body = Map.of(<br><br>        "client_id",     testMode ? testClientId : prodClientId,<br><br>        "client_secret", testMode ? testClientSecret : prodClientSecret,<br><br>        "grant_type",    "partner",<br><br>        "scope",         "GP"<br><br>    );<br><br>    SwitchAuthResponse auth = switchWalletClient.authenticate(body);<br><br>    cachedToken      = auth.getAccess_token();<br><br>    tokenExpiresAt   = now + 3_600_000L;   // 1 hour in ms<br><br>    return cachedToken;<br><br>}<br><br>// In validateAccount(): replace the auth block with:<br><br>// String bearerToken = "Bearer " + getBearerToken();|

End of Implementation Plan — mfs-backend-new  |  mfs-api-gateway  |  Switch Network: api.g-payment.net