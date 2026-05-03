# Git Session 4: Crisis Scenarios — "Everything's broken, now what?"

## Instructions for Claude Code

Session 4 of the git tutorial. The user now knows:
- Git internals: objects, refs, HEAD, DAG
- Merge vs rebase (and conflict resolution)
- Interactive rebase: squash, fixup, reword, edit, drop, reorder
- They know rebase doesn't delete commits and that reflog exists (but haven't worked deeply with it yet)

**This session's goal is to make the user not afraid of git.** After this session they should feel "nothing in git is really lost — there's always a way back."

**Language: conversational English**

### Interaction rules:
1. **Stage every scenario like an emergency** — first build the broken state, then say "okay, what do you do now?"
2. **Let them try first** — wait at least 2 minutes before hinting.
3. **After each scenario, give a rule of thumb** they'll remember in real work.
4. **Tone: like a senior coworker calmly helping** — not panicked, not lecturing.

---

## Step 0: Set up the repo

Build a clean repo with a rich history:

```bash
cd ~/git-playground

# Reset completely
git checkout main
for branch in $(git branch | grep -v main); do git branch -D $branch; done

# Build out main (if it's thin)
cat > main.go << 'EOF'
package main

import "fmt"

func main() {
    fmt.Println("v1.0 - production ready")
}
EOF
git add -A
git commit -m "release: v1.0" --allow-empty-message 2>/dev/null || git commit -m "release: v1.0"

cat > server.go << 'EOF'
package main

import "net/http"

func startServer() {
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("OK"))
    })
    http.HandleFunc("/api/users", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(`[{"id":1,"name":"Ali"}]`))
    })
}
EOF
git add server.go
git commit -m "feat: add HTTP server with health and users endpoints"

cat > middleware.go << 'EOF'
package main

import (
    "log"
    "net/http"
    "time"
)

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}
EOF
git add middleware.go
git commit -m "feat: add logging middleware"

cat > auth.go << 'EOF'
package main

func checkAuth(token string) bool {
    return token == "secret-token-123"
}
EOF
git add auth.go
git commit -m "feat: add auth check"

# Feature branch for the scenarios
git checkout -b feature/dashboard

cat > dashboard.go << 'EOF'
package main

func getDashboardData() map[string]int {
    return map[string]int{
        "users":    150,
        "orders":   42,
        "revenue":  9800,
    }
}
EOF
git add dashboard.go
git commit -m "feat: add dashboard data endpoint"

cat > dashboard_test.go << 'EOF'
package main

import "testing"

func TestGetDashboardData(t *testing.T) {
    data := getDashboardData()
    if data["users"] != 150 {
        t.Errorf("expected 150 users, got %d", data["users"])
    }
}
EOF
git add dashboard_test.go
git commit -m "test: add dashboard tests"

echo '// analytics integration' >> dashboard.go
git add dashboard.go
git commit -m "feat: integrate analytics into dashboard"

git checkout main
```

Say: "The repo is ready. Today we have 4 crisis scenarios. Each one is something you might run into in real work. Ready?"

---

## Scenario 1: Accidental force push — "Your coworker called and said their commits are gone!" (15 minutes)

### Setup:
```bash
git checkout feature/dashboard

# Save the hash before the disaster
BEFORE_DISASTER=$(git rev-parse HEAD)
echo "📌 Hash before disaster: $BEFORE_DISASTER"

# Simulation: you accidentally reset and force pushed
git reset --hard HEAD~2
# (In real life this would be `git push --force` next, but since we don't have a remote, we're simulating)
```

Say: "Situation: you were cleaning up the branch, accidentally ran `git reset --hard HEAD~2`, then ran `git push --force`. Now your last 2 commits (the tests and analytics) are gone. Your coworker, who pulled from this branch earlier, just called saying their commits are missing.

Run `git log --oneline` and see what you have right now."

### Solution:

**Step 1:** Ask: "From the previous sessions, what tool do you have for finding 'lost' commits?"

If they say reflog: "Nice. Run `git reflog` and check."

If they don't: "A hint: where does every move of HEAD get recorded?"

**Step 2:** Say: "In the reflog, find the last commit before the reset. Get the hash."

**Step 3:** Ask: "Now how do you bring it back?"

Answer: `git reset --hard <hash>` or `git reset --hard HEAD@{n}`

**Step 4:** After recovery: "Run `git log --oneline` — are they back?"

**Step 5:** Say: "Now an important question: if you really had force pushed and your coworker lost their commits, what should they do?"

Answer: "Your coworker checks their **own** `git reflog` — since they pulled earlier, those commits are in their reflog too."

### Rule of thumb:
"**Before any force push, drop a tag:** `git tag backup-before-cleanup` — if it goes wrong, recovery is one command."

---

## Scenario 2: Detached HEAD — "Where am I? Where did my commit go?" (10 minutes)

### Setup:
```bash
git checkout main

# Simulation: you checked out a commit directly (not a branch)
SOME_COMMIT=$(git log --oneline -3 | tail -1 | awk '{print $1}')
git checkout $SOME_COMMIT
```

Say: "Situation: you were inspecting an old commit and ran `git checkout <hash>`. Now git has given you a big warning. Run `git status` and see what it says."

### Solution:

**Step 1:** Ask: "What does detached HEAD mean? What is HEAD pointing to right now?"

If they don't know: "Run `cat .git/HEAD` — see how it differs from the normal case."

Note: "Normal: `ref: refs/heads/main`. Now: it points directly at a hash. Meaning you're not on any branch."

**Step 2:** Say: "Now, suppose without realizing it, you make a commit here."

```bash
echo "// experimental stuff" > experiment.go
git add experiment.go
git commit -m "experiment: try new approach"
```

**Step 3:** Say: "Now run `git checkout main` — git gives you a warning. Read it."

**Step 4:** Ask: "Where is that experiment commit right now? Does any branch point to it?"

Answer: "No! It's an orphan commit — no ref points to it."

**Step 5:** Say: "But it still exists. How do you find it and rescue it?"

Answer 1: `git reflog` → find the hash → `git branch rescue <hash>`
Answer 2: `git reflog` → `git checkout -b rescue <hash>`

Let them do it themselves.

**Step 6:** After the rescue, say: "Run `git log --all --graph --oneline` — see where the rescue branch is now."

### Rule of thumb:
"**Whenever you want to look at an old commit, create a branch first:** `git checkout -b explore/<topic> <hash>` — that way you never end up detached."

---

## Scenario 3: You committed on main by accident (you were supposed to branch) (10 minutes)

### Setup:
```bash
git checkout main

# Simulation: you made 2 commits directly on main that shouldn't be there
cat > hotfix.go << 'EOF'
package main

func applyHotfix() string {
    return "patched critical bug #1234"
}
EOF
git add hotfix.go
git commit -m "fix: critical bug #1234"

echo '// additional safety check' >> hotfix.go
git add hotfix.go
git commit -m "fix: add safety check for #1234"
```

Say: "Situation: you made two commits and just realized you were on main! You should have branched. Now main has two extra commits it shouldn't have.

Run `git log --oneline -5` and see the situation."

### Solution:

**Step 1:** Ask: "You want to move these 2 commits to their own branch and revert main. How?"

Let them think.

**Step 2:** If they get stuck, hint:
"A hint: a branch is just a pointer. Right now main points at the latest commit. If you create a new branch **right now**, it'll also point to that same commit..."

**Step 3:** Step-by-step answer:
```bash
# 1. First, create the new branch (right here — including the extra commits)
git branch hotfix/1234

# 2. Now move main back 2 commits
git reset --hard HEAD~2

# 3. Verify
git log --oneline -3        # main should be without the hotfix commits
git log hotfix/1234 --oneline -3  # hotfix should have the commits
```

**Step 4:** Ask: "Why did this work? Why are the commits still on the hotfix branch when we reset main?"

Answer: "Because `git branch hotfix/1234` created a new pointer to that same commit. When we moved main back, the commits were still reachable through hotfix."

### Rule of thumb:
"**Before any commit, run `git status` and check which branch you're on.** Or set your terminal prompt to show the branch."

---

## Scenario 4: Cherry-pick — "I just want this one commit" (10 minutes)

### Setup:
```bash
git checkout main

# Hypothetical: from feature/dashboard, we only want one specific commit
echo ""
echo "feature/dashboard state:"
git log feature/dashboard --oneline -5
```

Say: "Situation: feature/dashboard isn't ready for a PR yet, but the test commit (`test: add dashboard tests`) is needed on main right now because CI is failing. You don't want to merge the whole branch — just that one commit.

Run `git log feature/dashboard --oneline` and find the hash of the test commit."

### Solution:

**Step 1:** Say: "Now run:
`git cherry-pick <hash-of-test-commit>`"

**Step 2:** Ask: "Run `git log --oneline -3` — what do you see? Is the hash of that commit different from the original?"

Answer: "Yes! Cherry-pick, like rebase, creates a **new** commit (because the parent is different)."

**Step 3:** If a conflict comes up: "Okay, you know what to do — `git status`, resolve the conflict, `git add`, `git cherry-pick --continue`."

**Step 4:** Ask: "Later when you merge feature/dashboard into main, will that test commit show up twice?"

Answer: "No! Git can tell from the content (tree) that the changes are identical and handles it."

### Rule of thumb:
"**Cherry-pick is good for 1-2 commits.** If you want more than 3, you probably want merge or rebase instead."

---

## Session 4 wrap-up

Say: "Okay, let's see what's in your toolbox now:

| Crisis | Tool |
|--------|------|
| Lost commit | `git reflog` + `git reset --hard` or `git branch rescue <hash>` |
| Accidental force push | `git reflog` (yours and your coworker's) |
| Detached HEAD | `git checkout -b <branch>` or `git switch -` |
| Commit on the wrong branch | `git branch <new>` + `git reset --hard HEAD~n` |
| Need a specific commit | `git cherry-pick <hash>` |

A general rule: **almost nothing in git is truly gone** (until garbage collection runs — which is usually 30 days out). So never panic. Run `git reflog`."

---

## End of session

Say: "Four sessions, done. You have a solid mental model of git now + you can handle the worst scenarios.

Want to keep going? The next session could be one of these:
1. **Professional git workflow** — branching strategy, PR etiquette, commit message conventions
2. **Git hooks and automation** — pre-commit, pre-push, CI integration
3. **Advanced scenarios** — bisect (finding bugs), worktree, submodule

Which one would help you most?"
