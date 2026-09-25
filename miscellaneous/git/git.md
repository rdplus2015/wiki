
# Git Commands

## Installation
```bash
sudo apt update
```
```bash
  sudo apt install git
```
```bash
  git --version
```
--- 

## Configuration and Initialization

- Set username:
  ```bash
  git config --global user.name "user_name" # use --local for specific to a repository
  ```

- Set email:
  ```bash
  git config --global user.email "my@email.com" # use --local for specific to a repository
  ```

- Show all configurations:
  ```bash
  git config --global --list # use --local for specific to a repository
  ```

- Get help:
  ```bash
  git --help
  ```

- Set default branch name for new repositories:
  ```bash
  git config --global init.defaultBranch <branch_name>
  ```

- Rename default branch to `main`:
  ```bash
  git branch -m main
  ```

## SSH Key Configuration for GitHub

- Generate an SSH key:
  ```bash
  ssh-keygen
  ```

- Display the SSH public key:
  ```bash
  cat ~/.ssh/id_rsa.pub
  ```
---
## Basic Commands

### Repository Initialization 
- Initialize a Git repository:
  ```bash
  git init
  ```

- Check the status of the repository:
  ```bash
  git status
  ```

### Staging Area

- Add a file to the staging area:
  ```bash
  git add <file_name> #  or git add file1.txt file2.txt 
  ```

- Add all modified files to the staging area:
  ```bash
  git add . # or git add *.txt 
  ```

- Remove files from the staging area:

  ```bash
  git restore --staged <files>  # Part of new commands introduced to clarify Git operations 
  git reset  # Older command and can do other more complex operations on commits.
  
  #removes the specified files from the index, unstaging them, but keeps their changes in the working tree (the file is still modified but not prepared for commit).
  ```
- removes files from the staging area
  ```bash
  git rm -r --cached <files> 
  # This command removes files from the staging area (index) and stops tracking files in Git version tracking, while leaving the files intact on disk.
  # Use this command when you want Git to stop tracking a file that is already versioned.
  ```
### Commit 

- Create a commit with a message:
  ```bash
  git commit -m "enter your message here"  # or git commit 
  ```

- See differences between the working directory and the index:
  ```bash
  git diff <file_name> 
  # Shows you what you modified in your file before doing git add command
  # This will show you the differences between the current version of the file in your working directory and the one in the last commit.
  ```

- See differences after staging:
  ```bash
  git diff --cached # Shows you what is already in the staging area and ready to be committed in the next commit 
  ```
- Remove a Commit that Has Not Been Pushed 

  If you've committed changes locally that you don't want to keep, you can remove them before pushing.
  **Reset to the Previous Commit**
     ```bash
     git reset --soft HEAD~1
     ```

     -  use `--soft` keeps your changes staged.
     -  Use `--mixed` to unstage the changes.
     -  Use `--hard` to discard the changes completely.

     Warning: Using `--hard` will permanently delete your changes.


- Remove a Commit that has been pushed
If you have already pushed a commit to GitHub and want to remove it, you'll need to **force push** the changes.

  1. **Remove or rebase the commit locally**:
     - Use `git reset` to remove the commit from your local history.
     - You can also use `git rebase` if you need to modify the commit history.

  2. **Force push to the remote repository**:
     Since you're rewriting the history, you will need to use the `--force` option when pushing to the remote repository:
     ```bash
     git push origin <branch-name> --force
     ```
     - This ensures that you won't accidentally overwrite changes made by others.
     ```bash 
     git push origin <branch-name> --force-with-lease
     ```
### Safety Tips

- Use `git reflog` to recover from accidental resets.
- Always backup important changes before performing destructive operations.
- Communicate with your team when rewriting commit history in shared repositories.
- Use  [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) or `git filter-branch` to rewrite Git history.


### Revert to a Previous Commit

Reverting to a previous commit can be done if you want to discard recent changes and start from a specific commit.

1. **Find the Commit Hash**

   First, identify the hash of the commit you want to revert to. You can use the following command to list your commits:

   ```bash
   git log --oneline
   ```
2. **Checkout to the Specific Commit** 

   Once you have identified the commit hash, you can check out to that specific commit using:
   ```bash
   git checkout <commit-hash>
   ```
3. **Create a New Branch from the Previous Commit**

   It is a good practice to create a new branch from this commit so that you can continue working from this point:

   ```bash
   git checkout -b <new-branch-name>
   ```
---

## History, Inspection and Tag

- Show commit history:
  ```bash
  git log # or git log -n <number> (Show a specific number of commits:)
  ```

- Show details of a specific commit:
  ```bash
  git show <commit_SHA>
  ```

- Checkout an old commit:
  ```bash
  git checkout <commit_SHA> # or git checkout main (Checkout the latest commit on the default branch)
  ```

- Tag a commit:
  ```bash
  git tag <tag_name>
  ```

- Delete a tag:
  ```bash
  git tag --delete <tag_name>
  ```

- List all tags:
  ```bash
  git tag
  ```
---

## Working with Remote Repositories

- Clone a repository:
  ```bash
  git clone <repository_url>
  ```

- Show remote URLs:
  ```bash
  git remote -v
  ```

- Add a remote:
  ```bash
  git remote add origin <repository_url>
  ```

- Update the remote URL:
  ```bash
  git remote set-url origin <new_url>
  ```

- Push to a remote repository for the first time:
  ```bash
  git push -u origin main # git push --set-upstream origin feature-branch 
  ```
  ```bash
  git push  # Git will automatically know which remote branch to use.
  ```

- Push tags to a remote repository:
  ```bash
  git push origin --tags
  ```

- Fetch changes from a remote repository:
  ```bash
  git fetch
  ```

- Pull changes from a remote repository:
  ```bash
  git pull
  ```

- Get information about a file (useful for collaboration):
  ```bash
  git blame <file.extension>
  ```
---

## Branching

- List all branches:
  ```bash
  git branch
  ```

- List all branches including remote branches:
  ```bash
  git branch -a
  ```

- Create a new branch:
  ```bash
  git branch <branch_name>
  ```

- Switch to a branch:
  ```bash
  git checkout <branch_name>
  ```

- Create and switch to a new branch:
  ```bash
  git checkout -b <branch_name>
  ```

- Delete a local branch:
  ```bash
  git branch -d <branch_name>
  ```

- Delete a remote branch:
  ```bash
  git push origin --delete <branch_name>
  ```

- Cherry-pick a commit:
  ```bash
  git cherry-pick <commit_SHA>
  # If someone made an important commit in another branch 
  # and you need it on your branch without wanting to merge all the other changes from that branch.
  ```
  
---

## Stashing

- Save changes temporarily:
  ```bash
  git stash # This moves uncommitted changes to a temporary stack.
  ```

- List stashed changes:
  ```bash
  git stash list # Displays the list of saved stashes.
  ```

- Apply stashed changes:
  ```bash
  git stash apply # This applies changes from the last stash without removing it from the stack.
  ```

- Apply and remove stashed changes:
  ```bash
  git stash pop # This applies the changes from the last stash and removes it from the stack.
  ```

- Drop a specific stash:
  ```bash
  git stash drop <stash@{n}> # Replace n with the number of the stash you want to delete.
  ```

- Clear all stashes:
  ```bash
  git stash clear
  ```
  
  ### Advanced Usage
- Stash with a descriptive message:
  ```bash
  git stash save "Description"
  ```
- Stash only tracked files:

  ```bash
  git stash -k
  ```
- Stash including untracked files:

  ```bash
  git stash -u
  ```

## Merging and Rebasing

- Merge branches:
  ```bash
  git merge <branch_name>
  # Keep the history of both branches and create a merge commit (contains changes from two parents) with two parents
  # The history may become more complex with divergent commits.
  ```

- Rebase branches:
  ```bash
  git rebase <branch_name>
  # Rewrites history by replaying branch commits on the top of the target branch and creates new commits with different references.
  # produces a linear and cleaner history.
  ```

