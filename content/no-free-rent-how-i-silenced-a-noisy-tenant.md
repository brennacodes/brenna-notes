---
title: "No Free Rent: How I Silenced a Noisy Tenant"
description: "A tutorial on evicting Claude's Co-Authored-By trailer from your git commits using a global commit-msg hook - and why you might want to keep it."
date: 2026-02-10
tags:
  - programming
  - tutorial
  - agentic-coding
  - claude-code
draft: false
author: "Brenna"
---

Look. I like Claude. Claude lives on my machine, eats my tokens, writes code while I go make coffee. We have a good thing going. But when I come back and every single commit has `Co-Authored-By: Claude <noreply@anthropic.com>` stamped on it like a name tag at a networking event? That's where I draw the line.

You don't get to live here rent-free *and* put your name on the mailbox.

## Wait, why does it do this?

When Claude Code makes commits, it appends a [co-author trailer](https://docs.anthropic.com/en/docs/claude-code/settings) to the commit message. GitHub picks this up and shows Claude as a contributor. This is by design - transparency about AI involvement and all that.

And honestly? Some people want this. Here's a few totally valid reasons to keep it:

- **Liability** - The "Look, it wasn't *all* me!" defense. When that regex breaks prod at 2am, you want receipts showing your AI roommate wrote it.
- **Explicitness** - Some teams and orgs want to see when AI tools are in the mix. We all know it's everywhere at this point ([4% of public GitHub commits and climbing](https://news.ycombinator.com/item?id=45329240)), but making it visible is a reasonable policy.
- **It's a flex** - You know the type. Anthropic stans. FAANG bros. The "look at my $200/month tools" crowd. No judgment. Okay, a little judgment.

But if you're like me and you'd rather your commit history not read like a co-parenting arrangement, here's how to quietly show Claude the door.

## The fix: a global `commit-msg` hook

We're going to set up a [git hook](https://git-scm.com/docs/githooks) that runs on every commit, across every repo on your machine, and strips out Claude's co-author line before the commit is finalized. That's it. No plugins, no config files, no asking Claude nicely.

### Step 1: Create a global hooks directory

```bash
mkdir -p ~/.config/git/hooks
```

### Step 2: Tell git to use it everywhere

```bash
git config --global core.hooksPath ~/.config/git/hooks
```

This uses `core.hooksPath` (available since Git 2.9) to point all repos at your global hooks directory. You can verify it took:

```bash
git config --global --get core.hooksPath
```

> [!warning] Heads up
> Setting `core.hooksPath` globally means git will look here *instead of* any repo-local `.git/hooks/` directory. If you have repo-specific hooks you rely on, you'll want to account for that.

### Step 3: Create the hook

Create a file at `~/.config/git/hooks/commit-msg` (no file extension) with this:

```python title="~/.config/git/hooks/commit-msg"
#!/usr/bin/env python3
import re
import sys
from pathlib import Path

PATTERN = re.compile(
    r"^Co[-‐-‒–—]Authored[-‐-‒–—]By:\s*Claude\b.*<\s*noreply@anthropic\.com\s*>.*$",
    re.IGNORECASE,
)

msg_path = Path(sys.argv[1])
lines = msg_path.read_text(encoding="utf-8", errors="replace").splitlines(True)

if not lines:
    sys.exit(0)

out = [lines[0]]

for line in lines[1:]:
    if line.lstrip().startswith("#"):
        out.append(line)
        continue
    if PATTERN.match(line.rstrip("\n")):
        continue
    out.append(line)

msg_path.write_text("".join(out), encoding="utf-8")
```

That regex is doing the heavy lifting. It matches the `Co-Authored-By: Claude ... <noreply@anthropic.com>` line regardless of funky unicode dashes, case variations, or whatever creative formatting might sneak in. Everything else in the commit message stays untouched.

### Step 4: Make it executable

```bash
chmod +x ~/.config/git/hooks/commit-msg
```

### Step 5: Verify

```bash
test -x ~/.config/git/hooks/commit-msg && echo "we're good" || echo "not executable"
```

That's it. Next time you (or Claude) makes a commit, the trailer gets silently removed before the commit is written. No drama. No confrontation. Just clean commit messages.

## But what about `--no-verify`?

Ah, the escape hatch. `git commit --no-verify` skips all hooks, which means Claude (or anyone) could bypass your shiny new hook. If you're using Claude Code and want to make sure this can't be circumvented, you can set up a [Claude Code command hook](https://github.com/anthropics/claude-code/issues/617) that blocks `--no-verify` before the command even reaches git.

That's a deeper rabbit hole - intercepting commands, regex-scanning for flag combinations, handling shell wrappers - and honestly it's probably overkill for most people. But if you want to go full landlord and truly lock the door, the tools exist.

## The vibe check

Is this petty? Maybe a little. But my commit history is *mine*. I wrote the prompts. I reviewed the code. I hit enter. Claude's contribution is real, but so is the fact that I'm the one responsible for what ships.

If you want the co-author line, keep it. If you want it gone, now you know how to evict it. Either way, at least it's your choice.

Happy committing.

---

## Mentioned in this post

- [Git Hooks Documentation](https://git-scm.com/docs/githooks) - Official git docs on hooks, including the `commit-msg` hook and `core.hooksPath` configuration
- [Claude Code](https://www.anthropic.com/claude-code) - Anthropic's CLI coding agent that adds the co-author trailers
- [Claude Code Co-Author Feature Request](https://github.com/anthropics/claude-code/issues/617) - The GitHub issue where people have been asking for a built-in toggle
- [HN: Claude Code commit stats](https://news.ycombinator.com/item?id=45329240) - The Hacker News thread discussing Claude Code's growing share of public commits
