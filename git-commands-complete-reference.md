# Git — The Complete Technical Reference
### Every command, every recovery trick, every "wait, how do I undo that?" — in tables.

---

## 1. Initial Setup & `git config`

`git config` has **three scopes**, checked in this priority order: `--local` (this repo only) → `--global` (this user, all repos) → `--system` (every user on the machine).

| Command | What it does |
|---|---|
| `git config --global user.name "Your Name"` | Sets your commit author name globally |
| `git config --global user.email "you@example.com"` | Sets your commit author email globally |
| `git config --global core.editor "vim"` | Sets default editor (for commit messages, rebase, etc.) |
| `git config --global init.defaultBranch main` | Sets default branch name for new repos |
| `git config --global core.autocrlf input` | Line-ending handling (`input` on Mac/Linux, `true` on Windows) |
| `git config --global alias.co checkout` | Creates a shortcut: `git co` = `git checkout` |
| `git config --global alias.st status` | Shortcut: `git st` = `git status` |
| `git config --global alias.lg "log --oneline --graph --all"` | Custom pretty log alias |
| `git config --list` | Show all effective config (local + global + system) |
| `git config --global --list` | Show only global config |
| `git config --local -e` | Open this repo's config file (`.git/config`) in editor |
| `git config --global --unset user.name` | Remove a global config value |
| `git config --get user.email` | Print the current value of one key |

---

## 2. Repository Basics

| Command | What it does |
|---|---|
| `git init` | Create a new local repo in the current folder |
| `git clone <url>` | Copy a remote repo to your machine |
| `git clone --depth 1 <url>` | Shallow clone — only latest commit (fast, small) |
| `git clone -b <branch> <url>` | Clone and checkout a specific branch directly |
| `git status` | Show staged/unstaged/untracked files |
| `git status -s` | Short-form status (compact output) |
| `git add <file>` | Stage a specific file |
| `git add .` | Stage all changes in current directory (incl. new files) |
| `git add -p` | Interactively stage **parts** of a file (hunk by hunk) |
| `git commit -m "message"` | Commit staged changes |
| `git commit -am "message"` | Stage **all tracked, modified** files + commit in one step (skips new/untracked files) |
| `git commit --amend` | Edit the **most recent** commit (see §9) |
| `git log` | Full commit history |
| `git log --oneline` | One line per commit (short hash + message) |
| `git log --graph --oneline --all` | Visual branch/merge history — extremely useful |
| `git log -p <file>` | Show the actual diff for each commit touching a file |
| `git log --author="name"` | Filter commits by author |
| `git log -n 5` | Show only the last 5 commits |
| `git show <commit-hash>` | Show full details + diff of one commit |
| `git diff` | Unstaged changes vs last commit |
| `git diff --staged` (or `--cached`) | Staged changes vs last commit |
| `git diff <branch1>..<branch2>` | Diff between two branches |
| `git diff HEAD~2 HEAD` | Diff between 2 commits ago and now |

---

## 3. Branching — Create, Switch, Delete, Rename

| Command | What it does |
|---|---|
| `git branch` | List local branches |
| `git branch -a` | List **all** branches (local + remote-tracking) |
| `git branch -r` | List only remote-tracking branches |
| `git branch <name>` | Create a new branch (doesn't switch to it) |
| `git checkout <name>` | Switch to an existing branch |
| `git checkout -b <name>` | Create **and** switch to a new branch in one step |
| `git switch <name>` | Modern equivalent of `checkout` for switching branches |
| `git switch -c <name>` | Modern equivalent of `checkout -b` |
| `git branch -m <old> <new>` | Rename a branch |
| `git branch -d <name>` | Delete a **local** branch (safe — refuses if unmerged) |
| `git branch -D <name>` | **Force**-delete a local branch (even if unmerged) |
| `git push origin --delete <name>` | Delete a **remote** branch |
| `git push origin :<name>` | Older syntax — same effect as above |
| `git branch --merged` | List branches already merged into current branch (safe to delete) |
| `git branch --no-merged` | List branches with unmerged work (careful before deleting!) |

---

## 4. 🔥 Recovering a Deleted Branch (Local & Remote)

This is the scenario everyone panics about. Git almost **never actually deletes commit data immediately** — deleting a branch just removes the *pointer/label*; the commits stay in Git's object database until garbage collection runs (usually 30–90 days later). Here's how to get it back.

| Scenario | Recovery command | Notes |
|---|---|---|
| **Deleted local branch**, you know it was recently checked out | `git reflog` → find the commit hash → `git checkout -b <name> <hash>` | `reflog` logs every HEAD movement for ~90 days, even after branch deletion |
| **Deleted local branch**, don't remember the hash | `git fsck --lost-found` (lists dangling commits) then `git branch <name> <hash>` | Use when `reflog` doesn't have it (older/GC'd) |
| **Deleted local branch immediately after deleting** | `git branch <name> <hash-from-delete-message>` | `git branch -d` prints the deleted commit's hash in its output — copy it immediately! |
| **Branch deleted on remote (e.g. GitHub)**, still exists locally | `git push origin <local-branch-name>` | Just re-push it — recreates the remote branch |
| **Branch deleted on remote AND locally, but a teammate has it** | Ask them to `git push origin <their-local-branch>` | Fastest recovery — someone else's clone still has it |
| **Branch deleted everywhere, but a PR/MR was opened from it** (GitHub/GitLab) | Open the PR page → look for "**restore branch**" button, or use the PR's last commit hash | GitHub keeps PR-associated commits reachable even after branch deletion |
| **Deleted remote branch, no one has a local copy, no PR exists** | `git reflog` on **any clone that had it checked out recently**, or check CI/CD build logs for the commit hash | Last resort — hash may be recoverable from build artifacts/logs |
| **Just want to see what reflog remembers right now** | `git reflog show --all` | Shows reflog for every ref, not just current HEAD |

> **Golden rule:** The moment you realize you deleted the wrong branch, **stop making new commits** and run `git reflog` immediately — every new HEAD movement pushes old entries further down (and eventually out of) the reflog.

---

## 5. `git reset` — Types & What Each One Actually Touches

`git reset` moves the **current branch pointer** to a different commit. The three modes differ in **how much they also touch your working directory and staging area**.

| Type | Command | Moves branch pointer? | Touches staging area (index)? | Touches working directory (files)? | Use when... |
|---|---|---|---|---|---|
| **Soft** | `git reset --soft <commit>` | ✅ Yes | ❌ No (keeps staged) | ❌ No (keeps files as-is) | You want to **undo a commit** but keep everything staged, ready to re-commit (e.g., squash commits manually) |
| **Mixed** *(default)* | `git reset --mixed <commit>` or just `git reset <commit>` | ✅ Yes | ✅ Yes (unstages) | ❌ No (files untouched, now "modified") | You want to undo a commit **and** unstage, but keep your edits in the working directory |
| **Hard** | `git reset --hard <commit>` | ✅ Yes | ✅ Yes | ✅ **Yes — deletes changes** | You want to **completely discard** everything after that commit. ⚠️ Destructive — uncommitted work is lost |

**Quick mental model:**
`--soft` = "undo the commit only" · `--mixed` = "undo the commit + the `git add`" · `--hard` = "undo the commit + the `add` + the actual edits"

| Related command | What it does |
|---|---|
| `git reset HEAD~1` | Undo the last commit, keep changes unstaged (mixed, most common) |
| `git reset HEAD <file>` | Unstage a specific file (doesn't touch commit history) |
| `git reset --hard origin/main` | Force local branch to exactly match remote (discards all local commits/changes) |
| `git reset --hard ORIG_HEAD` | Undo the last reset/merge/rebase itself (Git saves the previous HEAD here) |

### `git reset` vs `git revert` vs `git checkout`/`git restore`

| Command | Rewrites history? | Safe on shared/pushed branches? | What it does |
|---|---|---|---|
| `git reset` | ✅ Yes | ❌ **No** — avoid on pushed commits | Moves the branch pointer backward, optionally discarding changes |
| `git revert <commit>` | ❌ No — adds a **new** commit that undoes the change | ✅ **Yes** — safe for shared branches | Creates a new commit that reverses a previous one, preserving history |
| `git restore <file>` | ❌ No | ✅ Yes | Modern command to discard working-directory changes to a file |
| `git restore --staged <file>` | ❌ No | ✅ Yes | Modern replacement for `git reset HEAD <file>` (unstage only) |

---

## 6. Rebase vs Merge

Both combine work from two branches. They differ entirely in **how the history ends up looking**.

| Aspect | `git merge` | `git rebase` |
|---|---|---|
| History shape | Preserves true history — creates a **merge commit** with two parents | **Rewrites** history — replays your commits one-by-one on top of the target branch, linear result |
| Commit graph | Non-linear, shows exactly when branches diverged/joined | Linear, looks like the work happened sequentially (even if it didn't) |
| Safe on shared/pushed branches? | ✅ Always safe | ❌ **Never rebase commits others have already pulled** — it rewrites hashes |
| Conflict resolution | Resolve once, in a single merge commit | May need to resolve the **same** conflict repeatedly, once per replayed commit |
| Typical use case | Merging a finished feature branch into `main` | Cleaning up your **own** local/feature branch before opening a PR |
| Command | `git merge <branch>` | `git rebase <branch>` |
| Abort mid-operation | `git merge --abort` | `git rebase --abort` |
| Continue after resolving conflict | `git commit` (merge commit) | `git rebase --continue` |
| Skip a commit during rebase | N/A | `git rebase --skip` |

| Rebase variant | Command | Purpose |
|---|---|---|
| Standard rebase | `git rebase main` | Replay current branch's commits on top of latest `main` |
| **Interactive** rebase | `git rebase -i HEAD~5` | Edit/squash/reorder/drop the last 5 commits (see §9) |
| Rebase during pull | `git pull --rebase` | Fetch + rebase instead of fetch + merge (keeps history linear) |
| Preserve merge commits while rebasing | `git rebase -i --rebase-merges` | Rebase without flattening merge commits |

> **Rule of thumb:** *"Merge in public, rebase in private."* Never rebase a branch that others have already pulled/based work on.

---

## 7. `git fetch` vs `git pull`

| Aspect | `git fetch` | `git pull` |
|---|---|---|
| What it does | Downloads new commits/branches from remote **without** touching your working files or current branch | `git fetch` **+ automatic merge (or rebase)** into your current branch |
| Changes your files? | ❌ No | ✅ Yes — updates working directory |
| Risk level | Very safe — just updates `origin/*` tracking refs | Riskier — can trigger merge conflicts immediately |
| When to use | You want to **see** what's new before deciding what to do | You're confident and just want to update your current branch now |
| Command | `git fetch origin` | `git pull origin main` |
| Fetch then inspect before merging | `git fetch` → `git log origin/main` → `git merge origin/main` | (this *is* effectively what pull automates) |
| Pull with rebase instead of merge | N/A | `git pull --rebase` |
| Fetch all remotes | `git fetch --all` | N/A |
| Prune deleted remote branches locally | `git fetch --prune` (or `git fetch -p`) | — removes local `origin/*` refs for branches deleted on remote |

---

## 8. Merge Conflicts

| Step | Command | Notes |
|---|---|---|
| 1. Trigger | `git merge <branch>` or `git rebase <branch>` | Git pauses and marks conflicted files |
| 2. See which files conflict | `git status` | Conflicted files listed under "Unmerged paths" |
| 3. Open the file | — | Look for `<<<<<<<`, `=======`, `>>>>>>>` conflict markers |
| 4. Resolve manually | Edit the file, remove markers, keep the correct code | — |
| 4b. Or use a merge tool | `git mergetool` | Opens configured visual diff tool (e.g., VS Code, Meld, KDiff3) |
| 5. Mark as resolved | `git add <file>` | Staging = "I've resolved this file" |
| 6a. Finish a merge | `git commit` | Completes the merge commit |
| 6b. Finish a rebase step | `git rebase --continue` | Moves to next replayed commit |
| Abort entirely (merge) | `git merge --abort` | Returns to pre-merge state |
| Abort entirely (rebase) | `git rebase --abort` | Returns to pre-rebase state |
| Keep **only your** version for a file | `git checkout --ours <file>` | Then `git add <file>` |
| Keep **only their** version for a file | `git checkout --theirs <file>` | Then `git add <file>` |
| See conflict in context (3-way) | `git diff` (while conflicted) | Shows all 3 versions: base, ours, theirs |
| Show just conflicted file names | `git diff --name-only --diff-filter=U` | Quick list of unresolved files |

---

## 9. Editing Commit Messages

| Scenario | Command | Notes |
|---|---|---|
| Fix the **most recent** commit message | `git commit --amend -m "new message"` | Opens editor if `-m` omitted |
| Fix most recent commit **and** add forgotten changes | `git add <file>` → `git commit --amend --no-edit` | `--no-edit` keeps the old message |
| Edit an **older** commit message (not the latest) | `git rebase -i HEAD~N` → change `pick` to `reword` on that commit → save | Opens each marked commit's message for editing |
| Edit older commit's **content** too | `git rebase -i HEAD~N` → change `pick` to `edit` → make changes → `git commit --amend` → `git rebase --continue` | — |
| Squash last N commits into one | `git rebase -i HEAD~N` → change `pick` to `squash` (or `s`) on all but the first | Combines them, lets you write one combined message |
| ⚠️ Already pushed? | `git push --force-with-lease` | Required after amend/rebase on a pushed branch — `--force-with-lease` is safer than `--force` (fails if remote has commits you don't have locally) |

**Interactive rebase command reference** (`git rebase -i HEAD~N`):

| Keyword | Effect |
|---|---|
| `pick` | Keep commit as-is |
| `reword` | Keep commit content, edit its message |
| `edit` | Pause here to amend the commit (content + message) |
| `squash` | Merge into previous commit, combine messages |
| `fixup` | Merge into previous commit, **discard** this message |
| `drop` | Delete the commit entirely |

---

## 10. `git stash` — Full Command Set

| Command | What it does |
|---|---|
| `git stash` | Save uncommitted changes (staged + unstaged tracked files) and revert working dir to clean |
| `git stash -u` (or `--include-untracked`) | Also stash **untracked** new files |
| `git stash -a` (or `--all`) | Stash **everything**, including ignored files |
| `git stash save "message"` | Stash with a custom descriptive label |
| `git stash list` | Show all stashes (`stash@{0}`, `stash@{1}`, ...) |
| `git stash show -p stash@{0}` | View the actual diff inside a stash |
| `git stash pop` | Apply the most recent stash **and remove it** from the stash list |
| `git stash apply` | Apply the most recent stash but **keep it** in the list |
| `git stash apply stash@{2}` | Apply a specific (non-latest) stash |
| `git stash drop stash@{0}` | Delete a specific stash without applying it |
| `git stash clear` | Delete **all** stashes |
| `git stash branch <new-branch>` | Create a new branch from a stash (great when stash conflicts with current branch) |

---

## 11. Tags — Creating, Listing, Pushing, Deleting

| Command | What it does |
|---|---|
| `git tag` | List all tags |
| `git tag v1.0.0` | Create a **lightweight** tag (just a pointer, no metadata) on current commit |
| `git tag -a v1.0.0 -m "Release 1.0.0"` | Create an **annotated** tag (recommended — stores author, date, message, checksum) |
| `git tag -a v1.0.0 <commit-hash> -m "message"` | Tag a **specific past commit**, not just HEAD |
| `git show v1.0.0` | Show tag details + the commit it points to |
| `git push origin v1.0.0` | Push a **single** tag to remote |
| `git push origin --tags` | Push **all** local tags to remote |
| `git push origin --follow-tags` | Push commits + only **annotated** tags reachable from them |
| `git tag -d v1.0.0` | Delete a **local** tag |
| `git push origin --delete v1.0.0` | Delete a **remote** tag |
| `git checkout v1.0.0` | Check out the tagged commit (detached HEAD state) |
| `git tag --contains <commit>` | Find which tags include a given commit |

**Lightweight vs Annotated:**

| Type | Stores metadata? | Use case |
|---|---|---|
| Lightweight | ❌ No — just a name pointing to a commit | Quick, private/local bookmarks |
| Annotated | ✅ Yes — tagger name, date, message, GPG-signable | **Official releases** (v1.0.0, v2.1.3) — always prefer this |

---

## 12. Git Hooks — Running a `pre-commit` Hook

Git hooks are scripts Git runs automatically at specific points (commit, push, merge, etc.). They live in `.git/hooks/` and are **not** version-controlled by default (each clone must set them up, or you use a tool like `pre-commit` or `husky`).

| Step | Command / Action | Notes |
|---|---|---|
| 1. See available hook templates | `ls .git/hooks/` | Git ships `.sample` files for every hook type |
| 2. Create the hook | `touch .git/hooks/pre-commit` then edit it (shell script, Python, etc.) | File name must exactly match the hook name, **no extension** |
| 3. Make it executable | `chmod +x .git/hooks/pre-commit` | Required on Mac/Linux or the hook silently won't run |
| 4. Trigger it | `git commit -m "message"` | Runs **automatically** — no separate command needed. If the script exits non-zero, the commit is **aborted** |
| 5. Test/run it manually (without committing) | `.git/hooks/pre-commit` | Just execute the script directly to debug it |
| 6. Skip a hook for one commit | `git commit --no-verify` (or `-n`) | Bypasses `pre-commit` and `commit-msg` hooks — use sparingly |

**Using the `pre-commit` framework (industry standard, shareable via repo):**

| Command | What it does |
|---|---|
| `pip install pre-commit` | Install the framework |
| Create `.pre-commit-config.yaml` in repo root | Define which linters/checks to run (this file **is** version-controlled) |
| `pre-commit install` | Installs the actual git hook that calls the framework on every commit |
| `pre-commit run --all-files` | Manually run all configured checks against the whole repo (e.g., in CI) |
| `pre-commit autoupdate` | Bump hook versions in the config to latest |

**Common hook names you can use:**

| Hook | Fires when |
|---|---|
| `pre-commit` | Before a commit is created (lint/format checks) |
| `commit-msg` | After message is written — validate message format (e.g., Conventional Commits) |
| `pre-push` | Before pushing to a remote (run tests) |
| `post-merge` | After a merge completes (e.g., auto-run `npm install` if `package.json` changed) |
| `pre-rebase` | Before a rebase starts |

---

## 13. Remotes

| Command | What it does |
|---|---|
| `git remote -v` | List remotes with their URLs |
| `git remote add origin <url>` | Add a new remote named `origin` |
| `git remote remove origin` | Remove a remote |
| `git remote rename origin upstream` | Rename a remote |
| `git remote set-url origin <new-url>` | Change a remote's URL (e.g., HTTPS → SSH) |
| `git remote show origin` | Detailed info: tracked branches, ahead/behind status |
| `git push -u origin <branch>` | Push **and** set upstream tracking (so future `git push` needs no args) |
| `git push --force-with-lease` | Safer force-push — fails if remote changed since your last fetch |

---

## 14. Cherry-pick, Bisect, Blame, Clean

| Command | What it does |
|---|---|
| `git cherry-pick <commit-hash>` | Apply **one specific commit** from another branch onto current branch |
| `git cherry-pick <hash1>..<hash2>` | Apply a **range** of commits |
| `git cherry-pick --continue` / `--abort` | Handle conflicts during cherry-pick |
| `git bisect start` | Begin binary-search debugging to find which commit introduced a bug |
| `git bisect bad` / `git bisect good <hash>` | Mark commits during bisect; Git auto-checks out the midpoint each time |
| `git bisect reset` | End bisect session, return to original branch |
| `git blame <file>` | Show who last modified each line, and in which commit |
| `git blame -L 10,20 <file>` | Blame only lines 10–20 |
| `git clean -n` | **Dry run** — preview untracked files that would be deleted |
| `git clean -fd` | Actually delete untracked files **and** directories |
| `git clean -fdx` | Also delete files ignored by `.gitignore` (use carefully) |

---

## 15. "Oh No" — Real-World Recovery Cheat Sheet

| "Oh no, I..." | Fix |
|---|---|
| ...committed to the wrong branch | `git reset HEAD~1` (undo commit, keep changes) → `git stash` → `git checkout correct-branch` → `git stash pop` → commit again |
| ...want to undo my last commit but keep the changes | `git reset --soft HEAD~1` |
| ...want to completely wipe my last commit and changes | `git reset --hard HEAD~1` |
| ...already pushed a bad commit and need it gone from history | `git revert <hash>` (safe) — avoid `reset --hard` + force-push unless you're sure no one else pulled it |
| ...have uncommitted changes but need to switch branches | `git stash` → switch → `git stash pop` |
| ...need to find a commit I "lost" after reset/rebase | `git reflog` → `git checkout <hash>` or `git branch recovery-branch <hash>` |
| ...want to discard all local uncommitted changes | `git restore .` (or `git checkout -- .` on older Git) |
| ...want my branch to exactly match remote, discarding local commits | `git fetch origin` → `git reset --hard origin/<branch>` |
| ...accidentally deleted a file and committed it | `git checkout <commit-before-delete> -- <file>` |
| ...merged the wrong branch | `git reset --hard ORIG_HEAD` (only if not yet pushed) |
| ...want to see exactly what changed in my last commit | `git show HEAD` |
| ...have a detached HEAD and want to keep the work | `git checkout -b new-branch-name` (right now, before switching away!) |

---

## 16. Quick Command Index (A–Z style)

| Task | Command |
|---|---|
| Check current branch | `git branch --show-current` |
| Compare 2 branches | `git diff branch1..branch2` |
| Count commits | `git rev-list --count HEAD` |
| Find commit that introduced a line | `git log -S "search text" -- file` |
| Ignore a file already tracked | `git rm --cached <file>` (then add to `.gitignore`) |
| List files in a commit | `git show --stat <commit>` |
| See who tracks what upstream | `git branch -vv` |
| Shallow → full clone | `git fetch --unshallow` |
| Squash all history into one commit | `git reset $(git commit-tree HEAD^{tree} -m "message")` |
| Undo `git add` (unstage all) | `git reset` |
| View file at an old commit | `git show <commit>:<path/to/file>` |
