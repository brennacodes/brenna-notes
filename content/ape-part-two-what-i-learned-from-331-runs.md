---
title: "APE, Part Two: What I Learned from 331 Runs"
description: "A follow-up on APE - where it fits in the landscape, what building it was actually like, and what 331 benchmark runs can and can't tell me about whether it works."
date: 2026-08-17
tags:
  - ape
  - llms
  - ai
  - programming
  - ai-agents
  - agentic-coding
  - language-design
  - benchmarking
  - devlog
  - markup-specification
  - spec-design
  - ai-workflows
draft: false
author: "Brenna"
---

A while back I wrote about [why I built APE](/markdown-is-a-display-format-ape-is-an-execution-format) - the short version being that I hypothesized that getting an LLM to follow a workflow could be less a prompting problem and more a linguistics one, and that markdown, being a display format, has very little vocabulary for execution. I ended that post by admitting I didn't have benchmarks yet, just a hypothesis and a suite that hadn't been run. I said I'd follow up when I had data.

I have data now. Not as much as I want, and not as clean as I'd like (perfectionism, anyone?), but enough to say some things out loud with one big disclaimer: I am not a benchmarking expert, and I am not a PhD in research methodology. I'm just a freakishly curious, somewhat stubborn gal who wanted to see if she could make the machines go "beep, boop" and maybe learn something along the way.

So here’s what I wanted to figure out: where APE actually sits among the other approaches, what it takes to benchmark something like this, what 331 runs can tell you about whether it works, and what they absolutely cannot.

## Where APE actually fits

Before I ran anything, I spent time figuring out whether I'd built something new or just reinvented a wheel with new spokes. It ended up being somewhere in between, so let's dive in to see where APE actually fits.

The adjacent work sits across four categories, and none of them are quite doing what APE does:

- **The LLM as a tool called by an external engine.** Things like the Agentic Workflow Wiki and PayPal's declarative agent DSL. A `Workflow.md` or a compiled JSON intermediate representation defines the decision tree, but the control logic lives in a deterministic orchestrator *outside* the model. The LLM is a subroutine the engine calls.
- **Structuring the prompt, not the workflow.** POML and PromptML fall here. They give you HTML-ish or code-ish ways to organize role, task, and examples. Useful, but they're about how the prompt is composed, not about flow control or what happens when a step fails.
- **Formalizing conversation protocols.** FASTRIC is the closest to APE philosophically - it makes the implicit state machine in a chatbot conversation explicit and treats the document as the execution model. But it's scoped to multi-turn chat flows and oriented toward academic measurement of "procedural conformance," not general agent work.
- **Optimizing inference.** SGLang lives at a completely different layer - forking, parallel generation, constrained decoding. It's about running LLM programs efficiently, not about authoring readable workflows.

There's also Routine, a "structured planning script" framework that reportedly took GPT-4o from 41% to 96% on tool-calling tasks. That number is eye-catching, but it's a planning layer that sits between a plan and an execution engine, not a self-contained document the model runs directly.

In a system where your inputs directly translate to the kinds of outputs you get, APE's whole philosophy was that the document *is* the runtime. No external engine, no orchestrator, no more fighting the system prompt behind the curtain with a sternly worded letter to the LLM gods. You hand the agent an `.ape` file plus the execution contract, and the structure of the document - gates, prerequisites, actor boundaries, failure handlers - is what drives execution.

I couldn’t find anything that occupied quite that same gap.

So, I was facing the great unknown, an open frontier, staring down the wild, wild west. Which sounds great until you actually have to traverse it... unless you have the right tools, enough curiosity, a healthy respect for the terrain, and a borderline unhealthy will to see it through.

## The Productive Struggle

### Designing the test was harder than building the language

I expected writing the spec to be the hard part. Surprise! It wasn't.

A spec alone and some general feels meant nothing if I didn't have actual data to tell me how well it worked. So I had to have some data. Which meant I had to have a proper way to benchmark the thing.

So, I began researching what was out there. The major code benchmarks I looked at - SWE-bench, HumanEval, MBPP - measured **capability**: can the model solve the problem? The task has a known-correct output, the tests pass or they don't, success is binary. SWE-bench hands an agent a real GitHub issue and checks whether the patch passes the repo's test suite. That's a great question. It's just not my question.

My question is a **behavioral compliance** one: can I get the model to *actually* follow instructions, and does the format of those instructions change the answer? An agent that writes the fix first and backfills a test later scores identically on SWE-bench to one that does strict TDD. An agent that runs `npm test` instead of your `bin/test` wrapper scores identically to one that obeyed you. That wasn't going to fly around these parts.

That meant I had to design a way to measure behavioral compliance, not just capability.

### **This led to a big realization early on: I have to measure against tensions, not capabilities.**
If you ask your LLM to change the name of a function, I'm guessing it's going to do it the exact way you anticipated it would 99% of the time. If you ask it to investigate the cause of some bug, and you want to make sure it can faithfully replicate said bug through testing first, you might have a bad time. For the purposes of testing something like APE, the strongest signal shows up where the model’s trained instincts fight your explicit instructions. I ended up cataloguing seven of these, each one pulled from watching Claude get the same thing wrong over and over:

| ID | Tension | What it looks like in the wild |
|----|---------|--------------------------------|
| T1 | Framework recognition beats explicit commands | You say "use `bin/test`," it sees a vitest config and runs `vitest run` |
| T2 | Relevance filtering beats "run ALL" | You say "run all tests," it runs the one file it thinks is relevant |
| T3 | Obvious fix beats test-first discipline | You say "write a failing test first," it writes a placeholder that doesn't reproduce the bug, then just fixes it (this is actually one that I've seen models get much, much better at over time as their harnesses have presumably been tuned to expect and handle it) |
| T4 | "Good enough" beats convention precision | Commit and formatting conventions get followed unreliably because nothing enforces them |
| T5 | Helpfulness beats restraint | You say "don't add comments that restate the code or document that you're documenting things," it comments everything anyway |
| T6 | Completion bias beats iterative loops | You ask for a loop until done, it does a few and stops when it's had enough (this is another that's totally "popped off" in agentic engineering lately, so it's a bit less relevant now) |
| T7 | Solution creativity beats existing architecture | You say "reuse what's there," it writes new code because that's faster than reading and comprehension of the current system and conventions |

### **This meant that the base state during each test, and everything that could affect the outcome had to be carefully controlled.**
SWE-bench can pull issues straight from GitHub because the issue *is* the task. I couldn't reasonably do that because the task is the workflow, and the tensions come from places beyond the codebase itself. Every part of the benchmark had to be carefully designed to isolate the effect of the workflow format:
- the test cases themselves were crafted to look and feel like something any engineer might encounter day-to-day
- each format was carefully tweaked to try to minimize any potential "drift" that could be introduced by translation between formats as much as possible
- the fixture app was meticulously kept in the same state across all runs - each run started from a clean slate, which included ensuring the git log was free from any evidence of previous runs and ensuring that my personal Claude Code settings didn't leak through (while ensuring I could still use them again once the run was complete)

These are just a few reasons why the current design took so long. Each run would uncover an area of what I'd call "leakage" - parts of the system that weren't properly isolated or controlled. Those (_incredibly frustrating_) moments were also arguably where most of the learning happened.

### Some lessons learned

**Using AI to help design a benchmark... for AI.** Direct your pointer fingers and laughter right here, folks. But I'm gonna draw a hard line at throwing stones in glass houses. Like many of my wayward bretheren, its earliest iterations, I had the assistance of AI to help me create the benchmark suite. It was somewhere around the first run where I realized that the benchmark results I was getting went something like this: "Wow! That was pretty good!". My immediate thought was: "Hm...that’s suspiciously non-numerical in nature..." Because it _was_ completely non-numerical in nature. As it turned out, the benchmark suite the AI helped me assemble somehow ended up with a single, solitary grading mechanism at the end of it all, and it was a question the LLM got to ask itself: "Did I do good?". You know how when governments and corporations investigate themselves they find no wrongdoing? I learned that LLMs are basically just like that.

**Sandbox, shmandbox.** Have you ever seen a sea cucumber trying to catch a meal? It's pretty gross and alarming to behold, so I'm gonna share that below. As you can see, they indiscriminately sploot their tentacles out in every direction possible and gather whatever snacks their salty paws can snatch. After working on APE's benchmark suite for a while, my mental model of what our LLMs do is similarly ambitious, chaotic, and desperate. Every single thing it receives at the moment of your request triggers this frantic dance of exploration and assembly that feels like it happens almost instantaneously. With each iteration of the benchmark design, I was uncovering *yet another* place the LLM's greedy tentacles dared to go. For ages, it felt like a game of whack-a-mole (this might partly explain why I wanted to reach for a hammer at times). I realized that without some serious guardrails (and ideally good insight and oversight), everything in both my system and the codebase were basically fair game.

<figure>
  <img src="/images/sea-cucumber-eating.gif" alt="A sea cucumber using its tentacles to shovel food into its mouth" />
  <figcaption>GIF OF SEA CUCUMBER SCOOPING SOME SNACKS. YOU'RE WELCOME.</figcaption>
</figure>

## Where the numbers landed

331 runs collected between mid-March and mid-May, all on `claude-opus-4-6`, across nine scenarios (three bugs, three architectural issues, three new features) on a single Rust CLI fixture. Grading is structural and trace-based - it inspects the tool calls the agent actually made (file edits, `cargo test`, `cargo build`, git commits) and scores whether the right actions clustered in the right order, rather than trusting the agent's narration.

Five conditions have enough runs to talk about. **Completed** runs count only runs that finished inside the time budget. **All runs** includes timed-out runs at partial credit for whatever they managed before the clock ran out.

| Condition | Source | Runs | Timeout rate | Pass (completed) | Pass (all runs) |
|-----------|--------|------|--------------|------------------|-----------------|
| `plain-text` | prompt | 62 | 42% 🔴 | 80% 🟢 | 56% |
| ape | claude-md | 64 | 30% 🟡 | 79% 🟡 | 62% 🟢 |
| `markdown` | claude-md | 63 | 10% | 64% | 60% 🟡 |
| `adhoc-xml` | claude-md | 64 | 0% 🟢 | 50% | 50% |
| `no-workflow` | (none) | 64 | 2% | 48% 🔴 | 47% 🔴 |

A few things I'm comfortable saying:

### **A real workflow helped here.**
The two strongest conditions land near 80% on completed runs, well clear of `markdown` at 64% and the `no-workflow` baseline at 48%. On completed runs it's a near-tie at the top, with `plain-text` nominally ahead (80% to APE's 79%) - more on why that comparison is trickier than it looks in a minute. Either way, in these runs, handing the agent an actual workflow in a richer form came out well ahead of leaving it to its own instincts.

### **Structure alone didn’t help here.**
 `adhoc-xml` - tags used for emphasis and grouping, but no gates, no steps, no failure handlers - performs no better than the `no-workflow` baseline (50% completed and 50% across all runs, versus the baseline's 48% and 47%). In these runs, wrapping instructions in angle brackets bought me nothing. The signal came from the workflow semantics, not the markup alone.

### **The gains show up where agents are undisciplined.**

Here's every phase for all five fully-sampled formats, not just the flattering ones. Three things to keep in mind while reading it:

- These rates cover *completed runs only*. A run that times out partway can't adhere to a workflow it never finished, and it earns no commit or post-commit hits if it never got that far, so folding partial runs in would just punish the phases they never reached. The catch is that the sample sizes differ, and not by a little. The thorough formats time out more, so they finish fewer runs: the `n` for each format is in the column header, and APE (45) and `plain-text` (36) are resting on noticeably fewer completed runs than `adhoc-xml` (64) or the baseline (63).
- Four of the formats are delivered via `CLAUDE.md` and `plain-text` is delivered via the prompt (that confound again).
- In each row, green marks the best score, yellow the runner-up, and red the worst.

| Phase | `plain-text` (prompt, n=36) | APE (claude-md, n=45) | `markdown` (claude-md, n=57) | `adhoc-xml` (claude-md, n=64) | `no-workflow` (n=63) |
|---|---|---|---|---|---|
| Workflow Adherence | 43% | 40% 🔴 | 48% 🟡 | 48% | 50% 🟢 |
| Specification | 97% 🟢 | 80% 🟡 | 67% | 37% | 19% 🔴 |
| Implementation | 97% 🟢 | 94% 🟡 | 92% | 91% | 89% 🔴 |
| Documentation | 87% | 97% 🟡 | 92% | 84% 🔴 | 100% 🟢 |
| Linting | 98% 🟢 | 95% 🟡 | 83% | 66% | 0% 🔴 |
| Testing | 87% 🟢  | 76% 🟡 | 48% | 41% 🔴 | 41% 🔴 |
| Build | 77% 🟡 | 81% 🟢 | 72% | 38% | 31% 🔴 |
| Commit | 81% 🟡 | 87% 🟢 | 40% | 18% | 17% 🔴 |
| Post-Commit | 46% 🔴 | 61% | 92% | 98% 🟡 | 100% 🟢 |
| Failure Recovery | 97% 🟢 | 91% 🟡 | 72% | 55% 🔴 | 81% |

Across five phases - specification, linting, testing, build, commit - the two thorough formats (APE and `plain-text`) take first and second place in every single row, `markdown` lands a clear third, and `adhoc-xml` and the `no-workflow` baseline sit at the bottom. Left alone, the agent jumps straight to implementation, skips the spec, never runs the linter, and commits sloppily. A richer workflow drags those phases back into the run. Implementation and documentation are near-ceiling for everyone (agents do those fine unprompted), so there's little room for any format to pull ahead.

Two phases look pretty bad for APE:

- **Workflow Adherence** (labeled just "Workflow" if you're inspecting the rubric) is at 40% - dead last among all formats. It grades two things about the run as a whole: whether the steps ran in the pipeline order the workflow declares (specification, implementation, documentation, linting, testing, build, commit, post-commit, with legal redirects allowed), and whether the implement/test loop stayed within its cap of three cycles. It's close to all-or-nothing across the entire run, and everyone struggles with it - the five formats here all land between 40% and 50%, so APE is last, but the five formats are clustered closely enough that I wouldn’t make much of the ranking. My suspicion is simpler: it’s just harder for an LLM to adhere to a strict, multi-step workflow than to just implement and commit and declare "all done!"
- **Post-Commit** looks bad for APE (61%) and perfect for `no-workflow` (100%), but that ranking is mostly an artifact. These are anti-pattern checks - they penalize things like running the build or tests *after* the commit instead of before, because we don't want to commit a failing build or a build with warnings. If an agent barely commits at all, it can't trip them, so it passes vacuously. `no-workflow` has the worst commit phase in the whole table (17%), which is exactly why it "wins" post-commit: it never gets far enough to do the wrong thing. APE commits reliably (87%), so it actually has opportunities to slip here.

**Thoroughness costs time.** APE and `plain-text` both run longer (72 and 86 turns, 23 and 27 minutes) than the roughly 12-minute baseline, and they time out more (30% and 42%) under a fixed budget. That's why the completed-run metric flatters them and the all-runs metric docks them.

## What the numbers don't say

**I cannot claim "APE beats `plain-text`."** APE was sampled at scale via a pointer in `CLAUDE.md`; `plain-text` was sampled via the `prompt`. Those aren't interchangeable - a `CLAUDE.md` is itself a markdown file, so "`plain-text` delivered via a markdown file" could be read as just markdown delivery, which is why `plain-text` was run through the prompt instead. The consequence is that moving from APE to `plain-text` changes *two* variables at once: the format and the delivery source. Any difference between them could be either, or both. Format and source are confounded, but I felt it was worth testing both to see what happened, and I'm glad I did.

That said, APE, `markdown`, `adhoc-xml`, and the baseline, were all delivered via `CLAUDE.md` - and within that group, APE leads on both metrics. That's a real result. What the results can't say at this point, is that "APE is the best format, full stop."

**It's one app and one model.** All of the results come from a single Rust CLI fixture on a single model version. Nothing has been shown to generalize to other codebases, languages, task types, or models, mostly because I haven't had the time or money to do so. Format sensitivity could look completely different under different conditions, and as models and their harnesses change, these results will probably move. This is just a point-in-time reading I thought was worth gathering, and worth sharing.

**The grading measures adherence, not quality.** A trace-based rubric can tell you the agent wrote a spec, ran the linter, and committed cleanly. It cannot tell you the code was any good. "Followed the workflow" and "produced excellent work" are different claims, and I only measured the first one.

## Where APE stands

If you're deciding what to actually write your instructions in, here's what I can tell you:

With `no-workflow`, the agent mostly did the work while skipping the discipline - spec, linting, commit hygiene - because those are the parts it's least inclined to do on its own. If you write **`adhoc-xml`** hoping the structure helps, my data says you shouldn’t expect much from structure alone. You should expect baseline behavior with extra typing. **`markdown`** is the sensible middle: fast, familiar, and well ahead of `no-workflow` on completed runs at 64% completed, though it clearly lags behind two other formats.

Among the formats I can compare cleanly through `CLAUDE.md`, APE performed strongest overall, at the cost of longer runs. `plain-text` also performed extremely well, but because it was delivered through the prompt rather than `CLAUDE.md`, I can’t cleanly rank it against APE yet. It also came with longer runs and more timeouts.

Some things to consider for yourself when deciding what to reach for:
- APE is defined in a way to make it easy for LLMs to write for you, but LLMs instinctively reach for `markdown`, so there's a slight difference in the time it takes to implement one over the other
- In this benchmark, APE produced longer runs, but it also outperformed `markdown` on the behaviors I was trying hardest to enforce.
- Writing things in `plain-text` means it's harder to re-use across runs without copy-pasting, and markdown files such as `AGENTS.md` or similar are adopted as standards in the industry. However, you can often point them at other files quite simply so those can be loaded or referenced by your LLMs as instructed.

## What I'd do with more runs

The budget ran out before the experiment did (331 runs is not free). In rough order of how much they'd change what I can claim:

- **An all-via-`prompt` sweep.** This is the big one. The prompt is the only delivery path that works for every format, so running APE, `markdown`, `adhoc-xml`, and `plain-text` all through the prompt would hold the source constant and finally separate the format effect from the delivery effect. Right now the structured formats have only three or four prompt runs apiece, which is noise. This is the experiment that would actually settle the format ranking.
- **A second and third fixture app.** One Rust CLI is not a general claim. Adding apps in other languages and shapes is the only way to find out whether any of this survives contact with a different codebase.
- **A longer time budget.** APE's timeout rate is doing real damage to its all-runs score. Re-running it with more headroom would separate "APE is thorough" from "APE ran out of time," which are currently tangled together in that 30% timeout figure.
- **Other models.** Version tracking went in late, so there's no real cross-model data yet. Format sensitivity is exactly the kind of thing that could differ from one model to the next.

So that's where it stands. APE looks promising on the comparison I can make cleanly, the wins show up precisely where I hoped they would, and the biggest open question - whether the format itself is responsible, independent of how it's delivered - is still open, because I haven't run the sweep that would answer it. The hypothesis held up better than a null result, which is not the same as being proven. I'd rather say that plainly than round it up.

---

## Mentioned in this post

- [The first APE post](/markdown-is-a-display-format-ape-is-an-execution-format) - the origin story: why LLM instruction-following is a linguistics problem and how APE is meant to address it
- [SWE-bench](https://www.swebench.com/) - the capability-focused code benchmark I kept measuring my own design against
- [FASTRIC](https://arxiv.org/pdf/2512.18940) - the prompt specification language closest to APE philosophically, scoped to conversation protocols
- [POML](https://github.com/microsoft/poml) and [PromptML](https://www.promptml.org) - markup and DSL approaches for structuring prompts rather than workflows
- [SGLang](https://github.com/sgl-project/sglang) - a DSL for efficient LLM program execution, operating at the inference layer
