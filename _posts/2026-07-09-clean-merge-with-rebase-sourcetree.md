---
title: "Merging a Feature Branch Cleanly (No Merge Commit) — A Git Rebase Walkthrough in Sourcetree"
date: 2026-07-09 00:00:00 +0900
categories: [Git, Workflow]
tags: [git, rebase, merge, sourcetree, branch]
---
# Merging a Feature Branch Cleanly (No Merge Commit) — A Git Rebase Walkthrough in Sourcetree

When you merge a long-lived feature branch into `master`, Git normally creates an extra **merge commit** — something like *"Merge branch 'feature/xyz' into master"*. Sometimes you want to avoid that and keep your history perfectly linear. Here's how to do it, both from the command line and in Sourcetree's GUI, along with how to safely undo the whole thing if needed.

## The Problem

Picture a `master` branch and a `feature/001-modify-code` branch that diverged a while ago. Since then:

- `master` picked up its own new commits (env config changes, refactors, docs, etc.)
- `feature/001-modify-code` picked up its own commits (new API endpoints, bug fixes, etc.)

```
master:    A---B---C
                \
feature:         D---E
```

If you just run `git merge feature/001-modify-code` on `master`, Git *has* to create a merge commit, because the two histories diverged and need to be tied back together.

## The Solution: Rebase, Then Fast-Forward Merge

The trick is to **replay the feature branch's commits on top of master's latest commit first**. Once that's done, merging becomes a simple fast-forward — master's pointer just moves forward, no extra commit needed.

```
Before rebase:
master:    A---B---C
                \
feature:         D---E

After rebase (feature branch is rewritten):
master:    A---B---C
                     \
feature:              D'---E'

After fast-forward merge:
master, feature:    A---B---C---D'---E'
```

### Step 1 — Rebase the feature branch onto master

```bash
git checkout feature/001-modify-code
git fetch origin
git rebase master
```

This takes every commit that master has that the feature branch doesn't, and puts the feature branch **on top of them**. If there are conflicts, Git will pause at the conflicting commit:

```bash
# resolve conflicts in the file(s), then:
git add .
git rebase --continue

# or, to bail out entirely and go back to the pre-rebase state:
git rebase --abort
```

Since rebase happens one commit at a time, you may need to resolve conflicts more than once if there are multiple commits.

### Step 2 — Fast-forward merge into master

```bash
git checkout master
git merge --ff-only feature/001-modify-code
```

`--ff-only` tells Git "only do this merge if it can be a fast-forward — otherwise stop and error out." This is a safety net: if something went wrong during the rebase and a true fast-forward isn't possible, you won't accidentally end up with a merge commit anyway.

### Step 3 — Push, carefully

- `master` can be pushed normally, since it just moved forward:
  ```bash
  git push origin master
  ```
- The feature branch, however, now has **different commit hashes** than what's on the remote (rebase rewrites history). A normal `git push` will be rejected, so you need:
  ```bash
  git push origin feature/001-modify-code --force-with-lease
  ```
  `--force-with-lease` is safer than a plain `--force` — it only overwrites the remote branch if nobody else has pushed to it since you last fetched, protecting you from accidentally wiping out a teammate's work.

## ⚠️ A Word of Caution About Rebase

Rebase **rewrites commit history** (new hashes for every replayed commit). If you're the only one working on that feature branch, this is perfectly safe. But if teammates have already pulled that branch and are working on it too, rebasing will cause their local history to diverge from yours, leading to confusing conflicts. In shared-branch situations, consider a **squash merge** instead:

```bash
git checkout master
git merge --squash feature/001-modify-code
git commit -m "feat: add EDMS document management insert API"
```

This condenses all of the feature branch's commits into a single commit on master — no merge commit, and no history rewriting on the feature branch either (though you do lose the fine-grained commit history).

## Doing This in Sourcetree (GUI)

### 1. Rebase the feature branch onto master
1. Double-click `feature/001-modify-code` in the sidebar to check it out.
2. Right-click on the commit that `master` points to in the graph.
3. Choose **"Rebase..."** (in Korean UI: **"재배치..."**).
4. Confirm. If conflicts appear, resolve them in the flagged files, stage them, and click **"Continue Rebase"**. Repeat as needed, or click **"Abort Rebase"** to cancel.

Once done, the feature branch should appear directly in line with `master` in the graph — no divergent branch lines.

### 2. Merge into master
1. Double-click `master` to check it out.
2. Right-click on the feature branch's latest commit.
3. Choose **"Merge..."**.
4. If there's a checkbox like *"Create a new commit even if fast-forward is possible,"* make sure it's **unchecked** — this keeps the merge a true fast-forward with no extra commit.
5. Confirm.

### 3. Verify success
After merging, `master` and `feature/001-modify-code` should point to the **exact same commit** in the graph, with a single unbroken line — no diamond-shaped merge pattern.

### 4. Push
- `master`: push normally.
- Feature branch: push with **"Force push"** (or "force with lease," if your Sourcetree version offers it) checked, since the commit hashes changed.

## Undoing Everything (If You Haven't Pushed Yet)

If you did all of this locally and haven't pushed anything yet, the remote (`origin`) still has the original, pre-rebase state — so reverting is simple.

**Command line:**
```bash
git checkout master
git reset --hard origin/master

git checkout feature/001-modify-code
git reset --hard origin/feature/001-modify-code
```

**Sourcetree GUI:**
1. Check out `master`, right-click on the commit `origin/master` points to, choose **"Reset current branch to this commit"** with the **"Hard"** option.
2. Repeat the same for `feature/001-modify-code`, resetting to `origin/feature/001-modify-code`.

⚠️ **Warning:** `reset --hard` discards any uncommitted local changes too — make sure you don't have unsaved work before running it. If you want a safety net, note the current commit hash (or check `git reflog`) before resetting, so you can recover the rebased state later if needed.

## Summary

| Goal | Command / Sourcetree Action |
|---|---|
| Replay feature commits onto master | `git rebase master` (on feature branch) / Sourcetree: right-click master's commit → **Rebase** |
| Merge without a merge commit | `git merge --ff-only feature-branch` (on master) / Sourcetree: right-click feature commit → **Merge**, uncheck "always create a commit" |
| Push feature branch after rebase | `git push origin feature-branch --force-with-lease` / Sourcetree: **Push** with **Force push** checked |
| Undo everything (not yet pushed) | `git reset --hard origin/<branch>` on each branch / Sourcetree: **Reset current branch to this commit → Hard** |

This workflow keeps your Git history linear and readable — no clutter from automatic merge commits — while still preserving every individual commit from the feature branch. Just remember: rebase is safe for solo branches, but coordinate with your team (or use squash merge instead) if others are working on the same branch.
