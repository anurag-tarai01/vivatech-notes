### 1. Check global Git user (applies to all repos)

```bash
git config --global user.name
git config --global user.email
```

### 2. Check local Git user (for current repository only)

Navigate to your project folder and run:

```bash
git config user.name
git config user.email
```

### 3. See all Git config (global + local)

```
git config --list
```

### 4. Check where values are coming from

```
git config --list --show-origin
```

---

### Quick understanding

- **Global config** → used for all projects unless overridden
- **Local config** → specific to a repo (overrides global)