## Command

```bash
git checkout <commit-id> -- <file-path>
```

## Example

```bash
git checkout 2ea933f343cf50f6496b30a3b1ee930a79aa412f -- src/main/resources/templates/admin/deposit/internal-agent/create.html
```

This restores only that file to the exact version from the specified commit.

## After Restore

Check changes:

```bash
git diff
```

Commit the restored file:

```bash
git add src/main/resources/templates/admin/deposit/internal-agent/create.html
git commit -m "Restore create.html from old commit"
```

## Notes

- Does NOT change branch history
    
- Does NOT revert entire commit
    
- Only affects the selected file
    
- Works even if current branch is different