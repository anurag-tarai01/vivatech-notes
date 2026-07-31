
## 1. Total Work Force Paid Last Month

Should this include:
- Only employees, or
- Both employees and vendors?

Also, should we count:
- Every successful payment record, or
- Distinct workforce members (unique employee/vendor) paid last month?

**Example:**
If the same employee receives both salary, a Bonus and a Commission in the same month, should the count be:
- 3 (payment records), or
- 1 (distinct employee)?

---

## 2. Pending Approvals

Should this represent:
### A) Number of pending uploaded payroll batches (`SalaryPaymentRequest`)

Example:
- Batch A (500 employees) = PENDING
- Batch B (200 employees) = PENDING

**Pending Approvals = 2**

### OR

### B) Number of individual pending payment records (`SalaryPayment`)

Example:
- Batch A contains 500 pending payments
- Batch B contains 200 pending payments

**Pending Approvals = 700**

---
# 3. Failed Transactions / Processed Transactions

- **Processed Transactions** = count of individual successful payments (`SalaryPayment`)
- **Failed Transactions** = count of individual failed payments (`SalaryPayment`)
- **Pending Approvals** = count of pending payroll batches (`SalaryPaymentRequest`)

| Dashboard Metric           | Entity               |
| -------------------------- | -------------------- |
| Pending Approvals          | SalaryPaymentRequest |
| Failed Transactions        | SalaryPayment        |
| Processed Transactions     | SalaryPayment        |
| Total Amount Processed     | SalaryPayment        |
| Work Force Paid Last Month | SalaryPayment        |
