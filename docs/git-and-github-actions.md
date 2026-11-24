# Git and GitHub Actions

> A feature-branch workflow from clone to merge, plus tagging and a few Actions snippets.
> Placeholders: `<ORG>`, `<REPO>`, `<BRANCH>`.

## 1. Fresh clone
```bash
cd <PARENT_FOLDER>
git clone https://github.com/<ORG>/<REPO>.git <REPO>
cd <REPO>
git fetch --all            # download every branch
git branch -a              # local + remote
```

## 2. Create a branch from main
```bash
git checkout -b <BRANCH> main
git branch --show-current
```

## 3. Review, stage, commit
```bash
git status
git diff README.md
git add path/to/dir/ README.md
git diff --cached                         # review what is staged

git commit -m "docs: add health check documentation" \
           -m "Longer body that explains why, not only what."
```
Conventional prefixes: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`.

## 4. Push
```bash
git push -u origin <BRANCH>               # first push, sets upstream
git push                                  # later pushes
```

## 5. Merge locally and push (direct)
```bash
git checkout <BRANCH> && git push
git checkout main
git pull origin main
git merge <BRANCH>                        # save/close the editor if it opens
git push origin main
```
Prefer a pull request when the repo has reviewers or required checks.

## 6. Tag a release
```bash
git checkout <BRANCH>
git pull origin <BRANCH>
git tag -a v1.0.0 -m "First version"
git push origin v1.0.0
```

## 7. GitHub Actions snippets
```yaml
# Export the triggering user to later steps
- run: echo "USERNAME=${{ github.actor }}" >> $GITHUB_ENV

# Run a step only outside main
- if: ${{ github.ref != 'refs/heads/main' }}
  run: echo "feature branch"
```

## 8. Before you make a repo public
```bash
# Search the working tree AND history for secrets
git log -p --all -S 'password' | head
gitleaks detect --source . --log-opts="--all"      # https://github.com/gitleaks/gitleaks
```
Deleting a file in a later commit does **not** remove it from history. Start a fresh history or rewrite it with `git filter-repo`, and rotate any credential that was ever committed.
