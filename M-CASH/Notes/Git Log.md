### 🔹 1. Basic commit history

```
git log
```

Shows full details: commit hash, author, date, message.

---

### 🔹 2. Short & clean view (very common)

```
git log --oneline
```

Example:

```
a1b2c3d Fix payment buge4f5g6h Add login API
```

---

### 🔹 3. See commits with branch graph

```
git log --oneline --graph --all --decorate
```

Great for understanding branching and merges.

---

### 🔹 4. Filter commits by author

```
git log --author="Your Name"
```

---

### 🔹 5. See commits affecting a file

```
git log filename.java
```

---

### 🔹 6. See changes in each commit

```
git log -p
```

---

### 🔹 7. Limit number of commits

```
git log -n 5
```

---

### 🔹 8. Check a specific commit details

```
git show <commit-id>
```

---

### 🔹 9. Very useful (blame – who changed what)

```
git blame filename.java
```

---

### 💡 Pro tip (you’ll use this a lot)

```
git log --oneline --graph --decorate
```

👉 This gives a clean, visual history — best for daily use.