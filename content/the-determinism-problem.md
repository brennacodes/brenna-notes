---
title: "The Determinism Problem - What Building AI Tools Actually Taught Me About AI"
description: "What building AI tools actually taught me about AI - determinism, auditability, trigger words, and why the conversation IS the product."
date: 2026-03-28
tags:
  - agentic-coding
  - ai-agents
  - programming
  - devlog
  - ai-tooling
  - ai-development
  - ai-dev
draft: false
author: "Brenna"
---

I've spent the last year-plus building AI tooling. Not using AI to build things (though I do that too) - building the tooling itself. An interface to manage and oversee your sea of AI files. Plugin suites for [Claude Code](https://github.com/brennacodes/brenna-plugs). A marketplace of workflow plugins tuned to my team's development processes. A [markup specification](https://blog.brennacodes.com/markdown-is-a-display-format-ape-is-an-execution-format) for structured agent workflows. Development lifecycle systems with database-backed state machines. And through all of it, I've been working with AI as a daily development partner across personal and professional projects.

What follows are the things I actually learned. Not the advice you'll find in "how to prompt better" threads, but the structural, architectural, you-will-hit-this-wall-eventually realizations that changed how I build.

## What's in here

- [Every Peg Fits the LLM-Shaped Hole](#every-peg-fits-the-llm-shaped-hole) - AI defaults to inserting itself everywhere
- [The Determinism Problem](#the-determinism-problem-1) - building predictability on top of unpredictability
- [Auditability in a Forgetful System](#auditability-in-a-forgetful-system) - why persistence is structural, not optional
- [The Danger of What Is Said and Unsaid](#the-danger-of-what-is-said-and-unsaid) - trigger words, invisible behaviors, and cascading assumptions
- [Spec-Driven Development Isn't Optional](#spec-driven-development-isnt-optional) - why structure beats vibes, every time
- [Show the Shape You Want](#show-the-shape-you-want) - leading by example, not instruction
- [Own Your Context](#own-your-context) - the shared window is your responsibility

## Every Peg Fits the LLM-Shaped Hole

Here's one of the most persistent patterns I run into: when you build AI-integrated systems *with AI assistance*, the AI defaults to inserting itself into every part of the process. Even when a deterministic solution exists. Even when the deterministic solution is objectively better.

I was designing a development lifecycle - phases like discovery, spec, plan, implement, test, review, deploy. Each phase needed actions, actors, and transitions. The AI's first instinct? Assign "agent" as the actor for *nearly everything*. "Agent determines mode." "Agent decides what to look for." "Agent evaluates sufficiency."

But hold on.

- Whether discovery runs in directed, undirected, or hybrid mode is a function of whether the user provided sources and how specific the description is. That's not inference. That's an `if` statement.
- "Did we evaluate all provided sources?" is a SQL query, not a judgment call.
- "Can we advance from discovery to spec?" is a checklist of conditions against database state, not a vibes check.

The principle I landed on: **`process > agent`** (whether `agent > human` is determined by the process, not by inference). If it can be deterministic, make it deterministic. Agent handles the stuff that genuinely requires inference (or where you genuinely don't want to insert yourself) - code generation, interpreting ambiguous requirements, indentifying and analyzing patterns. Human should always handle the high-stakes calls that neither process nor agent should make unilaterally.

This showed up everywhere. Building plugin marketplaces, the AI wanted to solve configuration problems with inference instead of structured config files. When I first started building out the benchmarking suite for [APE](https://blog.brennacodes.com/markdown-is-a-display-format-ape-is-an-execution-format), the AI essentially decided it would be its own judge. It took something that should be a simple pass/fail and turned it into an open-ended evaluation. Every time, the correction was the same: make it deterministic, make it explicit, make it enforceable.

## The Determinism Problem

Everyone building AI tooling hits the same wall. You're trying to build predictable, reliable systems on top of a component that is inherently unpredictable. That's the whole value proposition - the LLM can do things no deterministic program can. It can also do things no deterministic program *would*. Like hallucinating a state transition. Or skipping half the steps in a workflow. Or reading one word in its context and treating it as gospel for the rest of the session.

The answer isn't "avoid using the LLM." The answer is **constrain it with deterministic boundaries**. Most people attempt to do this with plain text by saying things like "if this happens, do that" or "follow these rules exactly." But plain text is inherently ambiguous - the LLM can interpret the same instruction in multiple ways, or simply decide your rules weren't important enough to follow and ignore them entirely. But in a language-based system, if prose can't enforce consistency, what can?

Your options depend on your use case. I've used anything from simple markdown files to JSON or YAML files, to inventing a markup specification, to database-driven state machines. The key insight is that the more you can codify the rules, the better.

### On Database-Driven State Machines

This isn't the right answer for every problem, but it's a powerful pattern when it is. Why? Because it helps constrain the LLM to inputs and outputs. This pattern has multiple benefits:
- It provides clear, deterministic boundaries for what the LLM can and cannot do
- It makes the process auditable
- It makes context composable (and reusable)
- It narrows the number of necessary permissions to database operations and a few predefined actions

I call this the DB-as-controller pattern. The database defines what phase the work is in, what transitions are valid, what actions are permitted, and what inputs exist. The LLM operates inside those boundaries. It can't hallucinate a jump from "discovery" to "deploy" because the transition gate will fail - the spec doesn't exist, the plan doesn't exist, the tests haven't run. The database says no, and the database doesn't hallucinate.

```sql
-- This is a deterministic gate. The LLM doesn't get a vote.
SELECT COUNT(*) FROM artifacts
WHERE work_item_id = ? AND phase = 'spec' AND approved = 1;
-- If this returns 0, you're not advancing to plan. Period.
```

This is also why I'm moving away from file-based storage toward SQLite for my personal plugin suite. A JSON file can't reject an invalid state. A markdown file with YAML frontmatter can't enforce referential integrity. SQLite can. `CHECK` constraints, foreign keys, `NOT NULL` - these are deterministic guardrails that work regardless of what the LLM is doing.

The realization that stuck: **the architecture of your data layer is the architecture of your determinism.** Fragile data layer, fragile determinism, unpredictable system. Your database isn't just where you store things. It's your enforcement mechanism.

## Auditability in a Forgetful System

Here's a fun constraint: the AI you're building with will literally forget what it did 30 minutes ago if the context window fills up.

Claude Code sessions have finite context windows. They get compacted - previous conversation is summarized and compressed. Information is lost. You can't engineer around this. You can only mitigate it.

That means: **if an action isn't recorded in persistent storage when it happens, it didn't happen.** Not in any operationally useful sense.

Say the agent evaluates a source file and finds important patterns but doesn't record the findings to the database immediately. The context compacts. Those findings? Gone. The agent would need to re-read the file - except it doesn't know it needs to, because the record of having read it was also compacted. So it proceeds without that context, building on an incomplete foundation.

Three rules fell out of this:

**Record when it happens, not after.** Every meaningful action gets persisted at the moment it occurs. Not batched. Not summarized later. Now.

**Make the audit trail queryable.** A markdown log that says "evaluated auth module, found middleware pattern" is fine for a human skimming later. It's useless for a process trying to answer "have we evaluated all provided sources?" The data needs structure. Tables with columns, not freeform text.

**What you record shapes what you can do later.** I ended up justifying every single column in every table by asking: "what future phase, query, or decision needs this data?" If I couldn't answer that, it didn't get stored. But the inverse is worse - if you *can* answer that and you *don't* store it, you've created a gap some downstream phase will hit and can't recover from without a human stepping in.

The pattern that emerged from this: before the agent investigates anything, it records a *strategy* - what it plans to look at, why, and what it expects to find. Then it records outcomes against that plan. If the context compacts mid-work, a resumed session reads the strategy from the DB: "planned to investigate A, B, C. Completed A and B. C is still pending." Without that, a resumed session would re-plan from scratch and potentially investigate completely different things.

## The Danger of What Is Said and Unsaid

This one became clearest during APE benchmarking, but I'd been feeling it everywhere - writing plugin skills, having design conversations, building workflow specs.

In a system that runs on natural language, what you say, how you say it, and what you *don't* say all matter. A lot.

### Single words cascade

This is not hyperbole. Agentic coding tools have keywords that trigger powerful, often invisible behaviors. Some are documented. Many are not. They're baked into system prompts, hidden instructions, or trained behaviors that most users never encounter unless they've done the kind of benchmarking work that surfaces them.

When I was benchmarking APE, this became measurable. A single word in a workflow step could change whether the agent ran tests, skimmed output, or did nothing at all. A word like "carefully" in an instruction could double the token output without adding proportional value. Omitting the word "only" could cause the agent to expand scope dramatically beyond what was intended.

If you don't know what trigger words exist in the system you're building on, you're fighting invisible forces. You write what looks like a clear instruction, and the system interprets it through a layer of keyword-driven behavior you can't see. Knowing that vocabulary - through docs, through benchmarking, through experience - is the difference between predictable results and hours of mystery debugging.

### Framing creates binding constraints

I shared a 2,000-line product blueprint with an AI as *reference material*. The AI immediately treated it as a *specification* and started mapping every decision to that blueprint's schema, trying to couple two systems that serve fundamentally different audiences.

A single framing statement can redirect an entire session. The AI takes everything you say and applies it as context for everything that follows. If you're imprecise, the output will be internally consistent with the wrong interpretation - which means you might not even notice until you're deep into implementation.

The same applies to a simple comment somewhere in your code. The AI will read the comment, internalize it, and use it to guide its behavior. Suddenly you're wondering why the AI is still fighting failing specs. And at some point you realize the comment was incorrect, oudated, or interpreted as a requirement. If you're really unlucky, you'll realize the AI never attempted to understand the actual code at all. This is why I'm almost entirely anti-comment at this point in my career. You, me, our AI - we all need to understand the code. Comments just add another layer of indirection.

### Gaps become assumptions

Describe a 7-phase lifecycle without specifying who determines transitions between phases and the AI will default to "the agent decides." Don't specify what happens when a phase gets triggered by a loop-back and the AI won't address it - or worse, it'll hand-wave something like "the agent re-explores based on feedback."

Every unstated constraint is a constraint the AI will violate. Not maliciously. It just doesn't know the constraint exists.

This is the flip side of the trigger word problem. Trigger words cause behavior you didn't intend. Missing words cause the system to fill in from wherever it fancies. Either way, invisible forces shape your output. The defense is the same in both cases: know the system, and be explicit. But the unspoken rule here is that your system is stronger the smaller it is - the fewer places for gaps to hide, the better.

## Spec-Driven Development Isn't Optional

There's a reason [spec-driven development](https://github.blog/engineering/spec-driven-development-with-ai/) has become the dominant workflow pattern for AI-assisted work, and it's not because developers suddenly love writing documentation.

It's because **ambiguity in, ambiguity out**. A vague prompt produces vague code. Missing constraints produce hallucinated dependencies. Absent schemas produce invented structures. The AI amplifies whatever you give it - and if what you give it is a loose description and good vibes, you get back plausible-looking code that slowly poisons your codebase. As [Addy Osmani](https://addyosmani.com/blog/ai-assisted-engineering/) put it: "the better your specs, the better the AI's output; the more comprehensive your tests, the more confidently you can delegate; and the cleaner your architecture, the less the AI hallucinates weird abstractions."

The spec is the contract. Discovery, then spec, then plan, then implement. Not because it's fun to write specs, but because every phase that skips specification is a phase where the AI fills in the blanks with its own assumptions. And as we've established, those assumptions trend toward "the agent handles it" - which means more inference, more unpredictability, more debugging.

The spec is also quality control. When you have acceptance criteria written before implementation starts, you can verify the output against something concrete. Without that, you're reading AI-generated code and trying to evaluate whether it "feels right" - and it always feels right, because it's internally consistent with whatever interpretation the AI landed on. You need an external reference point, and that's what the spec provides.

The practical workflow I've landed on:

1. **Discover** - gather context, understand the problem space, identify constraints
2. **Specify** - write concrete acceptance criteria that a process can verify
3. **Plan** - decompose into tasks that map to files and have defined verification
4. **Implement** - the AI works against the spec, not from vibes
5. **Verify** - check outputs against criteria, not against "does this look right"

Every step produces artifacts. Every artifact is versioned. Every transition between steps has a deterministic gate. The AI never decides on its own that the spec is "good enough" - a process checks that the required fields exist, the criteria are testable, and a human has approved it.

This isn't heavyweight ceremony. It's the minimum structure needed to prevent the AI from going off the rails. And it gets faster with practice - the spec doesn't have to be a 20-page document. It has to be specific enough that you can tell whether the output matches it.

### What to watch for along the way

**Over-engineering for hypothetical futures.** "We should add a config option for this" when a hardcoded value is fine. "This should be extensible" when there's exactly one use case. The AI loves premature abstraction.

**Under-specifying operational details.** Big-picture architecture diagrams? Easy. "What exactly gets written to the DB when the agent searches an external source, and why do we need that data?" Hard. The AI will happily give you the 10,000-foot view and skip the ground truth.

**Conflating similar things.** "These are basically the same pattern" when they serve different purposes for different audiences. I had a development-time CLI plugin and a future product for non-technical users. The AI kept insisting they were "essentially the same system" because they shared some patterns. They weren't. Different users, different constraints, different architectures.

**Defaulting to comfort.** Prose instead of structured specs. Agent-driven decisions instead of process-driven gates. Generic advice ("just start building") instead of engaging with the actual problem you're trying to solve.

## Show the Shape You Want

When the AI produces output in the wrong shape, telling it "this isn't right" is less effective than *showing* it what right looks like.

Building plugins: writing 20 lines of a SKILL.md in the correct format taught the agent more than 200 words of verbal instruction about what the format should be.

Benchmarking APE: providing a single well-structured workflow step as a reference produced better results than describing the format requirements in prose.

Designing a lifecycle spec: I manually edited one phase to add the skill invocation, parameters table, DB recording details, and operational specifics I wanted. That 50-line edit communicated more about the expected level of detail than everything I'd said up to that point.

The principle: **if you want structured output, provide a structured example.** Demonstrate the thinking on one piece, and the AI applies it to the rest. Show it what "why does this data exist?" looks like for one table column, and it'll start justifying the rest of them.

This connects to knowing when to iterate versus when to reset. If the AI has generated 500 lines in the wrong shape, it's often faster to rewrite 50 lines in the right shape and have the AI regenerate the rest than to patch 500 lines incrementally. Each patch risks introducing inconsistencies with the parts you didn't touch. Sometimes the cheapest move is to show, not tell.

## Own Your Context

The context window is shared. What's in it shapes every response. That makes you responsible for it.

**You control what's loaded.** If you load a 2,000-line blueprint, the AI will use it. If that blueprint is reference material and not a specification, say so explicitly. Otherwise it becomes a specification by default.

**Catch misunderstandings early.** If the AI misinterprets your intent in message 3, every subsequent message compounds that error. Correcting in message 4 is cheap. Correcting in message 20 means unwinding 16 messages of built-up assumptions.

**Know when to start over.** Sometimes the context has diverged so far from where you need to be that it's cheaper to open a new session with clear instructions than to steer the current one back on course. This isn't failure. It's recognizing that context pollution is real and sometimes can't be reversed within a session.

**Persist what matters.** The session is ephemeral. If it produced a useful artifact, save it somewhere durable before the window compacts. If you reached an important conclusion, record it. Your knowledge shouldn't be as temporary as the conversation that produced it.

## The actual takeaway

The real skill in building with AI isn't prompting. It's quality control. Generating output is the cheap part. Evaluating it, catching the shortcuts, correcting the assumptions, verifying the corrections - that's where the value is.

And the single best investment you can make in your AI tooling? Determinism. Every process-driven gate, every database constraint, every structured validation check is a place where your system behaves predictably regardless of what the LLM decides to do. Maximize those, and you'll spend a lot less time debugging behavior you can't explain.

Happy building, everyone.

---

## Mentioned in this post

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) - Anthropic's CLI tool for working with Claude as a development partner
- [APE (Applied Primitive Expression)](https://github.com/Applied-Primitive-Expression/APE) - an XML markup specification for structured LLM agent workflows, designed for portability and inspectability
- [brenna-plugs](https://github.com/brennacodes/brenna-plugs) - the Claude Code plugin marketplace referenced throughout
- [Spec-Driven Development with AI](https://github.blog/engineering/spec-driven-development-with-ai/) - GitHub's take on the spec-first workflow pattern
- [Addy Osmani on AI-Assisted Engineering](https://addyosmani.com/blog/ai-assisted-engineering/) - the source of the "AI rewards good engineering practices" framing
- [Simon Willison's AI Anti-Patterns](https://simonwillison.net/) - "inflicting unreviewed code on collaborators" and other things to avoid
