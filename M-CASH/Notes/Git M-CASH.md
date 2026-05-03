# Creating Bug Fix Branch

```bash
git checkout -b bugfix/subscriber-cashout-issue
```

git checkout -b bugfix/e-payroll-employee-issue

git checkout -b bugfix/third-party-issue
## Equivalent (2-step version)
```bash
git branch bugfix/subscriber-cashout-issue   # create branch
git checkout bugfix/subscriber-cashout-issue # switch to it
```

# Important concept (very important)

### Before command:

feature/implement-switch-wallet   ← you are here

---

### After command:

feature/implement-switch-wallet  
        ↓  
bugfix/subscriber-cashout-issue   ← you are now here

👉 New branch starts from your current code state