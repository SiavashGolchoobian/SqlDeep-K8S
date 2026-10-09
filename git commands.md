Commit changes locally and push it to origin/remote repository
```shell
git add -A
git commit -m "Update notes"
git pull --rebase
git push
```

What each does:
- `git add -A` stages new, changed, renamed, and deleted files.
- `git commit` saves a version locally.
- `git pull --rebase` gets changes made from another device before you push.
- `git push` uploads your local commits to git.