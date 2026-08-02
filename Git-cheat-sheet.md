# List of common Git commands

## Git config files
1. Add a global username to git config: `git config --global user.name "Your Name"`
2. To add username local to a repo: `git config user.name "Your Name"`
3. Add a global email: `git config --global user.email "you@example.com"`
4. Create git alias commands (windows): `git config --global alias.gcob "checkout -b"`
5. Create git alias commands (unix): `git config --global alias.gcob 'checkout -b'`  
6. List of all configs: `git config --list` or `git config --list --show-origin`  
[Click here](https://gist.github.com/kanwarpannu/9f0e589295a21338e27a791b3095557c) for sample windows git config file.  

## Git credential manager
Git has 5 "main" modes of storing credentials for a repository.  
1. **Default** - Don't store. Every connection will ask for a username and password.  
2. **Cache** - Store credentials in memory for a small amount of time. Default timeout is 15 min.  
`git config --global credential.helper cache`
OR  
`git config --global credential.helper 'cache --timeout 9000'` (Time in seconds) 
3. **Store** - Store credentials to disk in a plain file.  
`git config --global credential.helper store`  
OR  
`git config --global credential.helper 'store --file <dir>'`  
4. **Osxkeychain** (only on mac) - Store in apple keychain.  
`git config --global credential.helper osxkeychain`  
5. **Windows credential manager** (windows) - Store in a windows credential manager (Search for credential manager in start menu).  
`git config --global credential.helper manager`  

## Git ignore files
How to remove git ignored files from repo:  
1. Create and commit the .gitignore file in repo  
2. Remove all files that are to be removed from tracked status(on basis of .gitignore file): `git rm -r --cached .`  
3. `git add .`  
4. `git commit -m "fixed .gitignore files"`  
5. `git push`  
[Click here](https://gist.github.com/kanwarpannu/644b4dc3d9703e27c06c2e24967e41b6) for sample git ignore file.  

## Clone
1. Clone a repo: `git clone http://repo.url`  
2. Clone a specific branch: `git clone -b <branch-name> http://repo.url`  
3. Clone an authenticated repo: `git clone http://username@repo.url`  
4. Create a new repo (takes name of parent folder): `git init .`  

## Remote
1. See upstream branch access and info: `git remote -v`  
2. Add upstream repo:  `git remote add origin <remote-url>`  

## Upload local repo
Create a repo on github or bitbucket from the website. Take its url and run the following commands.  

```
git init
git add .
git commit -m "<message>"
git remote add origin <remote-url>
git remote -v
git push origin master
```

## Basic Commands
1. Clone git repos: `git clone https://repo.url`  
2. Stage files: `git add <file-name>`  
3. Commit files: `git commit -m "commit-message"`  
4. Push commits to remote: `git push`  
5. Force push commits: `git push --force` or `git push --force-with-lease`  
6. Switch branches: `git checkout <branch-name>`  
7. Move to a particular commit: `git checkout <commit-hash-code>`  
8. Pull the latest commits on to current branch from remote: `git pull`  

## Git add
1. Stage files: `git add <file-name1> <file-name2>`  
2. Stage all files: `git add .`  
3. Stage files in patch mode:  
```
git add -p
     y(add current hunk)
     n(skip current hunk)
     q(do not stage current and rest of hunks)
     a(stage all hunks)
     s(split current hunk)
     ?(print help)
```

## Working with log, status, diff and show
1. See current non-committed stages in branch: `git status` or `git status -suno`  
2. See recent commits: `git log`  
3. See detailed log info: `git log --oneline --graph --decorate`  
4. See changes in staged files: `git diff`  
5. See changes in local committed files: `git diff --cached`  
6. See changes in a commit: `git diff <commit-hash-code>`  
7. See a commit: `git show <commit-hash-code>`  
8. See how many changes a current branch is ahead: `git rev-list --left-right --count <origin/master>...<master>`  

## Stash
1. Quick save in a stash: `git stash`  
2. Save a commit to stash with a commit message: `git stash save "this is a message"`  
3. Check list of commits in stash: `git stash list`  
4. Apply a commit (say commit 2) from stash: `git apply stash@{2}`  
5. Apply top commit from the stash and drop that commit: `git stash pop`  
6. Drop a commit from stash: `git stash drop stash@{1}`  
7. Clear all commits from stash: `git stash clear`  

## Working with new branch
1. Create a new local branch: `git checkout -b <branch-name>`  
2. Push the branch to remote: `git push -u origin <branch-name>`  

## Git Worktree
A worktree lets you check out multiple branches at the same time in separate directories — all linked to the same single repo clone on disk.  

**How it differs from git branch:**  
- `git checkout <branch>` swaps your one working directory between branches — you can only actively work on one at a time, and you must stash or commit before switching.  
- `git worktree add` creates an additional working directory with its own branch checked out independently — both are live at the same time, no switching or stashing needed.  

**Commands:**  
1. Create a worktree on an existing branch: `git worktree add <path> <branch-name>`  
2. Create a worktree with a new branch: `git worktree add -b <new-branch-name> <path>`  
3. List all active worktrees: `git worktree list`  
4. Remove a worktree: `git worktree remove <path>`  
5. Clean up stale references (if directory was deleted manually): `git worktree prune`  

**Lifecycle:**  
```
// Step 1 - Create: check out a branch into a new directory alongside your main repo
git worktree add ../hotfix hotfix/login-bug

// Step 2 - Work: cd in and use git normally — add, commit, push all work as usual
cd ../hotfix
git add .
git commit -m "fix login bug"
git push

// Step 3 - Remove: when done, remove the worktree (deletes the directory and the link)
cd ../main-repo
git worktree remove ../hotfix

// Step 4 - Prune (optional): if you deleted the directory manually instead of using remove
git worktree prune
```

**Example — work on a hotfix while keeping your feature branch untouched:**  
```
// You're mid-feature on main repo, urgent hotfix needed
git worktree add ../hotfix hotfix/urgent-fix

// Fix the bug in ../hotfix without touching your feature work
cd ../hotfix
// ... make changes ...
git commit -am "hotfix: fix urgent bug"
git push

// Back to your feature, nothing disturbed
cd ../main-repo
git worktree remove ../hotfix
```

## Merge
Normally you would want to raise a pull request to merge two branches.  
However, it can also be done manually using merge.

Here we are merging <branch1 to master>
```
git checkout master
git merge <branch1>
git branch -d <branch1>  //delete the branch if not needed
```
The default behaviour of "git merge" is to fast-forward, that means only head pointer is moved.
However, if the head and incoming branch are out of sync it does three way merge (which creates another commit).
Git handles both these cases internally we don't have to do anything unless there is a merge conflict.  
[Click here](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) for the link to right page.

## Rebase
Here we are rebasing of the master branch on our current branch.
```
git checkout master
git pull 
git checkout <branch-name>  
git pull
git rebase master  
```  
OR  
```
git checkout <branch-name>
git pull
git fetch origin/master
git rebase origin/master
```  
If you run into merge conflicts, resolve the issues manually and run following command to continue rebase: `git rebase --continue`  
Or  
Run this to cancel rebase: `git rebase --abort` 

## Reset and Revert
1. Revert a previous commit (creates a new commit saying old commit reverted, used only if a commit is pushed to master): `git revert <commit-hash-code>` 
2. Remove un-staged changes permanently: `git reset --hard`  
3. Reset back 2 commits without losing data: `git reset HEAD~2 --soft`
4. Reset back 2 commits losing all data: `git reset HEAD~2 --hard`  
5. Reset everything to remote branch:
```
git fetch --all
git reset --hard origin/master
```
OR (to have local changes as un-staged)    
```
git fetch --all
git reset --soft origin/master
```  

## Amend and Squash commits
1. To append the current changes to a past commit: `git commit --amend` or `git commit --amend -m "This is message"` or `git commit --amend --no-edit`  
2. To amend the author: `git commit --amend --author="Author Name <email@address.com>"`  
3. To squash previous commits use rebase in the interactive mode: `git rebase -i HEAD~<number-of-commit>`  
**Example — squash last 3 commits into one:**  
```
git rebase -i HEAD~3
```
An editor opens listing the last 3 commits (oldest at top):
```
pick abc1234 First commit message
pick def5678 Second commit message
pick ghi9012 Third commit message
```
Change `pick` to `s` (squash) on every commit you want merged into the one above it.  
Keep the top commit as `pick` — that is the one everything squashes into:
```
pick abc1234 First commit message
s def5678 Second commit message
s ghi9012 Third commit message
```
Save and close. A second editor opens to write the final combined commit message. Edit it, save, and close — done.  
**Shorthand options for the editor:**  
- `pick` / `p` — keep commit as-is  
- `squash` / `s` — squash into commit above, prompts to combine messages  
- `fixup` / `f` — squash into commit above, silently discards this commit's message  
- `reword` / `r` — keep commit but edit its message  


## Cherry-pick
1. To cherry pick a commit from one branch to another first checkout the branch you want to be: `git cherry-pick -x <commit-hash-code>` (here -x auto-populates from where commit was taken)  

## Rename Branch
1. To rename current local branch: `git branch -m <new-branch-name>`  
2. To rename other local branch: `git branch -m <old-branch-name> <new-branch-name>`  
3. To rename a remote branch:  
```
git branch -m <new-branch-name> //Rename the local branch

git push origin --delete <old-branch-name> //Then delete remote branch

git push origin -u <new-branch-name> //Push new branch
//Dont forget to unset the old upstream: git checkout <new-branch-name> ; git branch --unset-upstream
```

## Remove branch and Prune
1. Remove a branch locally (fails if branch not merged yet. Fix: use -D instead to force delete): `git branch -d <branch-name>`  
2. Remove a remote branch: `git push origin --delete <branch-name>`
3. To check branches no longer accessed or detached commits: `git prune --dry-run --verbose`  
4. To remove detached commits: `git prune`  
5. To remove local branches no longer in remote (only the un-tracked are removed, new branches aren't touched): `git remote prune` or `git fetch --prune`

## Git Tags
1. Tags are created to mark a historic point or mark release in git history. We can create by going to the said branch: `git tag -a <tag-name> -m "tag-message"`  
2. To see all tags using power of wild char *: `git tag -l "v1.*"`  
3. Push tag to origin: `git push origin <tag-name>`  
4. Delete tags: `git tag -d <tag-name>` or `git push origin -d <tag-name>`  
5. Create tag at past commit: `git tag <tag-name> <commit-number>`  
6. Checkout branch from a tag: `git checkout -b <branch-name> <tag-name>`  

## Patch
Create a patch(always of a single commit, a patch looks like unix email structure):  
1. Create a list of patches with respect to a target branch: `git format-patch <new-branch> -o directory/where-patch-is-saved`  
2. Create a patch for one commit: `git format-patch -1 <hash-code> -o dir`  
3. Apply patches with am(apply mail): `git am dir/<patch-name>`  

## Reflog
1. To see the history of HEAD on a local branch (reflog only stores logs locally): `git reflog`  
2. To see all the changes: `git reflog show --all`  
3. To see changes in other branch: `git reflog show <branch-name>`  
4. To move back to a commit using reflog: `git reset HEAD@{<head-position-in-reflog-in-number>} --soft`  
