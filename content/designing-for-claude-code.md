---
title: "Designing for Claude Code"
description: "A practical framework for deciding when and how to use skills, hooks, subagents, sandboxing, memory, and composable workflows in Claude Code - grounded in community-tested patterns and hard-won lessons from building plugins."
date: 2026-02-25
tags:
  - claude-code
  - agentic-coding
  - programming
  - tutorial
  - claude-plugins
  - claude-marketplaces
  - best-practices
  - guide
draft: false
author: "Brenna"
---

I've been building [Claude Code plugins](https://github.com/brennacodes/brenna-plugs) for a few months now - a whole marketplace of them, actually - and I recently started building plugins at work too. Somewhere between my fifth skill rewrite and my third "why did Claude forget that rule" incident, I realized I'd accidentally assembled a mental framework for *when to use what*.

This post is that framework, written down.

Things are changing fast in this space. Like, *really* fast. I'll do my best to keep this updated as I learn more, and any time there are changes that impact what's here. But fair warning: if you're reading this six months from now, some of this might already be outdated. That's the deal with building on top of something that ships new features every week.

Let's get into it.

## Core Philosophy

Three principles guide every decision I make. These aren't theoretical - they've been validated repeatedly by people building real workflows, including me.

### 1. Determinism over suggestion

> If something must happen every time, use a hook. If it should happen when contextually relevant, use a skill. If it requires judgment in an isolated context, use a subagent.

Multiple practitioners have independently discovered that CLAUDE.md rules and prompt-based instructions get deprioritized in long sessions. [Lakshmi Narasimhan's guide](https://medium.com/@lakshminp/claude-code-hooks-the-feature-youre-ignoring-while-babysitting-your-ai-789d39b46f6c) puts it bluntly: "An instruction in your CLAUDE.md file is a suggestion. The AI will probably follow it, but in a long conversation, it might get pushed out of the context window or just de-prioritized. A hook is a hard-coded rule. It's a command that is guaranteed to run when its specific event happens."

I've felt this one personally. I had rules in my CLAUDE.md that Claude followed perfectly for the first 20 minutes of a session, and then just... stopped. Not maliciously. It just got crowded out by more immediate context. That's when hooks clicked for me.

### 2. Progressive disclosure over upfront loading

> Only load what's needed, when it's needed. Metadata is cheap; full context is expensive.

At startup, only the `name` and `description` from all Skills' YAML frontmatter are loaded into the system prompt. The full SKILL.md is read only when the skill becomes relevant. Supporting files (reference docs, templates, scripts) are read only when Claude accesses them. Scripts are executed via bash and only their *output* consumes tokens - the script itself never enters the context window.

This is by design: the context window is a shared resource, and every token your skill loads competes with conversation history and everything else Claude needs to reason.

### 3. Simplicity compounds; complexity decays

> Every layer of abstraction you add is a layer Claude can misinterpret. Start simple, add complexity only when you've hit a wall.

The most-cited best practices compilation - [shanraisshan's claude-code-best-practice repo](https://github.com/shanraisshan/claude-code-best-practice) - states it definitively: "Simple control loops outperform multi-agent systems. Low-level tools (Bash, Read, Edit) plus selective high-level abstractions beat heavy RAG or complex frameworks."

The most successful Claude Code users I've seen keep systems simple and iterate continuously, refining CLAUDE.md, skills, and workflows based on what Claude gets wrong. I'm one of them. My plugins have gone through four major versions, and every version got *simpler*, not more complex.

---

## The Decision Matrix

Use this as your primary routing logic. For any given behavior you want Claude to exhibit, ask these questions in order.

### 1. Does this need to happen every single time, no exceptions?

**Use a Hook.**

[Hooks](https://code.claude.com/docs/en/hooks-guide) are deterministic. They execute outside the agentic loop, which means Claude can't forget, deprioritize, or reinterpret them. They're shell commands triggered by lifecycle events, and they add zero token overhead because they run outside the context window.

The title of Lakshmi Narasimhan's [widely-shared article](https://medium.com/@lakshminp/claude-code-hooks-the-feature-youre-ignoring-while-babysitting-your-ai-789d39b46f6c) captures the consensus: "Claude Code Hooks: The Feature You're Ignoring While Babysitting Your AI." His conclusion: hooks transform Claude from "an AI assistant you have to supervise" into "an AI assistant that follows your rules automatically."

Good candidates:
- Auto-formatting after file edits (PostToolUse)
- Blocking dangerous commands before execution (PreToolUse)
- Running tests before Claude is allowed to "finish" (Stop)
- Sending notifications when Claude needs input (Notification)
- Re-injecting critical rules after compaction (PreToolUse on UserPromptSubmit)
- Validating output against a schema before accepting it

Bad candidates:
- Anything requiring judgment or context-awareness (use a skill)
- Anything that varies based on what Claude is working on

#### Hook types by complexity

**Command hooks** (90% of cases): Shell scripts. Fast, predictable, no token cost. This is your workhorse.

**Prompt hooks**: Send a question to a lightweight model for a yes/no decision. No tool access. Use when the decision is simple but not purely mechanical.

**Agent hooks**: Spawn a subagent that can read files, run commands, and reason. Use when verification requires understanding the codebase state. One developer describes combining a Stop hook with an agent hook that verifies whether all requested tasks were completed and tests were run - creating "an AI agent that verifies its own work before stopping." His conclusion: "the more you automate the verification of AI-generated code, the better the output quality."

#### The compaction survival pattern

This one deserves special attention because it solves a problem that will bite you eventually. Long sessions trigger context compaction, and Claude can "forget" important rules. The fix:

```json title="settings.json"
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "UserPromptSubmit",
        "hooks": [
          {
            "type": "command",
            "command": "cat .claude/rules.txt"
          }
        ]
      }
    ]
  }
}
```

This re-injects your core rules into the context window on every prompt. Claude can't "forget" what's staring it in the face. Multiple community members independently converged on this pattern as a solution to context rot.

### 2. Is this reusable knowledge, a workflow, or a procedure Claude should follow?

**Use a [Skill](https://code.claude.com/docs/en/skills).**

Skills are contextual instructions that Claude loads on-demand. They're the right tool for encoding "how to do X well" - domain knowledge, step-by-step procedures, templates, coding patterns, and reference material.

[Shrivu Shankar](https://blog.sshh.io/p/how-i-use-every-claude-code-feature), in a widely-shared post, argues that skills formalize a "scripting"-based agent model that is more robust and flexible than the rigid, API-like model that MCP represents. His framing: instead of hand-crafting tools and abstracting away reality for the agent, you give the agent access to the raw environment - binaries, scripts, docs - and it writes code on the fly to interact with them. Skills are the productized version of this pattern. [Simon Willison](https://simonwillison.net/2025/Oct/16/claude-skills/) echoed this, calling skills "maybe a bigger deal than MCP."

Good candidates:
- Multi-step workflows (e.g., "how to create a PR in this repo")
- Domain-specific knowledge injection (e.g., "our API conventions")
- Template-driven output (e.g., "generate a component using this structure")
- Procedures with decision points Claude needs to reason through
- Anything invoked by the user via `/skill-name`

#### Skill design principles

**Keep SKILL.md under 500 lines.** Move reference material, templates, examples, and data into supporting files in the skill directory. Once Claude loads SKILL.md, every token competes with conversation history. Supporting files have zero context cost until read.

**Use XML for structured workflows.** When the skill defines a predictable sequence of steps, wrap them in XML tags. Anthropic's own prompting docs confirm that XML tags help Claude parse prompts more accurately, leading to higher-quality outputs. XML is especially valuable for branching workflows where Claude needs to follow different paths based on conditions - the tags make the boundaries explicit and prevent Claude from improvising when it shouldn't.

```xml
<workflow>
  <step name="validate">
    Run the validation script in scripts/validate.sh.
    If validation fails, stop and report the errors.
  </step>
  <step name="generate">
    Read templates/component.tsx and generate the output
    following the template structure exactly.
  </step>
  <step name="verify">
    Run the quality gates defined in ../_shared/steps/run-quality-gates.md.
  </step>
</workflow>
```

**One skill, one job.** Don't create "do everything" skills. Create focused skills that can be composed. I learned this the hard way - my first plugin had one massive skill that tried to handle six different workflows. Version 2 had six focused skills. Guess which one actually worked.

**Attach scripts instead of documenting procedures.** Instead of writing "run eslint with these flags and then run prettier with these flags" in your SKILL.md, put that logic in a script within the skill directory and tell Claude to execute it. The script runs via bash and only its output enters the context. The script logic itself never consumes tokens.

**Put output templates in separate files.** If the skill produces structured output (a PR body, a changelog entry, a component scaffold), put the template in a file like `templates/output.md`. This keeps instructions and output format cleanly separated, makes templates easier to maintain, and avoids bloating SKILL.md.

#### Writing effective skill descriptions

The description field is critical - it's how Claude decides whether to load your skill. Community testing has revealed specific patterns that dramatically improve auto-invocation accuracy.

**Use a WHEN + WHEN NOT pattern.** Generic descriptions fail. Specific trigger conditions succeed. John Conneely's testing found that structuring descriptions with explicit invocation and exclusion conditions made skills invoke reliably every time:

```yaml
description: >
  Stakeholder context for Test Project when discussing product features,
  UX research, or stakeholder interviews. Auto-invoke when user mentions
  Test Project, product lead, or UX research. Do NOT load for general
  stakeholder discussions unrelated to Test Project.
```

**Use possessive pronouns for personal skills.** Conneely discovered that using "HIS/HER/THEIR" in descriptions scoped personal skills perfectly:

```yaml
description: >
  Personal work preferences and communication style for [Name].
  Auto-invoke when drafting HIS emails, Slack messages, or internal updates;
  planning HIS work or tasks; optimising HIS productivity workflows.
  Do NOT load for external blog posts or customer-facing communications.
```

**[Scott Spence's reliability testing](https://scottspence.com/posts/measuring-claude-code-skill-activation-with-sandboxed-evals):** In 200+ tests across different prompt types, Spence found that simple "suggestion" hooks for skill activation hit only ~50% reliability. A forced evaluation pattern - where Claude is required to explicitly evaluate each skill's relevance before proceeding - hit 80-84%. His [follow-up post](https://scottspence.com/posts/how-to-make-claude-code-skills-activate-reliably) digs into how to make skills activate reliably. The takeaway: if auto-invocation matters to you, invest in description quality and consider evaluation hooks.

### 3. Does this task need isolated context, restricted tools, or a specialized persona?

**Use a Subagent.**

Subagents get their own context window, their own system prompt, and their own tool permissions. Use them when the work would pollute your main conversation, when you need tool restrictions for safety, or when the task is truly independent.

Good candidates:
- Code review (isolated from the implementation context)
- Research/exploration that generates verbose output
- Security audits (read-only tool access)
- Architecture analysis that shouldn't modify code
- Any task where you want the result but not the intermediate reasoning

#### Use subagents sparingly

This is critical, and it's not just me saying it.

Anthropic's own prompting docs for Claude Opus 4.6 explicitly state that it "has a strong predilection for subagents and may spawn them in situations where a simpler, direct approach would suffice." Their recommendation: add explicit guidance about when subagents are and aren't warranted:

> Use subagents when tasks can run in parallel, require isolated context, or involve independent workstreams that don't need to share state. For simple tasks, sequential operations, single-file edits, or tasks where you need to maintain context across steps, work directly rather than delegating.

The [shanraisshan best practices repo](https://github.com/shanraisshan/claude-code-best-practice) recommends using "feature-specific subagents (extra context) with skills (progressive disclosure) instead of general QA, backend engineer" agents. If you're creating generic-persona agents, you've overcomplicated things. Feature-specific subagents paired with focused skills is the pattern that works.

The token cost is real, too. Every subagent gets its own context window. If you're spinning up agents for things that could be a direct tool call, you're burning tokens for no reason. Delegate verbose, isolated work. Do simple work directly.

#### Subagent memory

Subagents can maintain persistent memory directories that survive across conversations:

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
memory: user
---
```

The `memory` field scopes where the persistent directory lives:
- `user`: Memory persists across all your projects
- `project`: Memory is scoped to the current project

This turns a stateless reviewer into one that learns your codebase. The subagent can read and write to this directory across sessions, building up patterns, conventions, and recurring issues over time.

### 4. Is this project-level context that Claude needs in every session?

**Use CLAUDE.md.**

CLAUDE.md is for stable, always-true information. It loads at the start of every conversation and should act as a **router**, not an encyclopedia.

[Alex from alexop.dev](https://alexop.dev/posts/claude-code-customization-guide-claudemd-skills-subagents/) captures the consensus: "Use CLAUDE.md for short, always-true project conventions and standards. Use skills when you want Claude to auto-apply a richer workflow, often with supporting files." His [other post on progressive disclosure](https://alexop.dev/posts/stop-bloating-your-claude-md-progressive-disclosure-ai-coding-tools/) is worth reading if you're finding your CLAUDE.md growing out of control.

Good content for CLAUDE.md:
- Project structure overview
- Tech stack and key dependencies
- Build/test/lint commands
- Coding conventions and constraints (use MUST/MUST NOT language - "Prefer TypeScript" is a suggestion Claude might ignore; "MUST use TypeScript strict mode" is a rule it tends to follow)
- Pointers to skills, docs, and reference material
- Team norms (commit message format, PR process, etc.)

**Keep CLAUDE.md lean.** Community consensus suggests aiming for under 150 lines. As one practitioner noted: "/memory, /rules, constitution.md does not guarantee anything." This is why the layered approach matters - CLAUDE.md sets the baseline, hooks enforce the non-negotiables, and skills handle the complex procedures.

**Use path-scoped rules for directory-specific context.** Instead of bloating CLAUDE.md, put rules in `.claude/rules/*.md` with frontmatter globs:

```
.claude/rules/
├── api-rules.md       # paths: src/api/**
├── frontend-rules.md  # paths: src/components/**
└── test-rules.md      # paths: tests/**
```

These load only when Claude accesses files matching the glob pattern. Your CLAUDE.md stays lean; your rules stay relevant.

### 5. Does this need to persist across sessions?

**Use the right memory mechanism for the scope.**

| Memory Type | Scope | How It Works | Best For |
|---|---|---|---|
| CLAUDE.md | Project | Loaded every session. You maintain it manually. | Stable facts, conventions |
| [Session Memory](https://code.claude.com/docs/en/memory) | Project | Automatic. Claude writes summaries to disk; loads them in future sessions. | "What did I do yesterday?" continuity |
| Subagent Memory | User or Project | Persistent directory a subagent reads/writes across conversations. | Domain knowledge that accumulates |
| Tasks | Project | Filesystem-persisted DAGs (`~/.claude/tasks`). Survive across sessions and compaction. | Multi-session project tracking with dependencies |
| [Memory Tool](https://docs.claude.com/en/docs/agents-and-tools/tool-use/memory-tool) (API) | Custom | Client-side file directory for just-in-time retrieval. You control storage. | Long-running agentic workflows |

#### The structured recovery pattern

For long-running projects spanning multiple sessions, memory files need to be bootstrapped deliberately:

1. **Initializer session:** Set up a progress log (what's done, what's next), a feature checklist (scope of work), and references to startup scripts.
2. **End-of-session update:** Before ending, update the progress log with what was completed and what remains.
3. **Work on one feature at a time.** Only mark complete after end-to-end verification. This keeps the progress log trustworthy.

This turns memory into a structured recovery mechanism, so each new session picks up exactly where the last one left off - rather than Claude spending the first five minutes being re-explained what you built yesterday.

### 6. Does Claude need real-time code intelligence for this project?

**Configure LSP via `.lsp.json`.**

[LSP (Language Server Protocol)](https://code.claude.com/docs/en/discover-plugins) integration gives Claude IDE-like code intelligence: go-to-definition, find-references, hover info, symbol search, and real-time diagnostics. This means Claude navigates your codebase structurally rather than via text search - reportedly ~900x faster for navigation operations.

LSP plugins can be configured in `.lsp.json` at the plugin root, or inline in `plugin.json`:

```json title=".lsp.json"
{
  "typescript": {
    "command": "typescript-language-server",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".ts": "typescript",
      ".tsx": "typescriptreact",
      ".js": "javascript",
      ".jsx": "javascriptreact"
    }
  }
}
```

> [!tip] LSP plugins configure the connection, not the server
> You still need to install the language server binary separately (e.g., `npm install -g typescript-language-server`, `pip install pyright`, `go install golang.org/x/tools/gopls@latest`).

Available via the [official Anthropic marketplace](https://github.com/anthropics/claude-plugins-official) and community marketplaces like [Piebald-AI's claude-code-lsps](https://github.com/Piebald-AI/claude-code-lsps) (covering 17+ languages). Install with `/plugin marketplace add` then `/plugin install`.

> [!note] LSP is still maturing
> LSP support was officially added in v2.0.74 (December 2025). There are some known issues with plugin propagation. If things aren't working, check the `/plugin` Errors tab and verify the language server binary is in your PATH.

---

## Composable Workflow Patterns

The real power isn't in any single feature - it's in how they compose. These patterns are drawn from what the community has found works in practice. I use several of them in my own plugins.

### Pattern 1: Skill + Hook (Guided Execution with Guardrails)

The skill defines *what* to do. The hook ensures *quality gates* are met. This is the "babysitting elimination" pattern.

```
User invokes /deploy
  → Skill loads: deployment checklist, environment config template
  → Claude follows the procedure
  → Stop hook (agent type): verifies all tests pass and changelog is updated
  → PostToolUse hook: formats any modified files
```

The skill gives Claude the procedure; the hooks enforce the non-negotiable checkpoints. You get the benefits of Claude's reasoning on the complex parts, with deterministic enforcement on the mechanical parts.

### Pattern 2: Skill + Subagent (Divide and Verify)

The skill orchestrates; the subagent does isolated work.

```
User invokes /implement-feature
  → Skill loads: feature implementation workflow
  → Step 1: Claude writes the code (main context)
  → Step 2: Skill instructs Claude to delegate review to a subagent
  → Subagent (read-only tools) reviews the code, returns findings
  → Step 3: Claude addresses findings in main context
```

The subagent's verbose analysis stays out of the main context. Only the summary comes back. This mirrors how real engineering teams work: specialization without fragmentation.

### Pattern 3: Context-Dependent Branching (XML-Structured)

A single skill that adapts based on how it's invoked or what it finds. The XML structure makes branching explicit - Claude doesn't have to guess which path to take.

```xml
<workflow>
  <step name="detect-context">
    Check the current branch name and recent git log.
    Determine if this is a feature branch, hotfix, or main.
  </step>

  <step name="feature-branch" condition="feature branch detected">
    Follow the standard PR preparation workflow.
    Read templates/pr-template.md for the PR body format.
    Read ../_shared/steps/run-quality-gates.md for pre-PR checks.
  </step>

  <step name="hotfix" condition="hotfix branch detected">
    Follow the expedited review process.
    Skip changelog generation. Flag for immediate review.
    Read templates/hotfix-template.md for the PR body format.
  </step>
</workflow>
```

### Pattern 4: Shared Workflow Steps (The Composability Pattern)

Reusable workflow steps that multiple skills can reference. This is the closest thing to "shared libraries" that works today, and it's how I structure my plugin marketplace.

```
.claude/skills/
├── _shared/                          # Shared reference library
│   ├── SKILL.md                      # description + index of shared resources
│   ├── steps/
│   │   ├── validate-environment.md   # Reusable step: check env vars, deps
│   │   ├── run-quality-gates.md      # Reusable step: lint, test, type-check
│   │   └── prepare-commit.md         # Reusable step: stage, message, sign
│   ├── templates/
│   │   ├── pr-body.md
│   │   └── changelog-entry.md
│   └── scripts/
│       ├── check-deps.sh
│       └── validate-schema.sh
├── deploy/
│   └── SKILL.md                      # References: ../_shared/steps/validate-environment.md
├── implement-feature/
│   └── SKILL.md                      # References: ../_shared/steps/run-quality-gates.md
└── create-pr/
    └── SKILL.md                      # References: ../_shared/steps/prepare-commit.md
```

Each skill's SKILL.md explicitly tells Claude to read the shared step file:

```markdown
## Step 3: Quality Gates
Read and follow the procedure in `../_shared/steps/run-quality-gates.md`.
Execute the script at `../_shared/scripts/validate-schema.sh` and check its output.
```

This works because shared files live in one place (single source of truth), there's no context cost until Claude reads the referenced file, scripts are *executed* not *read*, any skill can reference any shared step, and changes propagate immediately.

> [!warning] Cross-plugin sharing is limited
> For cross-plugin sharing in a marketplace, you're limited to symlinks (which resolve at install time, giving each plugin its own copy). Alternatively, consider a dedicated "commons" plugin that other plugins reference. This is fragile and undocumented - worth watching as the plugin system matures.

### Pattern 5: Hooks as Autonomous Workflow Orchestrators

For fully autonomous workflows, hooks create a closed loop where Claude works, gets verified, and persists state - all without you watching.

```
SessionStart hook
  → Injects today's task list from tasks/ directory
  → Loads relevant session memory

PreToolUse hook (on UserPromptSubmit)
  → Re-injects critical rules (the compaction survival pattern)

PostToolUse hook (on Write/Edit)
  → Runs formatter
  → Updates dirty-file tracker

Stop hook (agent type)
  → Subagent verifies: did Claude complete what was asked?
  → If not: returns blocking signal with "Tests failing. Fix before finishing."
  → If yes: updates progress log, writes session summary

Notification hook
  → Desktop notification when Claude finishes or needs input
```

> [!warning] Two gotchas worth knowing
> **Formatter noise:** If your formatter changes files in a PostToolUse hook, Claude gets a system reminder about those changes every time, eating into your context window. The smarter approach for heavy formatting is to format on commit via a Stop hook rather than after every individual edit.
>
> **Always check `stop_hook_active` in Stop hooks.** When it's true, Claude is already continuing because of a previous Stop hook. Exit 0 immediately. Without this check, your hook blocks Claude forever. This is the #1 mistake new hook authors make.

---

## Sandboxing Integration

[Sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing) isn't just a security feature - it's an autonomy enabler. Anthropic's internal usage data shows sandboxing reduces permission prompts by 84%. With sandboxing on, you can grant broader permissions because the blast radius is contained.

### When to enable sandboxing

**Always, unless you have a specific reason not to.** Anthropic recommends sandboxing as the default configuration for all environments. It adds <15ms of latency and provides:
- **Filesystem isolation:** Writes restricted to working directory and subdirectories
- **Network isolation:** Only approved domains reachable, via a proxy server outside the sandbox
- **OS-level enforcement:** Uses macOS Seatbelt or Linux bubblewrap - not prompt-based, not bypassable by Claude, covers all subprocesses

Both isolation layers are critical. Without network isolation, a compromised agent could exfiltrate SSH keys. Without filesystem isolation, a compromised agent could escape the sandbox and gain network access.

### Sandboxing + Hooks synergy

This is the combination that turns Claude Code into a genuinely autonomous tool:

- **Without sandbox:** You need conservative permission rules → more prompts → more babysitting
- **With sandbox + auto-allow mode:** Sandboxed commands run automatically without prompting. Hooks handle quality gates. You handle only the exceptions that escape the sandbox boundary.

The result: Claude works freely within defined limits, hooks enforce your standards deterministically, and you only get interrupted for things that genuinely need your attention.

### Things that need to escape the sandbox

These gracefully fall back to the normal permission flow - you don't lose safety, you just get prompted:
- Docker commands (need daemon access - add `docker` to `excludedCommands`)
- Network calls to non-allowlisted domains
- Git operations requiring authentication
- Anything writing outside the project directory

---

## Anti-Patterns to Avoid

I've hit every single one of these. Learn from my mistakes.

### 1. The "God Skill"
A single skill that tries to handle every workflow. Split it. One skill, one job. Multiple focused skills that compose are always better than one massive skill. My first plugin version had one of these. It was 800 lines of SKILL.md. Claude got confused constantly. Don't be me.

### 2. The Agent Army
Creating subagents for everything - especially generic-persona agents like "backend engineer" or "QA tester." Anthropic explicitly warns that Opus 4.6 over-spawns subagents. Feature-specific subagents with skills beat general-purpose agents. If you have more than 3-4 subagents, you've probably overcomplicated things.

### 3. CLAUDE.md as Documentation
CLAUDE.md is not your project wiki. It's a bootstrap file and a router. Keep it under 150 lines. Extract everything else into skills, rules, and reference files. Your CLAUDE.md should become a router pointing to the right context, with each referenced file having a specific job.

### 4. Prompting What Should Be Hooked
If you find yourself repeatedly telling Claude to run the linter, format code, or check tests - that's a hook, not a prompt. "Prompts are suggestions; hooks are guarantees." If you're typing "please remember to run prettier" by hand, you're doing it wrong.

### 5. Documenting Procedures Instead of Attaching Scripts
Instead of writing step-by-step shell commands in SKILL.md, put the logic in a script file within the skill directory. Scripts execute via bash; only their output enters the context window. The script's source code never consumes tokens.

### 6. Putting Output Templates Inline in SKILL.md
Templates belong in separate files within the skill directory. They get read on-demand (no context cost until needed), they're easier to maintain, and they keep SKILL.md focused on instructions rather than output structure.

### 7. Ignoring Compaction and Context Rot
Context rot - where Claude gradually deprioritizes earlier instructions - is the primary failure mode in long sessions. Multiple community voices call this the #1 issue. Mitigate with:
- **PreToolUse hooks** that re-inject critical rules (the compaction survival pattern)
- **Session memory** for automatic cross-session persistence
- **Tasks** for structured progress tracking that survives compaction
- **Aggressive `/clear` usage** when switching topics
- **Subagent delegation** to keep the main context clean

One developer's mantra captures it: "Compound, don't compact. Extract learnings automatically, then start fresh with full context."

### 8. Overusing MCP Where a Skill Would Suffice
I've seen developers building bloated MCP servers with dozens of tools that just mirror a REST API (`read_thing_a()`, `read_thing_b()`, `update_thing_c()`). The "scripting" model - formalized by skills - is better for most cases. MCP's focused role should be as a secure gateway providing a few powerful, high-level tools, not as a replacement for domain knowledge and procedures that belong in skills.

---

## Quick Reference: "Where Does This Go?"

| What You Want | Where It Goes | Why |
|---|---|---|
| "Always format on save" | Hook (PostToolUse) | Deterministic, no tokens |
| "Block rm -rf" | Hook (PreToolUse) | Must happen every time |
| "Survive compaction" | Hook (PreToolUse on UserPromptSubmit) | Re-injects rules every prompt |
| "Notify me when done" | Hook (Notification/Stop) | No more watching the terminal |
| "How to deploy" | Skill | Multi-step procedure with judgment |
| "Our API conventions" | Skill or Rule | Domain knowledge, loaded on-demand |
| "Generate a component" | Skill + template file | Template stays out of context until read |
| "Run validation logic" | Skill + script file | Script output enters context, not source |
| "Review this code" | Subagent (read-only tools) | Needs isolated context |
| "Project uses TypeScript strict" | CLAUDE.md | Always-true constraint |
| "Test command is `pnpm test`" | CLAUDE.md | Stable project fact |
| "Frontend rules" | `.claude/rules/frontend.md` | Path-scoped, loads only when relevant |
| "What I did yesterday" | Session Memory (automatic) | Cross-session continuity |
| "Track feature progress" | Tasks | Persistent DAGs, survives compaction |
| "Reusable PR template" | Skill directory (`templates/`) | On-demand, no upfront cost |
| "Shared validation step" | `_shared/` skill directory | Single source of truth, referenced by other skills |
| "Type info and go-to-definition" | LSP plugin (`.lsp.json`) | Structural navigation, ~900x faster |

---

## On Hook Events

There are 17 hook events in Claude Code as of February 2026. Here's the complete list:

| Event | When it fires | Can block? |
|---|---|---|
| SessionStart | Session begins/resumes | No |
| UserPromptSubmit | User submits a prompt | Yes |
| PreToolUse | Before a tool call | Yes |
| PermissionRequest | Permission dialog appears | Yes |
| PostToolUse | After tool succeeds | No (feedback only) |
| PostToolUseFailure | After tool fails | No (feedback only) |
| Notification | Notification sent | No |
| SubagentStart | Subagent spawned | No |
| SubagentStop | Subagent finishes | Yes |
| Stop | Claude finishes responding | Yes |
| TeammateIdle | Teammate about to idle | Yes |
| TaskCompleted | Task marked completed | Yes |
| ConfigChange | Config file changes | Yes |
| WorktreeCreate | Worktree being created | Yes |
| WorktreeRemove | Worktree being removed | No |
| PreCompact | Before compaction | No |
| SessionEnd | Session terminates | No |

The ones you'll use most: **PreToolUse** (blocking dangerous commands, re-injecting rules), **PostToolUse** (formatting after edits), **Stop** (verification before Claude finishes), and **Notification** (desktop alerts).

---

## Getting Started

If you're building from scratch, layer things in this order. Each layer should solve a real friction point you've experienced - don't pre-optimize.

1. **CLAUDE.md** - Get the basics: project structure, commands, constraints. Keep it lean. Use MUST/MUST NOT language for rules you care about.
2. **Sandboxing** - Run `/sandbox` and enable it. Immediate reduction in permission noise.
3. **Two hooks** - Auto-format (PostToolUse) and notifications (Stop/Notification). Ten minutes of setup. Hours of saved babysitting.
4. **Your first skill** - Pick your most common workflow. Package it with a template file and at least one script. Use XML for the workflow structure. Write a specific description with WHEN/WHEN NOT conditions.
5. **LSP** - If you're working in a typed language, install the language server and the LSP plugin. Claude's code navigation goes from text-search to structural.
6. **Shared steps** - As you build more skills, extract common procedures into `_shared/`. Reference them from other skills. Attach scripts instead of documenting procedures.
7. **Path-scoped rules** - As CLAUDE.md grows, move directory-specific guidance into `.claude/rules/*.md` with frontmatter globs.
8. **Subagents** - Only when you genuinely need isolated context or restricted tools. Feature-specific, not generic-persona.
9. **Memory/Tasks** - When your projects span multiple sessions and you need continuity. Bootstrap deliberately with the structured recovery pattern.

The difference between frustration and productivity isn't the tool - it's how you layer the pieces. Start with what hurts most, automate it, and build from there.

Happy building, everyone.

---

## Mentioned in this post

- [Claude Code Hooks Documentation](https://code.claude.com/docs/en/hooks-guide) - Official Anthropic docs for hook events and configuration
- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills) - Official docs for creating and managing skills
- [Anthropic's Sandboxing Engineering Blog](https://www.anthropic.com/engineering/claude-code-sandboxing) - "Beyond permission prompts: making Claude Code more secure and autonomous"
- [shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice) - Community-maintained best practices repository
- [Shrivu Shankar - "How I Use Every Claude Code Feature"](https://blog.sshh.io/p/how-i-use-every-claude-code-feature) - The post that framed skills as a "scripting" model
- [Simon Willison - "Claude Skills are awesome, maybe a bigger deal than MCP"](https://simonwillison.net/2025/Oct/16/claude-skills/) - The post that started the skills-vs-MCP conversation
- [Lakshmi Narasimhan - "Claude Code Hooks: The Feature You're Ignoring"](https://medium.com/@lakshminp/claude-code-hooks-the-feature-youre-ignoring-while-babysitting-your-ai-789d39b46f6c) - Practical hooks guide
- [Scott Spence - Measuring Claude Code Skill Activation](https://scottspence.com/posts/measuring-claude-code-skill-activation-with-sandboxed-evals) - Reliability testing for skill auto-invocation
- [alexop.dev - Claude Code Customization Guide](https://alexop.dev/posts/claude-code-customization-guide-claudemd-skills-subagents/) - Separation of concerns between CLAUDE.md and skills
- [alexop.dev - "Stop Bloating Your CLAUDE.md"](https://alexop.dev/posts/stop-bloating-your-claude-md-progressive-disclosure-ai-coding-tools/) - Progressive disclosure for AI coding tools
- [Piebald-AI/claude-code-lsps](https://github.com/Piebald-AI/claude-code-lsps) - Community LSP plugin marketplace covering 17+ languages
- [Claude Code Memory Documentation](https://code.claude.com/docs/en/memory) - Managing Claude's persistent memory
- [Anthropic Memory Tool (API)](https://docs.claude.com/en/docs/agents-and-tools/tool-use/memory-tool) - Memory tool for long-running agentic workflows
- [brennacodes/brenna-plugs](https://github.com/brennacodes/brenna-plugs) - My Claude Code plugin marketplace (the one that taught me all of this)
