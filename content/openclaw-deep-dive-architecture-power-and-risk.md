---
title: "OpenClaw Deep Dive: Architecture, Power, and Risk"
description: "A comprehensive look at OpenClaw — the always-on AI agent that runs locally, connects to your messaging apps, and executes real actions on your machine. How it works, why people love it, and why security researchers are sounding alarms."
date: 2026-02-09
lastmod: 2026-02-09
tags:
  - programming
  - security
  - ai-agents
  - til
  - tools
draft: false
author: "Brenna"
---

So there's this open-source AI agent called OpenClaw. It runs on your machine as a persistent background process. You talk to it through WhatsApp, Telegram, Signal, Discord, Slack, iMessage, or a web chat. It talks back — and then it *does things*. Shell commands, file operations, browser automation, calendar management, message sending. Not pretend. Actually executed. On your actual computer.

It has over 100K GitHub stars and a growing community of people who use it daily. It's also been called a "security nightmare" by [Cisco](https://www.cisco.com/), [CrowdStrike](https://www.crowdstrike.com/), and [Trend Micro](https://www.trendmicro.com/).

Both of those things are true at the same time. Buckle up.

## What OpenClaw Actually Is

OpenClaw (formerly Clawdbot, then Moltbot — it's had a whole identity journey) was created by [Peter Steinberger](https://steipete.me/). It's MIT-licensed, runs locally, and connects to an LLM — Claude, GPT, DeepSeek, or others — to power an autonomous agent that lives inside your messaging apps.

Here's the thing that separates it from a chatbot: **OpenClaw has tools that interact with your system.** It reads and writes files, executes shell commands, controls browsers, sends messages across platforms, manages calendars. It runs continuously, even when you're not actively chatting with it. You can tell it "remind me about X tomorrow" and it will actually fire tomorrow.

The software is free. LLM API costs run $10-150/month depending on how much you put it to work.

## Architecture: The Gateway and the Agent Loop

### The Gateway

Everything in OpenClaw flows through a single long-running process called the **Gateway**. Think of it as the central nervous system. It:

- Manages all messaging channel connections simultaneously
- Exposes a WebSocket control plane on `ws://127.0.0.1:18789`
- Serializes agent runs per-session to prevent race conditions
- Stores credentials, transcripts, and state in `~/.openclaw/`

Every message from every channel passes through it. Every action the agent takes originates from it. It's the air traffic control tower — nothing lands or takes off without going through it first.

### The Agent Loop

When a message arrives, OpenClaw runs a standard tool-use loop. If you've worked with any agent framework, this'll look familiar:

1. **Validate and resolve session** — figure out who's talking, load their context
2. **Load skills** — inject relevant tool guidance into the system prompt
3. **Call the LLM** — send the conversation plus available tools
4. **Check response** — tool call? Execute it, feed the result back, loop to step 3. Text? Stream it to the user.
5. **Repeat** until resolution or the 600-second timeout

This is the same loop that [Claude Code](https://github.com/anthropics/claude-code) and other agent frameworks use. The difference? OpenClaw's runs *continuously* across conversations, maintaining context and acting proactively. It's not waiting for you to open an app. It's already running.

### The Agent Engine: Pi

The LLM agent loop isn't built from scratch. It's powered by **[Pi](https://github.com/badlogic/pi-mono)** (`@mariozechner/pi-agent-core`), a minimal agent core written by [Mario Zechner](https://marioslab.io/) — who you might know as the creator of the [libGDX](https://libgdx.com/) game framework. Small world.

Pi's design philosophy is extreme minimalism:

- **Four core tools only:** Read, Write, Edit, Bash
- **Shortest system prompt** of any known agent framework
- **Self-extending:** instead of downloading plugins, you ask the agent to write its own extensions
- **Four wire protocols** covering every major LLM provider (OpenAI Completions, OpenAI Responses, Anthropic Messages, Google Generative AI)

One particularly neat feature: cross-provider context handoff. You can switch models mid-session — say, from Claude to GPT — and Pi converts the message formats between providers. Anthropic thinking traces get converted to `<thinking>` tags inside assistant messages for OpenAI. That was designed in from the start. Pretty slick.

## Channel Integrations and Capabilities

OpenClaw connects to messaging platforms through built-in adapters:

| Channel | How it connects |
|---------|----------------|
| [WhatsApp](https://www.whatsapp.com/) | Web protocol (headless browser session) |
| [Telegram](https://telegram.org/) | Bot API |
| [Discord](https://discord.com/) | Bot SDK |
| [Slack](https://slack.com/) | Bot SDK |
| [Signal](https://signal.org/) | Signal protocol |
| iMessage | AppleScript / macOS system integration |
| Web chat | Built-in web server |

The macOS app is a menu-bar companion written in SwiftUI that owns the system permissions (Accessibility, Screen Recording, Microphone) and manages the Gateway via launchd. There are also native iOS and Android apps that pair as remote "nodes," giving the agent access to cameras, screens, and GPS on paired devices.

Yeah. Cameras, screens, and GPS. We'll come back to why that matters.

### Tools and Skills

Tools are organized into groups:

| Group | What it does |
|-------|-------------|
| `group:fs` | Read, write, edit files |
| `group:runtime` | Shell execution, process management |
| `group:web` | Web search, web fetch |
| `group:ui` | Browser control, canvas rendering |
| `group:messaging` | Cross-platform message sending |
| `group:sessions` | Inter-agent communication |
| `group:nodes` | Camera, screen capture, location on paired devices |
| `group:automation` | Cron jobs, scheduled tasks |

**Skills** are reusable knowledge packages — like Claude Code's skills — that provide usage guidance for tools. There's a community marketplace called **ClawHub** with around 4,000 skills. (Foreshadowing: that marketplace becomes relevant later.)

## Why People Use It

Honestly? The productivity gains are tangible and kind of magical when they work. A 2-minute morning brief with weather, calendar, and headlines. Grocery items auto-added to a shared list when someone texts "we need milk." Meeting transcriptions with extracted action items. Task management across Apple Notes, Reminders, Notion, and Trello from a single WhatsApp thread.

The friction is low — you interact through apps you already use. No new interface, no app switching. Text it like you'd text a friend. A friend who happens to have shell access to your computer.

And being local-first appeals to privacy-conscious users. The agent runs on your machine, not a vendor's cloud. For people uncomfortable with hosted AI services, this feels like a better trade-off. Whether it actually *is* a better trade-off... well, keep reading.

## Why Security Researchers Are Alarmed

Here's where things get spicy.

### 1. Unrestricted System Access by Design

The entire point of OpenClaw is giving an AI agent real system permissions. Shell execution, file read/write, browser control, message sending — these are *features*, not bugs. But a single prompt injection or misconfiguration turns every one of those capabilities into attack surface.

The most powerful tools — elevated shell execution, arbitrary commands on paired devices, full browser UI control, cross-platform messaging, device media capture — all require explicit authorization. But the permission model has three layers (tool profiles, allow/deny lists, provider-specific restrictions), and getting it right requires understanding all of them. The default configuration prioritizes convenience over security.

You see where this is going.

### 2. Prompt Injection: The Unsolvable Problem

OpenClaw's own security documentation states it plainly: **"prompt injection is not solved."**

The agent reads emails, web pages, documents, and messages. Any of that content can contain adversarial instructions that trick the LLM into executing unintended actions. This isn't theoretical — it's been demonstrated:

- **[Zenity](https://zenity.io/) researchers** embedded a prompt injection payload in a Google Doc that directed OpenClaw to create a new Telegram bot integration, establishing a backdoor
- **[HiddenLayer](https://www.hiddenlayer.com/) researchers** had OpenClaw summarize a malicious web page that commanded it to download and execute a shell script

Both attacks worked in controlled environments. They worked because the model processes everything in its context — trusted instructions and untrusted content — as the same text. There's no wall between "things you should do" and "things someone sneaked in there." It's all tokens.

### 3. The Skills Supply Chain

Remember the ClawHub marketplace I mentioned? Here's where it gets fun. And by "fun" I mean "alarming":

- **[Snyk](https://snyk.io/)** scanned all ~3,984 ClawHub skills and found **283 (7.1%)** contained flaws exposing credentials — API keys, passwords, even credit card numbers passed through the LLM's context window in plaintext
- **341 malicious skills** were found distributing Atomic Stealer malware via fake prerequisites on macOS
- **Snyk's ToxicSkills study** found 1,467 malicious payloads across the ecosystem, with 36% containing prompt injection

Skills run with Gateway privileges. An npm-installed skill can execute lifecycle scripts during installation. OpenClaw has since integrated [VirusTotal](https://www.virustotal.com/) scanning, but that only catches known malware signatures — prompt injection payloads aren't traditional malware. They're text that looks like instructions.

### 4. Credential Leakage and Exposed Servers

The default session model (`main`) shares one long-lived session across all DMs. Environment variables and API keys loaded in a "private" session were accessible to anyone who could message the bot. Files saved in one session could be retrieved from another.

And in early February 2026? Over **21,000 publicly accessible OpenClaw servers** were found exposed to the internet. Twenty-one thousand. Yikes.

### 5. Cross-Channel Amplification

Because OpenClaw connects to multiple platforms simultaneously and has access to email, calendars, and documents, a compromise in one channel cascades. An attacker who gets the bot to execute one action can use that foothold to access everything the bot is connected to. One bad Slack message could theoretically lead to your calendar, your files, and your email.

### 6. One-Click Remote Code Execution

A bug was discovered that enables one-click RCE via a malicious link. If OpenClaw is directed to visit a crafted URL, the attacker gains arbitrary code execution on the host. No further interaction required.

## The Fundamental Tension

To their credit, OpenClaw does have security mechanisms: tool profiles from `minimal` to `full`, DM pairing, per-peer session isolation, Docker sandboxing, read-only workspace mode, VirusTotal scanning. Their security documentation includes a detailed checklist — start with smallest access, keep the Gateway on loopback, use DM pairing by default, sandbox tools for untrusted input.

> [!warning] The Core Problem
> **The mitigations that make OpenClaw safe also make it less useful.** Sandboxing, read-only mode, minimal tool profiles — these all remove the capabilities that make people want OpenClaw in the first place. Full security requires giving up the features that justify the tool's existence.

This tension isn't unique to OpenClaw. It's the fundamental tension in *every* autonomous agent with real system access. The more an agent can do, the more damage a compromise can cause. The more you restrict it, the less useful it becomes. There's no free lunch here.

OpenClaw is the most visible example because it's open-source, popular, and designed to be maximally capable by default. It makes the trade-off explicit in a way that proprietary, cloud-hosted agents don't — because when the agent runs on your machine with your permissions, the consequences of failure are yours to bear.

## What This Means for Agentic AI

OpenClaw's situation is a preview of where all agentic AI is headed. As agents gain more tools, more system access, and more autonomy, the attack surface grows proportionally. The security model needs to evolve beyond "trust the model to follow instructions" — because prompt injection means you fundamentally cannot trust that the model's actions reflect the user's intent when untrusted content is in the context window.

The honest answer right now? There isn't a clean solution. You can reduce the risk surface, add layers of defense, audit configurations, and isolate untrusted contexts. But as long as the model processes untrusted text alongside trusted instructions in the same context window, the fundamental vulnerability remains.

OpenClaw at least has the virtue of being transparent about this. Their docs say prompt injection isn't solved. Their code is open for audit. Whether that transparency is enough to justify running an always-on agent with shell access on your machine — that's the question each user has to answer for themselves.

No easy answers here. But at least now you know what the question is.

---

## Mentioned in This Post

**People**
- [Peter Steinberger](https://steipete.me/) — Creator of OpenClaw ([GitHub](https://github.com/steipete))
- [Mario Zechner](https://marioslab.io/) — Creator of Pi agent core and libGDX ([GitHub](https://github.com/badlogic))

**Projects and Tools**
- [Pi agent core](https://github.com/badlogic/pi-mono) — Minimal LLM agent framework powering OpenClaw
- [libGDX](https://libgdx.com/) — Open-source game framework by Mario Zechner
- [Claude Code](https://github.com/anthropics/claude-code) — Anthropic's CLI-based AI coding agent
- [VirusTotal](https://www.virustotal.com/) — Malware and URL scanning service by Google

**Security Research**
- [Zenity](https://zenity.io/) — AI security research; demonstrated prompt injection via Google Docs
- [HiddenLayer](https://www.hiddenlayer.com/) — AI security research; demonstrated malicious web page exploit
- [Snyk](https://snyk.io/) — Developer security; audited ClawHub skills for credential exposure
- [Cisco](https://www.cisco.com/) — Flagged OpenClaw as a security concern
- [CrowdStrike](https://www.crowdstrike.com/) — Flagged OpenClaw as a security concern
- [Trend Micro](https://www.trendmicro.com/) — Flagged OpenClaw as a security concern

**Messaging Platforms**
- [WhatsApp](https://www.whatsapp.com/) | [Telegram](https://telegram.org/) | [Signal](https://signal.org/) | [Discord](https://discord.com/) | [Slack](https://slack.com/)
