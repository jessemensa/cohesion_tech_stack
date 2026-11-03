# GitHub Workflows

## ADDING PROJECT TO GITHUB

1. Create an empty repo on GitHub (github.com → **New repository**). Don't add a README, .gitignore or license if the project already has files.
2. In a terminal, go to the project folder:
   ```bash
   cd /path/to/project
   ```
3. Turn the folder into a git repo:
   ```bash
   git init
   git branch -M main
   ```
4. (Recommended) Create a `.gitignore` so secrets and build files aren't uploaded (e.g. `node_modules/`, `.env`, `.DS_Store`).
5. Stage and commit everything:
   ```bash
   git add .
   git commit -m "Initial commit"
   ```
6. Link the folder to the GitHub repo (copy the URL from the repo page):
   ```bash
   git remote add origin https://github.com/<username>/<repo>.git
   ```
7. Push:
   ```bash
   git push -u origin main
   ```

Shortcut with the GitHub CLI: `gh repo create <repo> --private --source=. --push`


## CLONING PROJECT FROM GITHUB

1. On the repo page, click **Code** and copy the HTTPS (or SSH) URL.
2. Go to the folder you want the project in:
   ```bash
   cd ~/Desktop
   ```
3. Clone it:
   ```bash
   git clone https://github.com/<username>/<repo>.git
   ```
4. Enter the project:
   ```bash
   cd <repo>
   ```
5. Install dependencies if the project needs them (e.g. `npm install`, `pip install -r requirements.txt`) and recreate any `.env` files, since those aren't stored on GitHub.

Shortcut with the GitHub CLI: `gh repo clone <username>/<repo>`


## UPDATING GITHUB PROJECT WITH CHANGES

1. Get the latest version first (avoids conflicts):
   ```bash
   git pull
   ```
2. Make your changes, then see what changed:
   ```bash
   git status
   git diff
   ```
3. Stage the changes:
   ```bash
   git add .                # everything
   git add path/to/file     # or specific files
   ```
4. Commit with a short description:
   ```bash
   git commit -m "Describe what you changed"
   ```
5. Push to GitHub:
   ```bash
   git push
   ```

Working on a feature branch instead of main (recommended for teams):
```bash
git checkout -b my-feature
# ...make changes, add, commit...
git push -u origin my-feature
```
Then open a Pull Request on GitHub (or `gh pr create`) and merge it into `main`.


## REVERTING CHANGES BACK IN PROJECT

Pick the case that matches where the changes are:

**A. Changes not staged yet (haven't run `git add`)**
```bash
git restore path/to/file    # one file
git restore .               # all files
```

**B. Changes staged but not committed**
```bash
git restore --staged path/to/file   # unstage, keeps your edits
git restore path/to/file            # then discard the edits if wanted
```

**C. Committed but NOT pushed**
```bash
git reset --soft HEAD~1    # undo last commit, keep changes staged
git reset --hard HEAD~1    # undo last commit AND delete the changes (permanent)
```

**D. Committed AND pushed (safe for shared repos)**
```bash
git log --oneline              # find the commit hash
git revert <commit-hash>       # creates a new commit that undoes it
git push
```

**E. Go back to an older version entirely**
```bash
git log --oneline                  # find the commit you want
git checkout <commit-hash>         # look around (read-only "detached" state)
git checkout main                  # come back

git reset --hard <commit-hash>     # actually move main back to that commit
git push --force                   # only if already pushed and nobody else uses the branch
```

Tip: avoid `reset --hard` and `push --force` on shared branches. Use `git revert` there instead.
