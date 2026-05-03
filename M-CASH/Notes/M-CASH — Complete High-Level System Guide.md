
#### For Fresher Developers | Reusable as AI Context

---

### 1. SYSTEM OVERVIEW

#### What This System Does

M-CASH is a **mobile money platform** — a digital financial system that lets people store, send, and receive money using just a phone number, without needing a traditional bank account.

Think of it exactly like **M-Pesa in Kenya** or **bKash in Bangladesh**. The system runs in **Cameroon**, uses **XAF (Central African Franc)** as the currency, and is operated by a company called Vivacom under the brand name **MyCash**.

In the simplest terms: M-CASH turns physical cash into digital money, moves it around instantly, and turns it back into physical cash — all through a network of local agents.

#### What Problem It Solves

Most people in regions like Cameroon don't have bank accounts. They carry physical cash. This creates real problems:

- Sending money to someone far away is slow and risky
- Paying bills or merchants requires physical presence
- Saving money safely is hard without a bank

M-CASH solves this by creating a **digital wallet tied to a phone number**. Anyone can own one. Local shopkeepers (called agents) act as the entry and exit points for physical cash. Everything in between is instant and digital.

#### Who the Main Users Are

```
┌─────────────────────────────────────────────────────────────────┐
│  USER TYPE        │  WHO THEY ARE                               │
├─────────────────────────────────────────────────────────────────┤
│  Subscriber       │  Everyday person with a digital wallet      │
│  Agent            │  Local shopkeeper who handles cash          │
│  Resale Agent     │  Agent who manages other agents below them  │
│  Distributor      │  Top-level agent managing resale agents     │
│  Merchant         │  Business that accepts digital payments     │
│  Internal Agent   │  Back-office staff (Customer Care)          │
│  Network Admin    │  Managers with back-office privileges       │
│  Super Admin      │  Full system control                        │
└─────────────────────────────────────────────────────────────────┘
```

---

### 2. FINANCE DOMAIN EXPLANATION

#### What is a Wallet

A wallet is just **a number in a database** that says how much digital money someone has.

When you "put money in your wallet," that number goes up. When you "send money," that number goes down and the recipient's number goes up.

No physical money moves between wallets — only database numbers change. This is exactly how a bank account balance works, except here it's tied to a phone number, not a bank.

Every user has at least one wallet. Agents typically have two: a **main wallet** (for transactions) and a **commission wallet** (where they collect earnings from fees).

#### Subscriber vs Agent — The Key Relationship

```
REAL WORLD ANALOGY:

  You (Subscriber)                     ATM / Bank Branch (Agent)
  ─────────────────                    ──────────────────────────
  Want to put cash                     Accepts your physical cash
  into your digital wallet    ──────►  Credits your digital wallet
                                       Their wallet goes DOWN by same amount

  Want to withdraw cash               Hands you physical cash
  from your digital wallet   ──────►  Debits your digital wallet
                                       Their wallet goes UP by same amount
```

The **subscriber** is the end customer. The **agent** is the human ATM. The agent holds physical cash in their shop, and holds digital money in their wallet. When they do cash-in for a subscriber, they give away digital money and receive physical cash. When they do cash-out, the opposite happens.

This is why **agent float management** (making sure agents have enough digital balance) is critical to the business.

#### Cash-In, Cash-Out, Transfer — The Three Core Concepts

**Cash-In** — Converting physical cash into digital money.

```
Subscriber gives physical cash to Agent
→ Agent's digital wallet goes DOWN
→ Subscriber's digital wallet goes UP
→ Net result: cash is now digital
```

**Cash-Out** — Converting digital money back into physical cash.

```
Subscriber requests withdrawal from Agent
→ Subscriber's digital wallet goes DOWN
→ Agent's digital wallet goes UP
→ Agent gives physical cash to Subscriber
→ Net result: digital money is now cash
```

**Transfer (P2P)** — Moving digital money between two wallets.

```
Subscriber A sends to Subscriber B
→ A's wallet goes DOWN
→ B's wallet goes UP
→ No physical cash involved at any point
→ Instant, purely digital
```

#### What is the Switch Wallet

The **switch wallet** is a special system-level wallet that acts as a **liquidity bridge** between M-CASH and external mobile money networks (like MTN MoMo or Orange Money).

```
PROBLEM:  M-CASH subscriber wants to send money to an MTN MoMo user
SOLUTION: The Switch Wallet acts as the middleman

  M-CASH Subscriber                          MTN MoMo User
  ─────────────────                          ─────────────
  Their wallet goes DOWN                     Receives money via external network
         │
         ▼
  Switch Wallet goes UP         ──external API──►  External Network delivers cash
  (internal accounting)
```

The switch wallet balance at any point = total funds currently held as float with the external provider. Admins manually top up (deposit) and withdraw from this float when reconciling with the external provider.

There are 4 switch operations:

- **Deposit** — Admin puts money INTO the switch float (2-step approval)
- **Withdraw** — Admin pulls money OUT of the switch float (2-step approval)
- **Send Switch Money** — Subscriber sends money OUT to another network (webhook-confirmed)
- **Receive Switch Money** — Subscriber collects money IN from another network (webhook-confirmed)

#### Digital Money vs Physical Cash

```
PHYSICAL CASH                        DIGITAL MONEY
─────────────────                    ──────────────────────────────
Lives in your pocket                 Lives as a number in a database
Transferred by handing over          Transferred by updating 2 numbers
Can be lost or stolen                Protected by PIN + system security
Takes time to move                   Instant
Counted manually                     Tracked automatically
No transaction history               Full audit trail
```

#### Real-World Analogy

M-CASH = M-Pesa (Kenya) = Paytm (India) = bKash (Bangladesh) = Wave (West Africa)

All of these solve the same problem: bringing financial services to people without bank accounts by using a mobile number as the account identifier and local agents as the physical cash entry/exit points.

---

### 3. MICROSERVICES OVERVIEW

The system is split into 5 separate applications, each with a single responsibility. They talk to each other but are independently deployable.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  SERVICE           │ PORT  │ PURPOSE                                     │
├─────────────────────────────────────────────────────────────────────────┤
│  mfs-webclient     │ –     │ Browser-based admin/agent portal (UI)       │
│  mfs-api-gateway   │ 3088  │ Security gate + request router              │
│  mfs-backend-new   │ 8090  │ All business logic and money movement       │
│  mfs-notification  │ 8989  │ SMS and push notifications                  │
│  mfs-reporting     │ 9090  │ Transaction reports and audit records        │
└─────────────────────────────────────────────────────────────────────────┘
```

#### mfs-api-gateway (Port 3088)

**Why it exists:** Every request from the outside world must pass through one controlled entry point. The gateway is that entry point — it checks who you are and whether you're allowed to do what you're trying to do.

**What it does:**

- Issues login tokens (JWT) after verifying username/password + OTP
- Validates that token on every subsequent request
- Checks if the logged-in user has permission for the action they're requesting
- Forwards approved requests to the backend
- Does zero business logic of its own

**How it interacts:** Receives requests from webclient (or mobile apps). Forwards them to mfs-backend-new. Also calls mfs-notification when needed (e.g., after cash-out success).

#### mfs-backend-new (Port 8090)

**Why it exists:** This is the brain of the system. All business rules, money movement, user management, and transaction processing live here.

**What it does:**

- All user lifecycle: create, approve, block, update subscribers/agents/admins
- All financial transactions: cash-in, cash-out, transfers, switch wallet flows
- Wallet balance updates (where actual money movement happens)
- Commission calculation and distribution
- Validation rules (is the user active? is their balance sufficient? are limits exceeded?)
- Talking to external APIs (like the Switch wallet provider)

**How it interacts:** Receives proxied requests from the gateway. Publishes events to RabbitMQ for reporting and notification. Calls external APIs (Switch, Amal Express, etc.) for remittances.

#### mfs-notification (Port 8989)

**Why it exists:** After a transaction completes, the people involved need to be informed. This service handles all communication.

**What it does:**

- Sends SMS to subscribers and agents after transactions
- Sends push notifications to mobile apps
- Handles OTP delivery for login
- Sends email notifications when relevant

**How it interacts:** Two ways — the gateway calls it directly via REST for some events (like requesting OTP). The backend sends it events via RabbitMQ for transaction notifications. It is purely outbound — it never triggers transactions.

#### mfs-reporting (Port 9090)

**Why it exists:** Every transaction needs to be recorded for audits, business reports, regulatory compliance, and customer statements. The main backend focuses on speed of transaction processing — the reporting service focuses on thorough record-keeping.

**What it does:**

- Maintains a complete record of every transaction (CustomerTransactionReport)
- Tracks wallet balance history over time
- Provides data for dashboards, reports, and statements
- Records audit trails for compliance

**How it interacts:** Listens to RabbitMQ events published by the backend. It is purely a consumer — it never initiates or modifies transactions. If it goes down, transactions still succeed; they'll be recorded when it comes back up.

#### mfs-webclient

**Why it exists:** Admins, agents, and back-office staff need a browser-based interface to do their work. Mobile apps handle the subscriber side; the webclient handles the admin/agent side.

**What it does:**

- Login portal for admins and agents
- Register new subscribers, agents, merchants
- Approve or reject pending registrations
- Initiate deposits, transfers, cash-in/cash-out on behalf of customers
- View reports and transaction history
- Manage roles, zones, grades, commissions

**How it interacts:** Calls the api-gateway for everything. The webclient is essentially a browser frontend — it never talks to the backend directly. All requests go through the gateway.

---

### 4. CORE FLOWS (HIGH LEVEL)

#### A. Subscriber Cash-Out

A subscriber walks into an agent's shop and says "I want 5,000 XAF cash."

```
Subscriber (via agent's webclient or agent's app)
  │
  ▼
[mfs-webclient] — Agent fills in: subscriber phone, amount, their own PIN
  │
  ▼
[mfs-api-gateway] — Validates agent's token. Checks agent has CASH_OUT permission
  │
  ▼
[mfs-backend-new] — The critical work happens here:
  │  1. Checks subscriber wallet status (must be ACTIVE)
  │  2. Checks subscriber has enough balance
  │  3. Checks amount is within allowed limits
  │  4. DEBIT subscriber wallet (balance goes down)
  │  5. CREDIT agent wallet (balance goes up)
  │  6. Records the transaction
  │  7. Calculates and credits agent commission
  │
  ▼
[mfs-notification] — (Async) Sends SMS to subscriber: "You withdrew 5000 XAF"
[mfs-reporting]    — (Async) Saves full transaction record for reports

Result: Subscriber receives physical cash from agent.
        Agent's digital wallet balance increased.
        Subscriber's digital wallet balance decreased.
```

#### B. Deposit / Transfer (AMT01 → Agent Float)

When an agent runs low on digital balance, the company deposits digital money into the agent's wallet so they can continue doing cash-ins. This is called "float management."

```
Admin (via webclient)
  │
  ▼
[mfs-webclient] — Admin selects agent, enters amount
  │
  ▼
[mfs-api-gateway] — Validates admin token, checks deposit permission
  │
  ▼
[mfs-backend-new]
  │  1. This is a 2-step flow: INITIATE then APPROVE
  │  2. Step 1 (Initiate): Creates a pending transaction record only
  │  3. A second admin reviews and approves
  │  4. Step 2 (Approve): DEBIT company float wallet (AMT01)
  │                        CREDIT agent's main wallet
  │  5. Records transaction
  │
  ▼
[mfs-reporting] — (Async) Records the deposit

Result: Agent now has more digital balance to give out during cash-ins.
```

#### C. Switch Wallet Flow

A subscriber wants to send money to someone on MTN MoMo (a different network).

```
SEND MONEY OUT (Subscriber → Other Network):

Subscriber (via mobile app or webclient)
  │
  ▼
[mfs-api-gateway] — Validates token, checks permission
  │
  ▼
[mfs-backend-new] — Phase 1: Initiate
  │  1. Validates subscriber balance
  │  2. Saves a "STARTED" transaction record (no money moves yet)
  │  3. Calls external Switch API → asks external network to pay the beneficiary
  │  4. Returns "Transaction Initiated" to user immediately
  │
  ▼ (External Switch API processes the payment asynchronously)
  │
[External Switch API] — sends a webhook (callback) back to backend
  │
  ▼
[mfs-backend-new] — Phase 2: Webhook Received
  │  If SUCCESS:
  │    DEBIT subscriber wallet (amount + fee)
  │    CREDIT Switch Wallet (internal accounting)
  │    Record transaction as DONE
  │  If FAILED:
  │    No wallet changes (or reverse if already done)
  │    Record transaction as FAILED
  │
  ▼
[mfs-reporting] — (Async) Records the outcome

Result: Beneficiary on MTN MoMo receives cash.
        Subscriber's wallet reduced by amount + fee.
        Switch wallet increased (tracks float held with external provider).
```

---

### 5. MONEY FLOW (CRITICAL)

#### The Golden Rule

**All real money movement — every debit and credit — happens exclusively inside mfs-backend-new.**

No other service touches wallet balances. Not the gateway. Not the webclient. Not notification. Not reporting.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     MONEY MOVEMENT MAP                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  mfs-webclient      → NEVER touches money. Only UI.                 │
│  mfs-api-gateway    → NEVER touches money. Only auth + routing.     │
│  mfs-backend-new    → THE ONLY SERVICE THAT MOVES MONEY.            │
│  mfs-notification   → NEVER touches money. Only sends messages.     │
│  mfs-reporting      → NEVER touches money. Only reads and records.  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Where Balance Updates Happen

Inside mfs-backend-new, the balance update is a direct SQL UPDATE on the Wallet table:

- Debit: `balance = balance - amount` on the sender's wallet row
- Credit: `balance = balance + amount` on the receiver's wallet row

These two updates happen within the same transaction flow. The system also uses a concurrency guard — if two transactions try to debit the same wallet simultaneously, only one succeeds. The other is rejected and must retry.

#### What is Critical vs What Can Fail Safely

```
MUST NEVER FAIL:                     CAN FAIL SAFELY:
─────────────────                    ─────────────────────────────
Wallet debit                         SMS notification
Wallet credit                        Push notification
Transaction record creation          Reporting record creation
Commission calculation               Email alerts
External API call (switch)           Audit log entries
Webhook handling                     Dashboard data updates

If debit fails  → Transaction aborted, nothing changes
If credit fails → System must reverse the debit (critical!)
If SMS fails    → Transaction still succeeded, user just didn't get notified
If reporting fails → Transaction still succeeded, report catches up later
```

---

### 6. SYSTEM DESIGN STYLE

#### How Services Communicate

The system uses **two communication patterns**, chosen based on whether speed or reliability matters more:

```
SYNCHRONOUS (REST / HTTP)           ASYNCHRONOUS (RabbitMQ / Message Queue)
───────────────────────             ───────────────────────────────────────
Used when the caller needs          Used when the caller doesn't need to
an immediate answer                 wait for the result

Examples:                           Examples:
- Webclient → Gateway               - Backend → Reporting (after transaction)
- Gateway → Backend                 - Backend → Notification (after transaction)
- Backend → External Switch API     - Backend → Audit log updates

Characteristics:                    Characteristics:
- Caller waits for response         - Caller fires and forgets
- If receiver is down, request      - If receiver is down, message waits
  fails immediately                   in queue until receiver recovers
- Used for transactions             - Used for notifications and reports
```

#### Role of the API Gateway

The gateway is the **single entry point** for all external traffic. Think of it as a security checkpoint at the entrance to a building.

```
WHAT THE GATEWAY DOES ON EVERY REQUEST:

Incoming Request
      │
      ▼
  ┌───────────────────────────────────────────┐
  │ 1. Is there a valid token? (Authentication)│
  │    NO → Reject immediately (401)           │
  │    YES → Continue                          │
  ├───────────────────────────────────────────┤
  │ 2. Does this token have the right          │
  │    permission for this action?             │
  │    (Authorization)                         │
  │    NO → Reject (403)                       │
  │    YES → Continue                          │
  ├───────────────────────────────────────────┤
  │ 3. Forward request to backend             │
  │    Add: user ID, trace ID, client IP       │
  └───────────────────────────────────────────┘
```

The gateway never does business logic. It only decides "who are you and are you allowed to do this?"

#### How a Request Travels End-to-End

```
Browser/App submits form
        │
        ▼
  [mfs-webclient]
  Builds a structured request, attaches JWT token
        │
        ▼ HTTP REST
  [mfs-api-gateway]
  Decrypts token, checks permission, forwards request
  Adds metadata headers (user ID, trace ID)
        │
        ▼ HTTP REST
  [mfs-backend-new]
  Validates business rules, updates wallet balances,
  saves transaction records, publishes events to RabbitMQ
        │
        ├──── RabbitMQ ────► [mfs-notification]
        │                    Sends SMS/push (async)
        │
        └──── RabbitMQ ────► [mfs-reporting]
                             Saves transaction reports (async)
        │
        ▼ HTTP Response
  [mfs-api-gateway] passes response back
        │
        ▼
  [mfs-webclient] shows success or error to user
```

---

### 7. SIMPLIFIED ARCHITECTURE DIAGRAM

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        M-CASH SYSTEM ARCHITECTURE                           │
└─────────────────────────────────────────────────────────────────────────────┘

  USERS                    ENTRY LAYER              CORE LAYER
  ─────                    ───────────              ──────────

  [Admin Browser]  ──────► [mfs-webclient]
  [Agent Browser]  ──────►  Spring MVC UI
                            Thymeleaf                    │
                            Port: internal               │ HTTP REST
                                                         ▼
  [Mobile App]  ─────────────────────────► [mfs-api-gateway]
  [Postman/API] ─────────────────────────►  JWT Auth + Permission Check
                                            Port: 3088
                                                         │
                                                         │ HTTP REST
                                                         ▼
                                           [mfs-backend-new]
                                            All Business Logic
                                            Wallet Balance Updates
                                            Transaction Processing
                                            Port: 8090
                                                  │
                      ┌──────────────────────────┤
                      │                          │
                      │ RabbitMQ                 │ RabbitMQ
                      │ (Async events)           │ (Async events)
                      ▼                          ▼
            [mfs-notification]         [mfs-reporting]
             SMS / Push / OTP          Transaction Reports
             Port: 8989                Audit Records
                                       Port: 9090

  EXTERNAL INTEGRATIONS
  ─────────────────────
  [Switch Wallet API]  ←──► mfs-backend-new  (cross-network payments)
  [Amal Express API]   ←──► mfs-backend-new  (international remittance)
  [Amal Bank API]      ←──► mfs-backend-new  (bank transfers)
  [SMS Gateway]        ←──► mfs-notification

  DATA LAYER
  ──────────
  [SQL Server DB]      ←──► mfs-backend-new  (primary: wallets, users, transactions)
  [SQL Server DB]      ←──► mfs-reporting    (reporting: reports, audit logs)
  [Redis Cache]        ←──► mfs-api-gateway  (roles and permission cache)
  [RabbitMQ]                                  (message broker between services)
```

---

### 8. RISK & FAILURE UNDERSTANDING

#### Where Money Inconsistency Can Happen

The most dangerous moments in any money system are when **two operations should happen together but only one completes**. In M-CASH:

```
RISK SCENARIO 1 — PARTIAL TRANSACTION:
  Subscriber wallet debited ✓
  Agent wallet credited     ✗  (system crashed here)
  Result: Money disappeared. Neither person has it.
  Prevention: The system checks both operations succeeded.
              If credit fails, the debit is reversed.

RISK SCENARIO 2 — DOUBLE PROCESSING:
  Transaction completed ✓
  Webhook arrives again from external Switch provider
  System processes it a second time → subscriber debited twice
  Prevention: The system checks if a transaction ID was already processed
              before doing anything. This is called idempotency.

RISK SCENARIO 3 — CONCURRENT DEBIT:
  Two cash-outs happen simultaneously from the same wallet
  Both see balance = 1000 XAF
  Both try to deduct 800 XAF
  Both succeed → wallet goes to -600 XAF (impossible negative balance)
  Prevention: The debit SQL query includes a version check.
              Only one update succeeds. The other is rejected.

RISK SCENARIO 4 — SWITCH WEBHOOK NEVER ARRIVES:
  Subscriber sent switch money, external API accepted
  Webhook never comes back (external system down)
  Transaction is stuck in STARTED state permanently
  Prevention: Admin can manually cancel stuck transactions.
              The subscriber's wallet was NOT debited yet, so no money is lost.
```

#### What Must NEVER Fail

```
┌─────────────────────────────────────────────────────────────┐
│  THESE OPERATIONS ARE THE SYSTEM'S HEARTBEAT.               │
│  If they fail, money is at risk. They must succeed or       │
│  rollback completely.                                        │
├─────────────────────────────────────────────────────────────┤
│  ✗ NEVER partial:  Debit without Credit                     │
│  ✗ NEVER partial:  Credit without Debit                     │
│  ✗ NEVER missing:  Transaction record after money moves     │
│  ✗ NEVER missing:  Webhook processing (Switch flows)        │
│  ✗ NEVER duplicate: Same transaction processed twice        │
└─────────────────────────────────────────────────────────────┘
```

#### What Can Fail Safely

```
✓ SAFE TO FAIL (system recovers gracefully):

  SMS notification     → Transaction already succeeded.
                          User just doesn't get the SMS.
                          Support team can send manually.

  Push notification    → Same as above.

  Reporting record     → Transaction succeeded.
                          Report will be created when service recovers.
                          RabbitMQ holds the message until then.

  Dashboard data       → Shows stale data temporarily.
                          Refreshes when reporting service is back.

  Audit log            → Non-critical for real-time operation.
                          Can be rebuilt from transaction records.

  PDF/Statement email  → User can request it again later.
```

---

### 9. HOW TO APPROACH THIS SYSTEM (FOR A FRESHER)

#### The Right Order to Learn This System

Learning a large system without a plan leads to confusion. Follow this sequence:

```
WEEK 1 — Understand the domain first (not the code)
────────────────────────────────────────────────────
Goal: Know WHY the system exists before HOW it works.

  □ Understand what M-Pesa / Paytm does in real life
  □ Understand: wallet, subscriber, agent, cash-in, cash-out
  □ Understand: why agents exist (they are human ATMs)
  □ Understand: why digital money needs to be "backed" by physical cash
  □ Read this document again after understanding the above

WEEK 2 — Understand the architecture (not the code)
─────────────────────────────────────────────────────
Goal: Know what each service does and how requests flow.

  □ Trace one complete flow on paper: subscriber cash-out
  □ Understand: gateway = security, backend = brain, notification = SMS
  □ Understand: REST (sync) vs RabbitMQ (async) and when each is used
  □ Understand: why 5 services instead of 1 (separation of concerns)

WEEK 3 — Look at the backend first (mfs-backend-new)
──────────────────────────────────────────────────────
Goal: Understand where money actually moves.

  □ Find where wallet balances are updated (one place — the wallet table)
  □ Understand: validate → calculate fee → debit → credit → record
  □ Understand: what happens if validation fails (nothing changes)
  □ Understand: what happens if credit fails (debit is reversed)
  □ Look at just ONE transaction type end-to-end (e.g. cash-in)

WEEK 4 — Explore the gateway and webclient
────────────────────────────────────────────
Goal: Understand how requests come in and are authorized.

  □ Understand: what a JWT token is and what it contains
  □ Understand: how permission check works (one privilege per endpoint)
  □ Understand: how webclient sends requests with the token attached
  □ Trace: login → get token → make authenticated request

WEEK 5 — Look at supporting services
──────────────────────────────────────
Goal: Understand notification and reporting as observers.

  □ Understand: RabbitMQ as a postal service (backend posts letter, receiver reads later)
  □ Understand: reporting only reads events, never writes to transactions
  □ Understand: notification only sends messages, never changes data
```

#### What to Focus On First

```
HIGH PRIORITY (understand these first):
  ✓ Finance domain concepts (wallet, agent, cash-in, cash-out)
  ✓ The cash-out flow end-to-end (most common transaction)
  ✓ How money debit/credit works in mfs-backend-new
  ✓ Role of the gateway (auth only, no business logic)

MEDIUM PRIORITY (understand after the above):
  ✓ User registration and approval flow
  ✓ Switch wallet concept and two-phase pattern
  ✓ How RabbitMQ connects services asynchronously
  ✓ Commission calculation and separate commission wallets
```

#### What to Ignore Initially

```
LOW PRIORITY (come back to these later):
  ✗ Amal Express and Amal Bank flows (international remittance — complex)
  ✗ CQRS and Event Sourcing concepts in the user module (advanced)
  ✗ Axon Framework details (complex architecture pattern)
  ✗ Akka actors in wallet-query module (concurrency pattern — advanced)
  ✗ Reporting service internals (important but not urgent)
  ✗ Grade/zone/area hierarchy management (configuration, not core logic)
  ✗ Biller payments and USSD flows (separate features)
```

#### Mental Model to Keep in Mind

Every time you look at any part of this system, ask yourself these three questions:

```
  ┌────────────────────────────────────────────────────┐
  │  1. Is money moving in this flow?                  │
  │     → If yes: focus on mfs-backend-new            │
  │     → If no: it's probably auth or notification   │
  │                                                    │
  │  2. Who is the actor?                              │
  │     → Subscriber: end customer                    │
  │     → Agent: human ATM                            │
  │     → Admin: back-office manager                  │
  │                                                    │
  │  3. Is this sync or async?                         │
  │     → Sync (REST): must succeed now               │
  │     → Async (RabbitMQ): can succeed later         │
  └────────────────────────────────────────────────────┘
```

---

### QUICK REFERENCE CARD

_Use this as a cheat sheet when you are confused_

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CONCEPT             │ SIMPLE EXPLANATION                               │
├─────────────────────────────────────────────────────────────────────────┤
│  Wallet              │ A number in DB = your digital balance            │
│  Cash-In             │ Agent gives digital, gets physical cash          │
│  Cash-Out            │ Subscriber gets physical, loses digital          │
│  Transfer            │ Number goes down for A, up for B                 │
│  Switch Wallet       │ System pool for cross-network payments           │
│  Commission Wallet   │ Separate account for earned fees                 │
│  AMT01 Wallet        │ Company's main float / treasury wallet           │
│  Gateway             │ Security door — auth only, no business logic     │
│  RabbitMQ            │ Postal service — backend sends, others receive   │
│  PENDING status      │ Created but not yet approved                     │
│  ACTIVE status       │ Approved, fully usable                           │
│  Webhook             │ External system calling us back asynchronously   │
│  Float               │ Digital balance an agent holds to do cash-ins    │
│  Service Charge      │ Transaction fee deducted from sender             │
│  Commission          │ Fee credited to agent for facilitating a txn     │
├─────────────────────────────────────────────────────────────────────────┤
│  WHICH SERVICE DOES WHAT                                                │
├─────────────────────────────────────────────────────────────────────────┤
│  Money moves         │ mfs-backend-new ONLY                             │
│  Auth/Security       │ mfs-api-gateway ONLY                             │
│  User interface      │ mfs-webclient ONLY                               │
│  SMS/Notifications   │ mfs-notification ONLY                            │
│  Reports/Audit       │ mfs-reporting ONLY                               │
└─────────────────────────────────────────────────────────────────────────┘
```