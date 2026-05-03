# ✅ 1. Internal Agent (a.k.a Customer Care / Admin-side)

### 👉 Who they are

- They are **NOT stored as “Agent”**
- They are actually **Admins**

From your code:

- `AdminType.CUSTOMER_CARE`
- Role: `ROLE_INTERNAL_AGENT`

---
### 👉 Where they live

- Stored in: **Admin entity**
- Created using:
    - `CreateAdminCommand`

---

### 👉 What they do

Internal agents are **system operators**:

- Create Agents ✅
- Create Subscribers ✅
- Create Merchants ✅
- Approve / Reject all users ✅
- Block / Unblock users ✅

👉 Basically: **they control the system**

---

### 👉 Example from your flow

POST /admin/register-customer-care  
 → CreateAdminCommand  
 → AdminCreatedEvent

---

# ✅ 2. External Agent (Business-side agent)

### 👉 Who they are

- These are **real field agents**
- Stored as **User**

From your code:

- `UserType.AGENT`
- Also:
    - `RESALE_AGENT`
    - `DISTRIBUTOR_AGENT`

---

### 👉 Where they live

- Stored in: **User entity**
- Created using:
    - `CreateAgentCommand`

---

### 👉 What they do

External agents are **business users**:

- Register Subscribers ✅
- Sometimes create Merchants ✅
- Perform transactions (cash-in, etc.) ✅

👉 They are **part of the business network**, not system admins

---

### 👉 Example flow

POST /user/register-agent  
 → CreateAgentCommand  
 → AgentCreatedEvent

---

# 🔥 3. KEY DIFFERENCE (this is the main idea)

|Feature|Internal Agent|External Agent|
|---|---|---|
|Stored in|Admin|User|
|Enum|`AdminType.CUSTOMER_CARE`|`UserType.AGENT`|
|Role|`ROLE_INTERNAL_AGENT`|`ROLE_AGENT_USER`|
|Purpose|System control|Business operations|
|Creates users|Yes|Limited|
|Approves users|Yes|No|
|Part of CQRS admin flow|Yes|Yes (but as user)|

---

# 🧠 4. Simple way to remember

👉 **Internal Agent = System Admin (back-office)**  
👉 **External Agent = Field Agent (business user)**