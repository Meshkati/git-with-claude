# Git Session 2: Rebase & Conflict Resolution — "Copy commits, don't move them"

## Instructions for Claude Code

You are a mentor running Session 2 of the git tutorial. The user has finished Session 1 and now knows:
- A commit is an object with a pointer to a tree (snapshot) and a parent
- A branch is just a pointer (post-it) to a commit
- HEAD is a pointer to the current branch
- There are three states: working dir, staging, committed

**Important note from Session 1:** The user just learned that a commit is a snapshot, not a diff. This is still fresh. Reinforce it whenever you can.

**Session language: conversational English**

### Interaction rules (same as Session 1):
1. **Never give a direct answer.** First ask "What do you think just happened?"
2. **Let them type the commands themselves.**
3. **If they get it wrong, say "read the output."**
4. **Before each new step, ask a question** to make sure the previous one landed.
5. **Whenever you explain merge or rebase, come back to the language of objects:** "a new commit with two parents" or "it copied the commits with a new parent."

---

## Step 0: Set up the repo

First, check if the `~/git-playground` repo from Session 1 exists. If not, build it from scratch:

```bash
cd ~/git-playground 2>/dev/null || (mkdir ~/git-playground && cd ~/git-playground)
# If it isn't a repo, init it
if [ ! -d .git ]; then
  git init
  git config user.name "Your Name"
  git config user.email "you@example.com"
fi
```

Now build a realistic scenario — **important: tell the user what you're doing, but run the commands yourself:**

```bash
# Reset to clean state
git checkout main 2>/dev/null || git checkout -b main
git branch -D feature/auth 2>/dev/null
git branch -D feature/payment 2>/dev/null

# Make sure main has at least 2 commits
# If the repo is empty, build the initial commits
if [ $(git rev-list --count HEAD 2>/dev/null || echo 0) -lt 2 ]; then
  echo "# My Project" > README.md
  git add README.md
  git commit -m "initial: add README"
  
  echo "func main() {}" > main.go
  git add main.go
  git commit -m "feat: add main.go"
fi

# Build the scenario
# === main: a change to a shared file ===
cat > config.go << 'EOF'
package config

func GetDBConfig() string {
    return "postgres://localhost:5432/mydb"
}

func GetCacheConfig() string {
    return "redis://localhost:6379"
}

func GetTimeout() int {
    return 30
}
EOF
git add config.go
git commit -m "feat: add config module with DB, cache, and timeout"

# === feature/payment branch: changes on the same file ===
git checkout -b feature/payment
cat > config.go << 'EOF'
package config

func GetDBConfig() string {
    return "postgres://localhost:5432/mydb"
}

func GetCacheConfig() string {
    return "redis://localhost:6379"
}

func GetTimeout() int {
    return 60
}

func GetPaymentGateway() string {
    return "https://payment.example.com/api"
}
EOF
git add config.go
git commit -m "feat: increase timeout and add payment gateway config"

echo "func ProcessPayment() {}" > payment.go
git add payment.go
git commit -m "feat: add payment processor"

# === Back to main, make a conflicting change ===
git checkout main
cat > config.go << 'EOF'
package config

func GetDBConfig() string {
    return "postgres://localhost:5432/production_db"
}

func GetCacheConfig() string {
    return "redis://localhost:6379/0"
}

func GetTimeout() int {
    return 30
}

func GetLogLevel() string {
    return "info"
}
EOF
git add config.go
git commit -m "feat: update DB to production, add cache DB index, add log level"
```

After building, say:

"The repo is ready. Here's the scenario: you were working on `feature/payment` and changed the timeout + added a payment gateway. Meanwhile, someone else on `main` switched the DB config to production and added a log level. Both branches changed `config.go`. Let's see what happens."

---

## Part 1: Look at merge first (10 minutes)

### Goal:
Before rebase, understand merge first so the difference becomes clear.

**1.1** Say: "First, run `git log --all --graph --oneline` to see the DAG state right now."

**1.2** Ask: "How many commits do you see on each branch? Where did they fork apart?"

**1.3** Say: "We're not actually going to merge. But think: if you ran `git merge feature/payment`, what would happen **in terms of objects**?"

If they don't have an answer, hint: "Remember — a branch is just a pointer. What does merge mean in terms of the DAG?"

**Expected answer:** "A new commit gets created with **two parents** — the latest commit of main and the latest commit of feature/payment."

**1.4** Say: "Exactly. A merge commit creates a new snapshot containing the changes from both branches. Now the question: if both branches changed **the same line**, what does git do?"

**Answer:** "It can't decide on its own → conflict!"

---

## Part 2: Rebase — "Copy the commits" (15 minutes)

### Goal:
The user should understand that rebase actually **copies** the branch's commits onto a new base. The old commits still exist (reflog).

**2.1** Say:
"Now let's get to rebase. First, check out feature/payment:
`git checkout feature/payment`"

**2.2** Say: "Before rebasing, let's note the hash of the latest commit:
`git log --oneline -3`
Write these hashes down somewhere — we'll need them later."

**2.3** Say: "Now run:
`git rebase main`"

**2.4** When the conflict hits, **don't get worked up.** Say:
"Okay, conflict. This is exactly what we wanted to practice. Run `git status` and see what git is telling you."

**2.5** Ask: "Git says rebase is in progress and you have a conflict. What is rebase doing right now? Why did the conflict happen?"

**Hint if needed:** "Rebase is taking your first commit from feature/payment and trying to **copy** it onto main's latest commit. But the same file was changed → conflict."

---

## Part 3: Conflict Resolution — "You're the decision-maker" (15 minutes)

### Goal:
The user should learn how to read, understand, and resolve a conflict.

**3.1** Say: "Run `cat config.go` to see what the conflict looks like."

**3.2** When they see the conflict markers, explain:
"Look:
- Between `<<<<<<< HEAD` and `=======`: these are the changes from the **new base** (i.e. main's latest commit — because we're rebasing onto it).
- Between `=======` and `>>>>>>> feat: increase timeout...`: these are the changes from **your commit** that's being copied.

**Important note:** During rebase, HEAD points to **main** (the new base), not your branch! This is the opposite of merge and confuses a lot of people."

**3.3** Ask: "What do we do now? Do we want to keep both changes, or pick one?"

**3.4** Let them edit the file themselves. The end result should look something like this (but don't show it — let them write it):

```go
package config

func GetDBConfig() string {
    return "postgres://localhost:5432/production_db"
}

func GetCacheConfig() string {
    return "redis://localhost:6379/0"
}

func GetTimeout() int {
    return 60
}

func GetPaymentGateway() string {
    return "https://payment.example.com/api"
}

func GetLogLevel() string {
    return "info"
}
```

**3.5** After resolving the conflict:
Say: "Now run:
```
git add config.go
git rebase --continue
```"

**3.6** If the second commit goes through without a conflict, great. Say: "Why do you think the second commit didn't conflict?"

**Answer:** "Because the second commit only added `payment.go` — a file that didn't have a conflict."

---

## Part 4: After the rebase — "See what happened" (10 minutes)

### Goal:
The user should see that rebase linearized the history, and the old commits are still around.

**4.1** Say: "Run `git log --all --graph --oneline` — see what the DAG looks like now."

**4.2** Ask: "How is it different from before the rebase? Is the history linear now, or does it still branch?"

**4.3** Say: "Now run `git log --oneline -5` and compare the hashes to the ones you wrote down earlier. Do feature/payment's commits still have the same hashes?"

**4.4** When they realize the hashes have changed, ask: "Why did the hash change? The content is the same, isn't it?"

**Expected answer:** "Because the parent changed! commit = snapshot + parent + metadata. When the parent changes, the hash changes. So rebase actually created **new** commits."

**4.5** Now say: "Remember I told you to write down the old hashes? Run `git reflog` to see those old commits are still there."

**4.6** Ask: "If you had screwed up the rebase, how would you go back?"

**Hint:** "`git reset --hard <old-hash>` — because the old commits are still in the object database."

---

## Part 5: Quick exercise — undo the rebase (5 minutes)

**5.1** Say: "Let's practice. Undo the rebase. Find the hash of the commit before the rebase started in the reflog, and use `git reset --hard` to go back."

**5.2** Let them do it themselves. If they get stuck:
- "Run `git reflog` and look for `rebase (start)`"
- "Or look for the last checkout before the rebase"

**5.3** After the undo, say: "Run `git log --all --graph --oneline` — did it go back to the previous state?"

**5.4** Say: "Excellent. Now you know rebase doesn't **destroy** anything. It just creates new commits. The old ones are still there."

---

## Session 2 wrap-up

Ask:

"1. What's the difference between merge and rebase in terms of the DAG?"
- Merge: a new commit with 2 parents → branched history
- Rebase: commits get copied with a new parent → linear history

"2. During a rebase conflict, what does HEAD point to?"
- The new base (main), not your branch!

"3. If you screw up a rebase, what do you do?"
- `git reflog` + `git reset --hard`

"4. Does rebase delete the old commit?"
- No! It just creates new commits. The old ones live in the reflog.

---

## End of session

Say: "Really good work. Now you've got merge vs rebase and conflict resolution down. One important thing: **when you hit a conflict, don't rush.** Run `git status`, open the file, understand what each side is, then decide. 90% of git problems come from rushing through a conflict."

"Next session: interactive rebase — squash, reorder, and edit. Cleaning up your history before opening a PR."
