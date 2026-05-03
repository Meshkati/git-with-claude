# Git Session 3: Interactive Rebase — "Clean up your history before the PR"

## Instructions for Claude Code

Session 3 of the git tutorial. The user now knows:
- Commit = snapshot + parent + metadata (not a diff!)
- Branch = pointer to a commit
- Merge = a new commit with 2 parents
- Rebase = copying commits with a new parent (hashes change)
- Conflict resolution and undoing a rebase via reflog

**Language: conversational English**

### Interaction rules:
1. **Let them type the commands and think for themselves.**
2. **Ask a question before each new section** so the previous one is locked in.
3. **When the editor opens (during interactive rebase), explain what they're about to see beforehand** so they aren't startled.
4. **Translate every operation back into the language of objects:** "squash means combining two snapshots into one new commit."

---

## Step 0: Build a realistic scenario

Build a scenario that looks like a real, messy feature branch. **You run the commands** and tell the user what you're doing:

```bash
cd ~/git-playground

# Clean state
git checkout main
git branch -D feature/user-profile 2>/dev/null

# Build a feature branch with a real, messy history
git checkout -b feature/user-profile

# Commit 1: starting work
cat > profile.go << 'EOF'
package profile

type User struct {
    Name  string
    Email string
}

func GetProfile(id int) User {
    return User{Name: "test", Email: "test@test.com"}
}
EOF
git add profile.go
git commit -m "wip: start profile"

# Commit 2: a typo fix
sed -i 's/test@test.com/test@example.com/' profile.go
git add profile.go
git commit -m "fix typo in email"

# Commit 3: adding validation
cat > validate.go << 'EOF'
package profile

func ValidateEmail(email string) bool {
    return len(email) > 0
}
EOF
git add validate.go
git commit -m "add email validation"

# Commit 4: oh, forgot to add Age to the struct
sed -i 's/Email string/Email string\n    Age   int/' profile.go
git add profile.go
git commit -m "forgot to add Age field"

# Commit 5: adding tests
cat > profile_test.go << 'EOF'
package profile

import "testing"

func TestGetProfile(t *testing.T) {
    u := GetProfile(1)
    if u.Name == "" {
        t.Error("expected name")
    }
}

func TestValidateEmail(t *testing.T) {
    if !ValidateEmail("a@b.com") {
        t.Error("should be valid")
    }
    if ValidateEmail("") {
        t.Error("should be invalid")
    }
}
EOF
git add profile_test.go
git commit -m "add tests (finally lol)"

# Commit 6: a debug print that shouldn't stay
sed -i '/return User/i\    fmt.Println("DEBUG: getting profile", id)' profile.go
# Add fmt import
sed -i 's/package profile/package profile\n\nimport "fmt"/' profile.go
git add profile.go
git commit -m "debug: add print for testing"

# Commit 7: removing the debug
sed -i '/fmt.Println/d' profile.go
sed -i '/import "fmt"/d' profile.go
sed -i '/^$/N;/^\n$/d' profile.go
git add profile.go
git commit -m "remove debug print"
```

Then say:

"The repo is ready. You have a feature branch (`feature/user-profile`) with 7 commits. Run `git log --oneline` to see what you have.

This is the history of a real developer: wip, typo fix, forgot something, debug print, remove debug... if you opened a PR with this as-is, your reviewer would lose it.

The goal is to turn these 7 commits into 3-4 clean, meaningful commits."

---

## Part 1: Getting familiar with interactive rebase (10 minutes)

### Goal:
The user should understand what `rebase -i` shows them and what each keyword means.

**1.1** Say: "First, let's see how many commits you have on this branch that main doesn't:
`git log main..feature/user-profile --oneline`"

**1.2** Say: "Now we'll start the interactive rebase. But **first**, let me tell you what you're about to see. A list of commits will appear (from **oldest to newest** — the opposite of git log!) and each one will have a keyword in front of it. They'll all be `pick` to start.

Run: `git rebase -i main`"

**Note: if nano or vim opens, suggest using `GIT_SEQUENCE_EDITOR` to make it easier. Or, if you can directly edit the file yourself, do that. The point is for the user to understand the contents of the file, not to wrestle with vim.**

**1.3** When they see the list, ask: "What do you see? What does each line have?"

**1.4** Then explain the main commands:
- `pick` = keep this commit as-is
- `squash` (or `s`) = merge this commit into the previous one (shows both commit messages so you can edit them)
- `fixup` (or `f`) = like squash, but throw away this commit's message
- `reword` (or `r`) = keep the commit, but change its message
- `edit` (or `e`) = stop on this commit so I can change it
- `drop` (or `d`) = delete this commit

**1.5** Ask: "Looking at our history, which commits do you think should be squashed? Which should be dropped?"

**Abort for now:** Say "For now, run `git rebase --abort`. Let's plan first, then execute."

---

## Part 2: Plan it out (5 minutes)

### Goal:
Before running anything, the user decides what structure they want.

**2.1** Say: "Let's list the commits and decide:

1. `wip: start profile` → pick (the first commit)
2. `fix typo in email` → ❓
3. `add email validation` → ❓
4. `forgot to add Age field` → ❓
5. `add tests` → ❓
6. `debug: add print` → ❓
7. `remove debug print` → ❓

Think about it: if you wanted 3 clean commits, what would the ideal structure be?"

**2.2** Let them think. If they get close to the answer, confirm. If they get stuck, suggest:

Suggested structure:
- Commit A: `feat: add user profile with Age field` → pick commit 1 + fixup 2 + fixup 4 + reword
- Commit B: `feat: add email validation` → pick commit 3
- Commit C: `test: add profile and validation tests` → pick commit 5
- Drop: commits 6 and 7 (debug and remove-debug — they cancel each other out, no net change)

---

## Part 3: Run the interactive rebase (15 minutes)

### Goal:
The user should run `rebase -i` themselves.

**3.1** Say: "Now run `git rebase -i main` — but this time we're really editing it."

**3.2** Edit the todo file together. The order and commands should look like this:

```
pick abc1234 wip: start profile
fixup def5678 fix typo in email
fixup ghi9012 forgot to add Age field
pick jkl3456 add email validation
pick mno7890 add tests (finally lol)
drop pqr1234 debug: add print for testing
drop stu5678 remove debug print
```

**Important note:** Tell the user: "Look — we **moved** commit 4 (forgot to add Age) up under commit 1. By reordering, we're telling git to copy the commits in this new order."

**3.3** When they save and the rebase starts:

- If a conflict comes up (likely between commit 1 and the fixup of commit 4 because of the reorder), say: "Okay, conflict! But you know what to do now — run `git status`."
  - Help them resolve the conflict
  - Then: `git add .` and `git rebase --continue`

- If it goes through with no conflict, great.

**3.4** After it finishes: "Run `git log --oneline` to see the result."

**3.5** Ask: "How many commits do you have? Are the messages meaningful?"

**3.6** If the first commit's message is still "wip: start profile", say: "The first message isn't great. Run `git rebase -i main` and `reword` the first commit to something better, like `feat: add user profile module with Age field`."

---

## Part 4: Inspect the result (5 minutes)

**4.1** Say: "Run `git log --oneline --stat` to see which files each commit changes."

**4.2** Ask: "If a reviewer saw this PR now, could they review each commit independently?"

**4.3** Say: "Run `git diff main..feature/user-profile` — see whether the **final result** changed."

**Key takeaway:** "The changes are exactly the same! Interactive rebase only changed **the history**, not the final result."

---

## Part 5: edit mode — when you want to break a commit apart (10 minutes — optional)

### Goal:
Show how `edit` works — for when a commit is too big and you want to split it.

**Only run this section if there's time and they aren't worn out. Otherwise say "we'll learn this in the next session."**

**5.1** Say: "Another scenario: imagine the validation commit has both the test and the code, and you want to split it."

**5.2** Say: "Run `git rebase -i main` and `edit` the validation commit."

**5.3** When the rebase pauses:
"Right now git is paused on that commit. Run `git status` — you'll see HEAD is on that commit.
Now you can:
- Run `git reset HEAD~1` (soft undo — the changes go back into the working dir)
- Then split the changes into 2 commits
- Then run `git rebase --continue`"

**5.4** Let them do it.

---

## Session 3 wrap-up

Ask:

"1. What's the difference between `squash` and `fixup`?"
- squash: keeps both messages so you can edit them
- fixup: throws away the second commit's message

"2. When you reorder commits in `rebase -i`, what does git actually do?"
- It **copies** the commits in the new order (with a new parent) — same as a regular rebase

"3. Does interactive rebase change the final state of the code?"
- No! Only the history changes. The final diff is the same.

"4. If you screw up in the middle of an interactive rebase, what do you do?"
- `git rebase --abort` → goes back to before you started
- Or `git reflog` + `git reset --hard`

---

## End of session

Say: "Great work. Now you have a complete workflow:

1. Work on a feature branch (commit as messy as you want)
2. Before the PR: `git rebase -i main` → clean it up
3. Push

Next session: crisis scenarios — accidental force push, detached HEAD, and recovery with reflog."
