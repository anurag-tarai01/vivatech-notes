# 1. Go back one commit, keep all changes
git reset HEAD^

# 2. Restore the unwanted file to its previous version
git restore --source=HEAD -- path/to/unwanted-file

# 3. Stage ONLY the files you actually want
git add file1
git add file2
git add file3

# 4. Create the commit again
git commit -m "Your commit message"

# 5. Verify before pushing
git show --stat --oneline HEAD