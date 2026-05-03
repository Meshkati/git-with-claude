# git-with-claude

Learn Git step by step — not by memorizing commands, but by building the mental model behind them. An AI mentor (Claude) walks you through it Socratically, one question at a time.

## The approach

Inspired by Karpathy-style "build it from scratch to understand it" teaching, plus the Socratic method.

- **Step by step.** No leaps. A session won't move forward until the current concept actually clicks.
- **You drive the keyboard.** The mentor asks; you type, read the output, and explain what you think happened.
- **Questions before answers.** Instead of "here's how rebase works," you'll get "what do you think just happened?" Wrong answers are how the gaps get found.

## What you'll actually learn

Most Git tutorials teach you `git rebase -i`. This one teaches you what a commit *is*.

- **Did you know** every commit is a full snapshot of your project, not a diff? You'll prove it yourself with `git cat-file`.
- **Did you know** a branch is literally a 40-character file inside `.git/refs/heads/`? You'll `cat` it.
- **Did you know** "deleted" commits usually aren't deleted at all? You'll find them with `reflog` and bring them back.

Once these click, rebase, conflicts, detached HEAD, and "oh no I lost my work" stop being scary.

## The four sessions

1. **Mental model** — objects, refs, HEAD, the three states (working dir / staging / committed)
2. **Rebase & conflict resolution** — copy commits vs. move them, why conflicts happen, how to resolve them calmly
3. **Interactive rebase** — squash, fixup, reword, edit, drop, reorder
4. **Crisis scenarios** — you ran `reset --hard`, you force-pushed the wrong branch, you're in detached HEAD. Get out alive.

## A taste — 3 commands that change how you see Git

Run these in any repo. Each one peels back a layer.

**1. See your history as a graph and grab a commit's hash.**

```bash
$ git log --oneline --graph --all
*   a3f9c21 (HEAD -> main) merge: bring in login flow
|\
| * 8e4b1c0 (feature/auth) feat: add logout
| * 5c9d2a7 feat: add login
* | 7c2e11d feat: add helper.go
|/
* 6f0a3e9 feat: add main.go
* 4d1f8b2 initial: add README
```

**2. Look inside that commit. It's not a diff — it's an object that points to a snapshot (tree) and its parents. A merge commit has two.**

```bash
$ git cat-file -p a3f9c21
tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904
parent 7c2e11d8e4f3a9b1c5d6e7f8a9b0c1d2e3f4a5b6
parent 8e4b1c0f9a2d3b5c7e1f4a6d8b9c2e0f1a3d5b7c
author You <you@example.com> 1709123456 +0330
committer You <you@example.com> 1709123456 +0330

merge: bring in login flow
```

**3. Look inside the tree. It's just a list of files (blobs) with their hashes.**

```bash
$ git cat-file -p 4b825dc642cb6eb9a060e54bf8d69288fbee4904
100644 blob 8d0e41234f4b... README.md
100644 blob 1a2b3c4d5e6f... main.go
100644 blob 9f8e7d6c5b4a... auth.go
```

That's the whole model. Commit → tree → blobs. Branches are pointers to commits. HEAD is a pointer to a branch. Everything else is built on top.

## How to use this

1. Open Claude (Claude Code, Claude Desktop, or claude.ai — anything that can read a file).
2. Tell it: *"Read `git-session-1-mental-model.md` and start the session with me."*
3. The file contains instructions for the AI — it will play mentor, ask you questions, and let you drive the keyboard. Don't peek at the file yourself; the surprise is part of it.
4. When you finish a session, move to the next one.

You'll need a terminal with `git` installed. The first session creates a small playground repo for you to break and fix.

## Contributing

Found a confusing step? An exercise that didn't land? Open an issue or PR. The goal is for someone to walk away thinking *"oh, that's all Git is?"* — anything that gets in the way of that moment is worth fixing.
