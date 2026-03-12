---
title: "Human vs APE - Markdown is for Display. APE is for Execution."
description: "Why getting LLMs to do what you want is a linguistics problem, not a prompting problem - and how a markup language might fix it."
date: 2026-03-12
tags:
  - programming
  - ai-agents
  - agentic-coding
  - language-design
  - devlog
draft: false
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

But when a human says "it's cold in here" to another human, there's a whole universe of shared context - where the speaker and listener are in relation to one another, body language, room temperature, relationship history, social norms - that helps the listener land on the right interpretation. As humans, we're decent at it. Not perfect, but good enough for most situations.

LLMs are an attempt to replicate that human-to-human communication pattern at scale. And by its very nature, the imperfections at human scale can quickly compound when translated to machine execution.

## The Double Drift Problem

When you write instructions for an LLM, semantic drift happens *twice*.

### First, at the **authoring end**.

You have an intent - something you want the agent to do. You encode that intent into text. But text is lossy. Your intent passes through your vocabulary, shaped by your lived experiences, your assumptions about how the reader will interpret things, your implicit mental model of how the task should flow. By the time it's written down, it's already a *translation* of what you meant, not a *transcription*.

### Then, at the **interpretation end**.
The LLM reads your text and builds its own model of what you meant. But it's pattern-matching against its training distribution, not against your intent. It has no access to what you were thinking when you wrote it. It has the locution (the literal words) and it's trying to reconstruct the illocution (the intended force) from that alone.

That’s a locutionary failure on both ends. You fail to encode your intent perfectly into words, and the LLM fails to decode those words back into your intent. By the time the instructions reach the execution phase, the illocutionary force—the "do this or else"—has been diluted into a mere suggestion.

What I think makes this fundamentally harder than the human version: with humans, you can go back and forth. You can clarify. You can read the room. With an LLM executing a workflow, ideally the instructions are fired and forgotten. The agent reads them once and runs.

(At least that's the spooky "singularity" version of it we're all busy building.)

## Where Markdown Falls Down

How do most people write LLM instructions today?

Markdown. System prompts. Scattered files full of prose.

And markdown is great! I love markdown. But **markdown is a display format**. It was designed to describe how things should *look*. Headers, bold text, lists, code blocks - these are visual affordances. They tell a renderer how to present content to a human reader.

When you write LLM workflow instructions in markdown, you're using a display format for an execution task. You're describing *appearance* and hoping the agent infers *behavior*. Every heading, every bullet point, every bold phrase is a locution that the LLM has to decode into an illocution.

`## Step 3: Validate the Output` - is that a suggestion? A requirement? What happens if validation fails? Do I stop? Try again? Skip it? The markdown doesn't say. It *can't* say, because it doesn't have the vocabulary for execution semantics. Markdown has no concept of gates, failure handlers, prerequisites, or scope. We need _more markdown_ (and more prose) for that.

So, we end up compensating: `IMPORTANT: Do not proceed until...` and `NOTE: If this fails, go back to step 2 and...` and `Make sure you...` All of which are illocutionary cues stuffed into a format that has no structural way to enforce them.

The LLM might follow them. It might not. It depends on how much attention it pays to your capitalized "IMPORTANT" versus the structural pull of the next heading versus how it's feeling that day.

And that leaves us in a neverending state of push and pull, with us constantly pushing the LLM to follow our wants and needs, and the whole process pulling us away from whatever-we-were-trying-to-do-in-the-first-place.

## APE

This is the problem I built APE to solve.

APE - Applied Primitive Execution - is an XML markup language designed to treat the LLM as a runtime execution engine. Where markdown describes how things should look, APE describes what things should be.

That's the core distinction. Markdown is a display format. APE is an execution format.

In APE, the document is the workflow. It's self-contained. It declares who does what, what tools to use, when to stop and wait, and what to do on success or failure. Hand it to an agent; it runs.

The whole spec is designed to be boiled down to only the absolute necessary primitives for workflow execution: flow control, input, output, and actions.

When you stop thinking in "prose" and start thinking in "primitives," your instruction set stops being a suggestion and starts being a runtime contract. `<command>` becomes an action with defined success/failure states. `<resource>` becomes an input dependency that must be satisfied. A `<gate>` becomes a conditional flow control element.

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
  <action id="tests-pass"><command ref="all-tests" /></action>
  <gate>
    <criteria ref="tests-pass" />
    <on-fail goto="debug" />
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

I can already hear the mechanical keyboards clicking in protest:
>"XML? In the year of our lord 2026?"

This "LLM" thing we're talking about - it was raised by the XML, molded by it. If we want to talk to our machines, it's helpful to know what language *they* natively speak.

There was actually a study that showed JSON can help, but this was for a slightly different purpose - output format. But regardless, have you ever tried to write a workflow by hand in JSON? JSON - that poor bastard of a data structure (jk, love you JSON, you're doing great!). But just think of all the matching brackets, the lack comments, no mixed content... It's miserable for humans. YAML is better to write but worse to parse unambiguously, and it has no schema language worth using.

But that verbosity is a feature, not a bug. LLMs are essentially massive pattern-matchers; when they see a `<gate>` tag, they aren't just reading a word - they are entering a high-probability state associated with "validation" and "checkpointing." Structure provides semantic anchoring. It gives the model a clear hierarchy to attend to, making it much harder for the "important" instructions to get lost in the noise of the surrounding prose.

## What's Next

APE is at v0.2.2-draft. The spec is written. The schema validates. The authoring guide teaches you how to think in commands and resources. The LLM execution contract tells agents how to interpret and run APE documents.

What I don't have yet are benchmarks. The benchmark suite is built - it tests how workflow instructions in different formats (APE XML, markdown, plain text) perform against real apps with realistic prompts, running in isolated workspaces. But the results are still pending.

That's the next milestone, and it's the one that matters most. The hypothesis that structural intent reduces semantic drift is either going to show up in the numbers or it isn't. I think it will - but I'm building the tools to prove it, not just argue it.

If you've ever stared at a system prompt wondering why the LLM keeps ignoring your very clear instructions, you've felt the illocutionary gap. APE is my attempt to close it - not by writing better prose, but by giving authors a language where the structure *is* the intent.

Markdown describes how things should look. Using Markdown to control an agent is like trying to use a restaurant menu to teach someone how to cook. It looks great, but it lacks the verbs.

Think about the last time you added:

`!!! REALLY SUPER DUPER IMPORTANT (SERIOUSLY) !!!`
to a prompt.

That was you trying to manually increase the illocutionary force of a format (markdown) that doesn't support it. You weren't programming; you were pleading.

APE describes what things should be *and* how they should act. For people who want our machines to do things, the bet is that APE will help us get there faster.
