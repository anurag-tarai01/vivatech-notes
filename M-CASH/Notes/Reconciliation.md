> > **Matching records between two systems to ensure everything is correct**

### 🧠 Real-life example:

Your bank says:

Balance = ₹1000

You calculate manually:

You think = ₹1200

👉 Now you compare both → find mismatch  
👉 This process = **reconciliation**

---

### 🏦 In YOUR system:

You have **2 sides**:

#### 1. Your system (mfs-backend)

#### 2. External Switch Provider

---

### Example:

|System|Balance|
|---|---|
|Your Switch Wallet|50,000|
|Switch Provider|48,000|

👉 ❌ mismatch = problem

---

### So reconciliation means:

Check:  
Our records == Switch records ?

---

### If mismatch:

- adjust using:
    - `SWITCH_WALLET_DEPOSIT`
    - `SWITCH_WALLET_WITHDRAW`

---

### 🎯 Key idea:

> Reconciliation = “Are our numbers matching with external system?”