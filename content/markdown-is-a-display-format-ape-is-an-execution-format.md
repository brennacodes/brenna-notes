---
title: "Markdown Is a Display Format. APE Is an Execution Format."
description: "Why getting LLMs to do what you want is a linguistics problem, not a prompting problem - and how a markup language might fix it."
date: 2026-03-12
tags:
  - programming
  - ai-agents
  - agentic-coding
  - language-design
  - devlog
draft: true
author: "Brenna"
---

I've been thinking about why it's so hard to get an LLM to do what you want - and I think the answer has less to do with prompting technique and more to do with a problem linguists identified decades ago.

In the 1960s, a philosopher named J.L. Austin described three things that happen every time someone says something. Three layers of meaning, stacked on top of each other, all happening at once. And if you've ever watched an LLM confidently do the *wrong* thing after reading your very clear instructions, you've experienced the exact problem Austin was describing - just with a machine on the other end instead of a person.

## The Three Layers of Saying Things

Austin called them **locution**, **illocution**, and **perlocution**. Stay with me - this is less academic than it sounds.

**Locution** is the literal utterance. The words, in order, as written. "It's cold in here."

**Illocution** is the intended force. What you *meant* by saying it. "Please close the window."

**Perlocution** is the effect on the listener. What actually happens as a result. Maybe they close the window. Maybe they hand you a sweater. Maybe they say "yeah, it is" and keep scrolling.

Three layers. Three chances for the meaning to drift.

Now here's the thing: when a human says "it's cold in here" to another human, there's a whole universe of shared context - body language, room temperature, relationship history, social norms - that helps the listener land on the right interpretation. We're pretty good at it. Not perfect, but pretty good.

LLMs don't have that universe.

## The Double Drift Problem

When you write instructions for an LLM, semantic drift happens *twice*.

First, at the **authoring end**. You have an intent - something you want the agent to do. You encode that intent into text. But text is lossy. Your intent passes through your vocabulary, your assumptions about how the reader will interpret things, your implicit mental model of how the task should flow. By the time it's written down, it's already a *translation* of what you meant, not a *transcription*.

Then, at the **interpretation end**. The LLM reads your text and builds its own model of what you meant. But it's pattern-matching against its training distribution, not against your intent. It has no access to what you were thinking when you wrote it. It has the locution - the literal words - and it's trying to reconstruct the illocution - the intended force - from that alone.

Two translation layers. Two places where meaning can slip. The gap between human "want" and machine "do" is where most prompt engineering pain lives.

And here's what I think makes this fundamentally harder than the human version: with humans, you can go back and forth. You can clarify. You can read the room. With an LLM executing a workflow, the instructions are fired and forgotten. The agent reads them once and runs.

## Where Markdown Falls Down

So we have this drift problem. How do most people write LLM instructions today?

Markdown. System prompts. Scattered files full of prose.

And markdown is great! I love markdown. But here's the thing - **markdown is a display format**. It was designed to describe how things should *look*. Headers, bold text, lists, code blocks - these are visual affordances. They tell a renderer how to present content to a human reader.

When you write LLM workflow instructions in markdown, you're using a display format for an execution task. You're describing *appearance* and hoping the agent infers *behavior*. Every heading, every bullet point, every bold phrase is a locution that the LLM has to decode into an illocution.

"## Step 3: Validate the Output" - is that a suggestion? A requirement? What happens if validation fails? Do I stop? Try again? Skip it? The markdown doesn't say. It *can't* say, because it doesn't have the vocabulary for execution semantics. Markdown has no concept of gates, failure handlers, prerequisites, or scope.

You end up compensating with prose: "IMPORTANT: Do not proceed until..." and "NOTE: If this fails, go back to step 2 and..." and "Make sure you..." All of which are illocutionary cues stuffed into a format that has no structural way to enforce them.

The LLM might follow them. It might not. It depends on how much attention it pays to your capitalized "IMPORTANT" versus the structural pull of the next heading.

## APE: What Things Should *Be*

This is the problem I built APE to solve.

APE - Applied Primitive Expression - is an XML markup language designed to treat the LLM as a runtime execution engine. Where markdown describes how things should *look*, APE describes what things should *be*.

That's the core distinction. **Markdown is a display format. APE is an execution format.**

In APE, the document *is* the workflow. It's self-contained. It declares who does what, what tools to use, when to stop and wait, and what to do on success or failure. Hand it to an agent; it runs.

The whole language boils down to two primitives:

- **`<command>`** - things you *do*. Shell commands, actions, decisions.
- **`<resource>`** - things you *need*. Files, tools, services.

Everything else is flow control and metadata. Steps, gates, variables, actors, constraints - these are all structural scaffolding around the two things that actually matter: what you do and what you need to do it.

Here's what a gate looks like in APE versus markdown:

**In markdown:**

```markdown
## Step 3: Run Tests

Run the test suite. **IMPORTANT:** All tests must pass before
proceeding. If any tests fail, fix them before moving on.
```

**In APE:**

```xml
<step id="run-tests" number="3">
  <description>Run Tests</description>
  <instruction>Run the test suite.</instruction>
  <gate>
    <criteria>All tests pass.</criteria>
    <on-fail>Fix failing tests before proceeding.</on-fail>
  </gate>
</step>
```

See the difference? In the markdown version, "all tests must pass before proceeding" is a bolded sentence embedded in a paragraph. It's a request. A suggestion with extra formatting.

In the APE version, the gate is *structural*. It's not a sentence the LLM has to decide is important - it's a node in a tree that the execution model has to pass through. The `<on-fail>` isn't a footnote - it's a handler. The semantics aren't encoded in prose; they're encoded in the document's shape.

## Collapsing the Illocutionary Gap

Here's where speech act theory comes back.

The fundamental problem with prose-based LLM instructions is the gap between locution and illocution. You write words, and you hope the agent infers your intent. APE tries to collapse that gap by making intent *structural* rather than *conversational*.

When a `<gate>` has `<criteria>` and `<on-fail>`, the illocution isn't ambiguous. The author didn't write "please make sure" and hope the LLM would interpret the force of that correctly. They declared a structural constraint that has a defined behavior when it's not met.

When a `<resource>` is marked `required="true"`, that's not a "NOTE: you'll need..." buried in a paragraph - it's a declaration with semantic weight in the execution model.

When a `<constraint>` appears inside a `<step>`, it doesn't depend on the LLM noticing it was bolded or capitalized. It's a first-class element scoped to that step.

The hypothesis is simple: if you give authors a way to express intent structurally instead of conversationally, you reduce the surface area for semantic drift. The author doesn't have to hope their prose carries the right illocutionary force. They just... declare what they mean.

## Why XML?

I can already hear it. "XML? In 2026? Really?"

Yeah, really. And here's why.

Structure is the whole point. APE is trying to encode execution semantics - gates, scopes, failure handlers, prerequisites, conditional flows. These are inherently tree-shaped. XML gives you a tree where the tag names carry semantic meaning.

JSON would work for the data model, but have you ever tried to write a workflow by hand in JSON? Matching brackets, no comments, no mixed content. It's miserable for humans. YAML is better to write but worse to parse unambiguously, and it has no schema language worth using.

XML has XSD for structural validation, it supports mixed content (prose + child tags in the same element), it's self-documenting (tags literally say what they mean), and every language on the planet can parse it.

Is it verbose? Sure. But the verbosity is *clarity*. `<gate>` is clearer than whatever convention you'd invent in markdown to mean "this is an enforceable checkpoint."

## What's Next

APE is at v0.2.2-draft. The spec is written. The schema validates. The authoring guide teaches you how to think in commands and resources. The LLM execution contract tells agents how to interpret and run APE documents.

What I don't have yet are benchmarks. The benchmark suite is built - it tests how workflow instructions in different formats (APE XML, markdown, plain text) perform against real apps with realistic prompts, running in isolated workspaces. But the results are still pending.

That's the next milestone, and it's the one that matters most. The hypothesis that structural intent reduces semantic drift is either going to show up in the numbers or it isn't. I think it will - but I'm building the tools to prove it, not just argue it.

If you've ever stared at a system prompt wondering why the LLM keeps ignoring your very clear instructions, you've felt the illocutionary gap. APE is my attempt to close it - not by writing better prose, but by giving authors a language where the structure *is* the intent.

Markdown describes how things should look. APE describes what things should be. And the bet is that "what things should be" is a lot closer to what we actually mean.
