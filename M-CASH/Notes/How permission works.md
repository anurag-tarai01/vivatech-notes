## How Permissions Work — Complete Flow, Client to DB

---

### The Big Picture First

There are **two completely separate permission systems** working at the same time in this codebase. This is the single most important thing to understand before reading any menu code:

```
┌──────────────────────────────────────────────────────────────────────┐
│  SYSTEM 1: Gateway Permission Check                                  │
│  Where: mfs-api-gateway                                              │
│  What: Guards every backend API endpoint                             │
│  How: @PreAuthorize("@permissionService.check('PRIVILEGE_NAME')")    │
│  Used by: Every controller in the gateway                            │
├──────────────────────────────────────────────────────────────────────┤
│  SYSTEM 2: Webclient Menu/UI Permission Check                        │
│  Where: mfs-webclient (Thymeleaf + Spring Security)                  │
│  What: Shows/hides menu items and page buttons                       │
│  How: sec:authorize="@permissionService.check('PRIVILEGE_NAME')"     │
│  Used by: sidebar-left.html and every page template                  │
└──────────────────────────────────────────────────────────────────────┘
```

Both use a `PermissionService.check()` method. Both look **identical in name** but they are **different classes in different projects**. Both however work the **same way** — they look at the current user's granted authorities in the Spring Security context.

---

### Step 1 — Where Permissions Live in the Database

The gateway's SQL Server database (same `MFS_Live_CMR` database used by everything) has these tables:

```
┌─────────────┐         ┌─────────────────┐         ┌───────────────┐
│   Role      │         │ roles_privileges │         │  Privilege    │
│─────────────│         │─────────────────│         │───────────────│
│ id (PK)     │──┐  ┌──►│ role_id (FK)    │◄──┐ ┌──│ id (PK)       │
│ name        │  └──┘   │ privilege_id(FK)│   └─┘  │ name          │
│ displayName │         └─────────────────┘        │ displayName   │
└─────────────┘                                    └───────────────┘
       │                                                  │
       │ users_roles (JoinTable)        users_privileges  │
       ▼                                (JoinTable)       ▼
┌─────────────┐                              ┌─────────────────┐
│   User      │──────────────────────────────│ users_privileges│
│─────────────│                              └─────────────────┘
│ id (PK)     │
│ msisdn      │  ← One row per subscriber/agent/merchant/admin
│ type        │    in the gateway's User table
└─────────────┘
```

**In plain English:**

- A `Role` is a named group: e.g., `"ROLE_INTERNAL_AGENT"`, `"ROLE_SUPER_ADMIN"`, `"MyCash Manager"`
- A `Privilege` is a specific permission string: e.g., `"AGENT_CREATE"`, `"AGENT_VIEW_ACTIVE"`, `"CHILD_AGENT_ACTIVE_LIST"`
- A `Role` can contain many `Privilege`s (via `roles_privileges` join table)
- A `User` can have many `Role`s (via `users_roles` join table)
- A `User` can also have individual `Privilege`s directly (via `users_privileges` join table — rare, usually done via roles)

The Redis cache (`CachedRoleInfo`) mirrors this: `role name → [list of privilege names]`. The gateway reads from Redis on every request instead of hitting SQL Server.

---

### Step 2 — Login: How Permissions Get Into the Browser Session

This is the **entire foundation**. Everything downstream depends on what happens here.

#### Path A — Admin Login (`/admin/post-login` → `/admin/verify-admin-otp`)

```
LoginController.postLogin()
  │  POST /authenticate-admin to gateway
  │  Gateway checks email + password
  │  Returns: { status: "OTP sent" }
  ▼
LoginController.verifyAdminOtp()
  │  POST /verify-admin-otp to gateway
  │  
  │  Gateway.SecurityController.verifyAdminOtp():
  │    1. Finds admin in DB by email
  │    2. Loads admin.getRoles()       → e.g. [Role("MyCash Manager")]
  │    3. Loads admin.getPrivileges()  → e.g. [Privilege("EXTRA_PRIV")]
  │    4. Calls rolesAndPermissionsAsString(roles, privileges):
  │         iterates each Role → adds role.getName()
  │         iterates each role.getPrivileges() → adds privilege.getName()
  │         iterates direct privileges → adds privilege.getName()
  │         Result: ["MyCash Manager", "AGENT_CREATE", "AGENT_VIEW_ACTIVE",
  │                  "SUBSCRIBER_CREATE", "INTERNAL_AGENT_CREATE", ...]
  │    5. Also calls rolesAsString(roles) for the JWT token:
  │         Result: ["MyCash Manager"]  ← ONLY role names, no privileges
  │    6. generateToken(id, fullName, ["MyCash Manager"], expiry, deviceId)
  │         → JWT scope claim = ["MyCash Manager"] (roles only)
  │    7. Returns AdminJwtResponseDto:
  │         { token: "eyJ...",
  │           id: 42,
  │           roles: ["MyCash Manager", "AGENT_CREATE", "AGENT_VIEW_ACTIVE", ...],
  │           level: 1 }
  │
  ▼ Back in webclient LoginController.verifyAdminOtp():
  │
  │  1. authenticateUser(jwtResponse):
  │       getGrantedAuthorities(jwtResponse.getRoles())
  │         → creates SimpleGrantedAuthority("MyCash Manager")
  │         → creates SimpleGrantedAuthority("AGENT_CREATE")
  │         → creates SimpleGrantedAuthority("AGENT_VIEW_ACTIVE")
  │         → ... for every string in jwtResponse.getRoles()
  │       new UsernamePasswordAuthenticationToken(name, null, authorities)
  │       SecurityContextHolder.getContext().setAuthentication(token)
  │       ← NOW the user's permissions are in the Spring Security context
  │
  │  2. SessionHelper.setToken(jwtResponse.getToken())     ← JWT stored
  │  3. SessionHelper.setAdminId(jwtResponse.getId())      ← DB id stored
  │  4. SessionHelper.setRoles(jwtResponse.getRoles())     ← full list stored
  │  5. SessionHelper.setAdminLevel(jwtResponse.getLevel()) ← level stored
  │  6. SessionHelper.setSubmitterByType(UserType.ADMIN)
  │  7. POST /user/get-customercare-info → loads UserInfoDto into session
  │
  └─► redirect to /admin/dashboard
```

#### Path B — Agent/Subscriber/Merchant Login (`/admin/get-otp-login-token`)

```
LoginController.getOtpToken()
  │  POST /get-otp-login-token to gateway
  │
  │  Gateway.SecurityController.getOtpLoginToken():
  │    1. Finds user by msisdn + OTP
  │    2. userType == AGENT → agentQueryRepository.findById(userId)
  │         agentType = agent.getAgentType()  ← DISTRIBUTOR_AGENT, RESALE_AGENT, or AGENT
  │         userRoles = agent.getRoles()
  │         userPrivileges = agent.getPrivileges()
  │    3. rolesAndPermissionsAsString(userRoles, userPrivileges)
  │         e.g. ["AGENT", "CHILD_AGENT_ACTIVE_LIST", "DEPOSIT_SUBSCRIBER_CASH_IN", ...]
  │    4. rolesAsString(userRoles) for JWT = ["AGENT"]
  │    5. Returns OtpTokenResponse:
  │         { token: "eyJ...",
  │           userId: 99,
  │           userType: AGENT,
  │           agentType: DISTRIBUTOR_AGENT,   ← KEY FIELD
  │           roles: ["AGENT", "CHILD_AGENT_ACTIVE_LIST", ...] }
  │
  ▼ Back in webclient LoginController.getOtpToken():
  │
  │  1. authenticateUser(res):
  │       getGrantedAuthorities(res.getRoles())
  │         → SimpleGrantedAuthority("AGENT")
  │         → SimpleGrantedAuthority("CHILD_AGENT_ACTIVE_LIST")
  │         → ... for every string in res.getRoles()
  │       SecurityContextHolder.getContext().setAuthentication(token)
  │
  │  2. SessionHelper.setToken(res.getToken())
  │  3. SessionHelper.setAdminId(res.getUserId())
  │  4. List<String> roles = new ArrayList<>();
  │     roles.add(res.getUserType().toString());  ← "AGENT" only!
  │     SessionHelper.setRoles(roles)             ← session stores only ["AGENT"]
  │  5. SessionHelper.setAgentType(res.getAgentType()) ← DISTRIBUTOR_AGENT stored
  │  6. SessionHelper.setSubmitterByType(res.getUserType())  ← AGENT
  │
  └─► redirect to /admin/dashboard
```

> **Important:** For agents, `SessionHelper.setRoles()` only stores `["AGENT"]`. But `SecurityContextHolder` has **all** the privilege strings. These two stores serve different purposes — read on.

---

### Step 3 — How the Gateway Validates Permissions on Every API Request

When the webclient calls the gateway for any data (e.g., `GET /user/active-agent`), the JWT is sent in the `Authorization: Bearer` header. The gateway's `ApiAuthorizationFilter` runs:

```
ApiAuthorizationFilter.doFilterInternal()
  │
  │  1. Extract "Bearer eyJ..." from Authorization header
  │  2. TokenService.getAuthentication(token):
  │       a. Parse and decrypt the JWE token (RSA-OAEP-512 key)
  │       b. Extract scope claim → ["MyCash Manager"] (just role names)
  │       c. getGrantedAuthorities(["MyCash Manager"]):
  │            - SimpleGrantedAuthority("MyCash Manager")
  │            - roleInfoRepository.findById("MyCash Manager") from Redis
  │              → CachedRoleInfo { role: "MyCash Manager",
  │                                 privileges: ["AGENT_CREATE","AGENT_VIEW_ACTIVE",...] }
  │            - SimpleGrantedAuthority("AGENT_CREATE")
  │            - SimpleGrantedAuthority("AGENT_VIEW_ACTIVE")
  │            - ... for every privilege in cache
  │       d. new UsernamePasswordAuthenticationToken(userId, null, authorities)
  │       e. SecurityContextHolder.getContext().setAuthentication(token)
  │
  │  3. Request proceeds to controller
  │
  ▼
TransferController.SubscriberCashIn():
  @PreAuthorize("@permissionService.check('DEPOSIT_SUBSCRIBER_CASH_IN')")
  │
  │  gateway PermissionService.check("DEPOSIT_SUBSCRIBER_CASH_IN"):
  │    SecurityContextHolder.getContext().getAuthentication().getAuthorities()
  │    → loops through [SimpleGrantedAuthority("MyCash Manager"),
  │                      SimpleGrantedAuthority("DEPOSIT_SUBSCRIBER_CASH_IN"), ...]
  │    → found "DEPOSIT_SUBSCRIBER_CASH_IN" → return true
  │
  └─► method executes
```

---

### Step 4 — How the Webclient Menu Uses Permissions

Every time a page is rendered (every `GET` request to the webclient), Thymeleaf re-renders `aside-left.html`. Every `sec:authorize` and `th:if` expression in the sidebar is evaluated against the **live `SecurityContextHolder`** at render time.

#### The `sec:authorize` expression

html

```html
<li sec:authorize="@permissionService.check('AGENT_CREATE', 'AGENT_VIEW_ACTIVE',
                   'AGENT_VIEW_PENDING', 'AGENT_VIEW_BLOCKED')">
```

Thymeleaf Security extras calls `permissionService.check(...)` — the **webclient's** `PermissionService` (identical logic to the gateway's):

java

```java
// mfs-webclient/security/PermissionService.java
public boolean check(String... permission) {
    Collection<? extends GrantedAuthority> roles =
        SecurityContextHolder.getContext().getAuthentication().getAuthorities();
    List<String> allowedRoles = Arrays.asList(permission);   // ["AGENT_CREATE","AGENT_VIEW_ACTIVE",...]

    for (GrantedAuthority role : roles) {
        String roleString = role.getAuthority();

        if (roleString.equals("SUPER_ADMIN")) return true;  // Super admin bypasses all checks

        if (allowedRoles.contains(roleString)) return true; // match found → show the menu item
    }
    return false;  // no match → hide the menu item
}
```

The authorities in `SecurityContextHolder` are exactly what was set during login — the full flat list of role names + privilege names — e.g.:

```
["MyCash Manager", "AGENT_CREATE", "AGENT_VIEW_ACTIVE", "AGENT_VIEW_PENDING",
 "AGENT_VIEW_BLOCKED", "SUBSCRIBER_CREATE", "INTERNAL_AGENT_CREATE", ...]
```

So `check('AGENT_CREATE', 'AGENT_VIEW_ACTIVE', ...)` loops this list and returns `true` if ANY of the passed strings appear in the authorities list. The External Agent menu item renders.

#### The `th:if` expression (for Child Agents menu item)

html

```html
<li sec:authorize="@permissionService.check('CHILD_AGENT_ACTIVE_LIST')"
    th:if="${@authService.myChildType() != null
             and (@authService.myChildType().toString() == 'RESALE_AGENT'
                  or @authService.myChildType().toString() == 'AGENT')
             and @authService.isAgent() }">
```

This has **two guards**, both must be true:

**Guard 1 — `sec:authorize`:** Does the user's authorities list contain `"CHILD_AGENT_ACTIVE_LIST"`? This privilege must be assigned to the Agent role in the database.

**Guard 2 — `th:if`:** Three conditions all chained with `and`:

java

```java
// AuthService.myChildType():
public AgentType myChildType() {
    AgentType agentType = SessionHelper.getAgentType();  // read from HTTP session
    if (agentType.equals(AgentType.DISTRIBUTOR_AGENT)) {
        return AgentType.RESALE_AGENT;
    } else if (agentType.equals(AgentType.RESALE_AGENT)) {
        return AgentType.AGENT;
    } else {
        return null;   // ← plain AGENT returns null
    }
}
```

java

```java
// AuthService.isAgent():
public boolean isAgent() {
    List<String> roles = SessionHelper.getRoles();  // reads ["AGENT"] from HTTP session
    return CollectionUtils.containsAny(roles, Collections.singletonList("AGENT"));
}
```

So the three conditions mean:

- `myChildType() != null` → agent type must be DISTRIBUTOR or RESALE (not plain AGENT)
- `myChildType().toString() == 'RESALE_AGENT' or 'AGENT'` → the child type must be RESALE_AGENT or AGENT
- `isAgent()` → the session role must be "AGENT"

---

### Why Only Distributor/Resale Agents See the Child Agent Menu

This is the exact answer to your question. Here is the complete logic:

```
┌────────────────────────────────────────────────────────────────────────┐
│  Agent logs in → SessionHelper.setAgentType(res.getAgentType())       │
│                                                                        │
│  agentType = AGENT          → myChildType() returns null               │
│                             → th:if = null != null → FALSE             │
│                             → menu item HIDDEN ✗                      │
│                                                                        │
│  agentType = RESALE_AGENT   → myChildType() returns AgentType.AGENT   │
│                             → "AGENT" == 'RESALE_AGENT' → false        │
│                             → "AGENT" == 'AGENT'        → TRUE        │
│                             → isAgent() = true                         │
│                             → th:if = TRUE                             │
│                             → menu item SHOWS ✓                       │
│                             → label: "My AGENT'S"                     │
│                             → lists their child AGENT accounts         │
│                                                                        │
│  agentType = DISTRIBUTOR_AGENT → myChildType() returns RESALE_AGENT   │
│                             → "RESALE_AGENT" == 'RESALE_AGENT' → TRUE │
│                             → isAgent() = true                         │
│                             → th:if = TRUE                             │
│                             → menu item SHOWS ✓                       │
│                             → label: "My RESALE AGENT'S"              │
│                             → lists their child RESALE_AGENT accounts  │
└────────────────────────────────────────────────────────────────────────┘
```

The menu label is also dynamic — the Thymeleaf expression:

html

```html
th:text="${'My ' + #strings.replace(#strings.toString(@authService.myChildType()), '_', ' ')} + '\'S'"
```

Produces:

- For DISTRIBUTOR_AGENT: `"My RESALE AGENT'S"`
- For RESALE_AGENT: `"My AGENT'S"`

And when the page loads, `myChildAgents()` in `AgentController` sends the logged-in agent's DB id and the child type (`AgentType.AGENT` or `AgentType.RESALE_AGENT`) to the backend's `/user/get-child-agents` endpoint, which filters the agent list by `parentAgent = loggedInAgentId AND agentType = myChildAgentType`.

---

### Complete Flow — One Diagram

```
DATABASE (SQL Server via gateway)
  Role table          e.g.  "ROLE_INTERNAL_AGENT", "MyCash Manager", "AGENT"
  Privilege table     e.g.  "AGENT_CREATE", "CHILD_AGENT_ACTIVE_LIST", ...
  roles_privileges    maps which privileges belong to which role
  users_roles         maps which roles a user has
  users_privileges    maps any direct privileges a user has
        │
        │ loaded at login
        ▼
REDIS (CachedRoleInfo)
  key="MyCash Manager" → value=["AGENT_CREATE","AGENT_VIEW_ACTIVE","SUBSCRIBER_CREATE",...]
  key="AGENT"          → value=["CHILD_AGENT_ACTIVE_LIST","DEPOSIT_SUBSCRIBER_CASH_IN",...]
        │
        │ used on every API call by TokenService.getGrantedAuthorities()
        ▼
mfs-api-gateway: LOGIN ENDPOINT
  SecurityController.verifyAdminOtp() or getOtpLoginToken()
    Loads user from DB → gets roles + privileges
    Builds flat list: ["MyCash Manager","AGENT_CREATE","AGENT_VIEW_ACTIVE",...]
    JWT token scope   = ["MyCash Manager"] (role names only)
    Response roles    = ["MyCash Manager","AGENT_CREATE","AGENT_VIEW_ACTIVE",...] (full flat list)
    Response agentType= DISTRIBUTOR_AGENT (for agents only)
        │
        │ HTTP response back to webclient
        ▼
mfs-webclient: LoginController.verifyAdminOtp() or getOtpToken()
    authenticateUser(response):
      → getGrantedAuthorities(response.getRoles())
        → SimpleGrantedAuthority("MyCash Manager")
        → SimpleGrantedAuthority("AGENT_CREATE")
        → SimpleGrantedAuthority("AGENT_VIEW_ACTIVE")
        → ... all privilege strings
      → UsernamePasswordAuthenticationToken(name, null, authorities)
      → SecurityContextHolder.setAuthentication(token)     ← AUTHORITIES IN CONTEXT

    SessionHelper.setToken(token)                          ← JWT for API calls
    SessionHelper.setRoles(["MyCash Manager","AGENT_CREATE",...]) ← full list (admin)
    SessionHelper.setRoles(["AGENT"])                             ← type only (agents)
    SessionHelper.setAgentType(DISTRIBUTOR_AGENT)                 ← agent hierarchy
        │
        │ every subsequent page render
        ▼
mfs-webclient: Thymeleaf renders aside-left.html
    sec:authorize="@permissionService.check('AGENT_CREATE','AGENT_VIEW_ACTIVE',...)"
      → PermissionService.check():
          SecurityContextHolder.getAuthentication().getAuthorities()
          → loops authorities → finds "AGENT_VIEW_ACTIVE" → return true
          → External Agent menu item RENDERED

    sec:authorize="@permissionService.check('CHILD_AGENT_ACTIVE_LIST')"
    + th:if="${@authService.myChildType() != null
               and myChildType() in ['RESALE_AGENT','AGENT']
               and isAgent()}"
      → permissionService.check(): found "CHILD_AGENT_ACTIVE_LIST" → true
      → myChildType(): SessionHelper.getAgentType() = DISTRIBUTOR_AGENT
                        → returns RESALE_AGENT (not null)
      → "RESALE_AGENT" == 'RESALE_AGENT' → true
      → isAgent(): SessionHelper.getRoles() contains "AGENT" → true
      → ALL conditions true → Child Agents menu item RENDERED

        │
        │ agent clicks "My Child Agents"
        ▼
mfs-webclient: AgentController.myChildAgents()
    @PreAuthorize("@permissionService.check('CHILD_AGENT_ACTIVE_LIST')")
    → same check → passes

    SessionHelper.getAgentType() = DISTRIBUTOR_AGENT
    myChildAgentType = RESALE_AGENT
    loggedInAgentId = SessionHelper.getAdminId() = 99

    POST /user/get-child-agents to gateway
    body: { agentType: "RESALE_AGENT", parentAgent: 99 }
    header: Authorization: Bearer <JWT>
        │
        ▼
mfs-api-gateway: UserController.getChildAgents()
    ApiAuthorizationFilter: decrypt JWT → load "AGENT" roles from Redis
      → authorities = ["AGENT", "CHILD_AGENT_ACTIVE_LIST", ...]
    @PreAuthorize("@permissionService.check('CHILD_AGENT_ACTIVE_LIST')") → pass
    Proxy to backend: POST /agent/get-child-agents
        │
        ▼
mfs-backend-new: AgentController.getChildAgent()
    userQueryRepository.findAllByParentAgentAndAgentType(99, RESALE_AGENT)
    → SQL: SELECT * FROM User WHERE parentAgent = 99 AND agentType = 'RESALE_AGENT'
    → returns only that distributor's direct resale agents
    → NOT all agents in the system
```

---

### Now You Can Modify aside-left.html

With this understanding, here is exactly how each guard works and what you need to change for different scenarios:

**To show a new menu item to ALL admins but not agents/subscribers:**

html

```html
<li sec:authorize="@permissionService.check('YOUR_PRIVILEGE_NAME')">
```

→ Add `YOUR_PRIVILEGE_NAME` to the admin role in DB via the roles management screen.

**To show a menu item only to DISTRIBUTOR agents:**

html

```html
<li sec:authorize="@permissionService.check('CHILD_AGENT_ACTIVE_LIST')"
    th:if="${@authService.getAgentType() != null
             and @authService.getAgentType().toString() == 'DISTRIBUTOR_AGENT'
             and @authService.isAgent()}">
```

**To show a menu item only to SUPER_ADMIN:**

html

```html
<li sec:authorize="hasAuthority('SUPER_ADMIN')">
```

Note: `SUPER_ADMIN` bypasses all `permissionService.check()` calls automatically — it's the `if (roleString.equals("SUPER_ADMIN")) return true;` line in `PermissionService.java`.

**To show a menu item only to internal agents (Customer Care):**

html

```html
<li th:if="${@authService.isCustomerCare()}">
```

→ `isCustomerCare()` checks `SessionHelper.getRoles()` for `"ROLE_INTERNAL_AGENT"`.

**The two stores — when to use which:**

|Use `sec:authorize` / `permissionService.check()`|Use `th:if` with `authService`|
|---|---|
|Permission-based visibility (privilege in DB)|Role-type-based visibility|
|"Does this user have AGENT_CREATE privilege?"|"Is this user an agent? a distributor? an admin?"|
|Driven by DB role setup|Driven by session type fields set at login|
|Changes without code change (just update DB role)|Requires code change to alter logic|