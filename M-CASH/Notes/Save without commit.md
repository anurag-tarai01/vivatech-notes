### 🟢 Option 1: Use **stash** (BEST PRACTICE)

This is exactly for your case.

```
git stash
git checkout -b feature/new-branch
git stash pop
```

✔ Keeps your changes  
✔ No commit needed  
✔ Clean history

---

### 🟢 Option 2: Create branch WITHOUT losing changes

If you are still on same branch:

```
git checkout -b feature/new-branch
```

👉 This usually works **if no conflicts**, and your changes move with you.

But if Git blocks → use stash.

---

### 🟡 Option 3: Commit (safe but noisy)

```
git add .git commit -m "temp changes"git checkout -b feature/new-branch
```

👉 Later you can:

```
git reset HEAD~1
```

---

### 🔴 Option 4: Force (NOT recommended)

```
git checkout -b feature/new-branch --force
```

❌ Risky → can lose changes