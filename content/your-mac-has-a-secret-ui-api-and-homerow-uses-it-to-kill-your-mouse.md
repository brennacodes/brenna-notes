---
title: "Your Mac Has a Secret UI API (And Homerow Uses It to Kill Your Mouse)"
description: "Homerow lets you click anything on screen with your keyboard. It works by walking a hidden tree of UI elements that every macOS app exposes through the Accessibility API. Here's how it works, where it breaks down, and why I paid $50 for it anyway."
date: 2026-02-10
lastmod: 2026-02-10
tags:
  - programming
  - macos
  - tools
  - til
draft: false
author: "Brenna"
---

I paid $50 for a macOS app that lets me avoid using my mouse. One-time purchase. No subscription. It's called [Homerow](https://www.homerow.app/), and I would do it again without thinking.

The pitch doesn't sound like much: "click things with your keyboard." But once you're used to it, reaching for the mouse feels like getting up to change the channel on a TV. You *can*. You just... don't want to.

What surprised me wasn't how well it works (it does, mostly). It was *how* it works. Your Mac has this whole hidden tree of every UI element on screen - buttons, links, text fields, menu items - exposed through a public API. Homerow just walks the tree, finds the clickable stuff, and puts labels on it.

I went down the rabbit hole. Here's what I found.

## Your Mac Has a Secret DOM

If you've done any web development, you know the DOM - the tree structure a browser builds from HTML so JavaScript can find and manipulate elements. Every button, every div, every link has a node in the tree.

Your Mac has the same thing, but for the entire operating system. It's called the **Accessibility API**, and it's been around since the early days of macOS. It exists so screen readers like VoiceOver can navigate apps for visually impaired users. Every standard AppKit or SwiftUI component automatically generates a node in this tree.

```
┌─────────────────────────────────────────────────┐
│  Application (AXApplication)                    │
│  ├── Window (AXWindow)                          │
│  │   ├── Toolbar (AXToolbar)                    │
│  │   │   ├── Button "Back"     ← has actions    │
│  │   │   ├── Button "Forward"  ← has actions    │
│  │   │   └── TextField "URL"   ← has actions    │
│  │   ├── ScrollArea (AXScrollArea)              │
│  │   │   ├── Link "Home"       ← has actions    │
│  │   │   ├── Button "Submit"   ← has actions    │
│  │   │   └── StaticText "Hi"   ← NO actions     │
│  │   └── Sheet (AXSheet)                        │
│  │       └── Button "OK"       ← has actions    │
│  └── MenuBar (AXMenuBar)                        │
│      ├── MenuItem "File"       ← has actions    │
│      └── MenuItem "Edit"       ← has actions    │
└─────────────────────────────────────────────────┘
```

This isn't some reverse-engineered hack. It's a [documented Apple framework](https://developer.apple.com/documentation/applicationservices/axuielement). Any app built with standard macOS UI components gets this for free. The framework just generates the tree automatically.

Homerow's big insight? That tree is queryable. It's right there. It just needed someone to put a nice interface on it.

## How Homerow Actually Works

Homerow is closed-source, but it evolved from [Vimac](https://github.com/nchudleigh/vimac) - an open-source project by the same developer. Vimac's code shows us the architecture, and the core hasn't changed. There are three layers.

### Layer 1: Walk the tree, find the clickable things

When you hit the Homerow shortcut, it needs to figure out what's on screen. Fast. Here's the process:

```
┌──────────────────────────────────────────────────────────┐
│  1. Get the frontmost app                                │
│     NSWorkspace.shared.frontmostApplication → PID        │
│                         │                                │
│  2. Create accessibility reference                       │
│     AXUIElementCreateApplication(pid) → root node        │
│                         │                                │
│  3. Recursively traverse children                        │
│     For each element, batch-read:                        │
│     ┌─────────────────────────────────────────────┐      │
│     │  kAXRoleAttribute     → button? link? etc.  │      │
│     │  kAXPositionAttribute → screen coordinates  │      │
│     │  kAXSizeAttribute     → width and height    │      │
│     │  actionsAsStrings()   → what can it DO?     │      │
│     └─────────────────────────────────────────────┘      │
│                         │                                │
│  4. Filter: does it have actions?                        │
│     Yes → gets a hint label                              │
│     No  → ignored                                        │
└──────────────────────────────────────────────────────────┘
```

That filtering step is the key. Homerow doesn't guess what's clickable based on what it looks like. It asks each element: "can you do anything?" A button reports that it supports `AXPress`. A static text label reports nothing. Only elements with actions get labels.

There's a nice optimization for tables - instead of crawling every row (imagine a spreadsheet with 10,000 rows), it reads `kAXVisibleRowsAttribute` to only label what you can actually see on screen.

Different areas get different query strategies too. The main window, the menu bar, system tray icons, notification center - each has its own traversal service. Electron apps and WebViews get special handling because their accessibility trees look different from native apps.

### Layer 2: Draw the labels

Once Homerow has a list of elements and their screen positions, it creates a transparent window floating above everything and drops hint labels at each element's coordinates.

The labels come from the home row keys - that's where the name comes from. Your fingers naturally sit on A, S, D, F, J, K, L. If there are only a few elements on screen, you get single-character hints. More elements? Two-character combos like "ka", "jf", "sd". As you type, non-matching hints disappear in real-time.

### Layer 3: Click it

You've typed your hint. Now Homerow needs to actually perform the action. It has two ways to do this, and which one it uses matters.

**The clean way:** Call `AXUIElementPerformAction(element, kAXPressAction)`. This tells the accessibility system to trigger the element's native action. The app receives it through the same channel as VoiceOver or any other assistive technology. No side effects, no weirdness.

**The fallback:** When `AXPerformAction` doesn't work for a particular element (and sometimes it doesn't), Homerow creates fake mouse events and injects them into the system event stream. It literally fabricates a mouse-down and mouse-up at the element's coordinates. The app thinks a mouse click happened. It didn't. Homerow lied.

This fallback is also how right-clicks (Shift+Enter) and Cmd-clicks (Cmd+Enter) work - synthetic mouse events with modifier flags set.

## The Accessibility Permission (And Why It's a Big Deal)

You know that dialog: "Homerow would like to control this computer using accessibility features"? Here's what's actually behind it.

Without Accessibility permission:
- The tree query APIs return empty results
- Synthetic mouse events get blocked by the system
- Homerow is completely inert

With it:
- Full read access to every element's role, position, size, text, and available actions
- Ability to trigger actions on any element
- Ability to inject synthetic mouse/keyboard events

macOS gates this behind a manual toggle in System Settings because an app with this permission can see everything on screen and interact with anything. That's incredibly powerful for legitimate tools like screen readers, window managers, and yeah, keyboard navigation apps. It's also the exact set of capabilities you'd want if you were building something [less well-intentioned](/the-security-problem-ai-cant-solve-yet) (see: [OpenClaw](/openclaw-deep-dive-architecture-power-and-risk)).

The permission is an all-or-nothing deal. There's no "let this app read my UI but not click things" option. You either trust it fully or not at all.

## Where It Falls Apart

Here's the thing nobody tells you before you buy Homerow: **it's only as good as the app you're using it in.**

The accessibility tree is built by the app, not by Homerow. If the app doesn't populate its tree well, Homerow has nothing to work with. And the variation is... wide.

| App Type | What Homerow Sees |
|----------|-------------------|
| **Native macOS** (AppKit/SwiftUI) | Everything. Every button, field, and link has a node. |
| **Electron apps** (VS Code, Slack, Discord) | Most things. Chrome exposes a tree for web content, but it's often incomplete or weirdly nested. |
| **Custom-rendered** (games, Flutter, some Java apps) | Almost nothing. They draw pixels to a canvas. No standard components = no tree. |
| **Catalyst** (iPad apps on Mac) | Depends entirely on how well the iOS developer implemented accessibility. |

So you'll have this great experience in Safari, Finder, System Settings - everything just works. Then you switch to some Electron app and half the buttons are invisible to Homerow. Or you open a game and Homerow sees one big opaque rectangle. Welcome back, mouse.

Homerow does maintain explicit support for a bunch of popular Electron apps - Chrome, Firefox, Arc, VS Code, Slack, Discord, Notion, Obsidian, Spotify. But "support" means "we've made it work as well as Chrome's accessibility tree allows," which is a different bar than "everything works perfectly."

> [!tip] Debugging tip
> If you're ever wondering why a specific element isn't showing up, press `?` while Homerow is activated. It opens a "Tutor" that shows the accessibility properties of whatever you're focused on. It's like the Elements panel in browser devtools, but for your whole OS.

## The Vimac Story

Homerow didn't appear from nowhere. It's a full rewrite of [Vimac](https://github.com/nchudleigh/vimac), an open-source project (GPL-3.0) by the same developer, Dexter Leng. Vimac was free. Homerow is $49.99.

[HN had opinions about this transition](https://news.ycombinator.com/item?id=34964904). As HN does.

The core architecture is the same. But Homerow added one feature that genuinely changed the experience: a search bar. Vimac was Vimium-style only - labels appear everywhere, you type the hint letters to click. Homerow lets you type the *name* of the thing you want ("Save", "File", "Submit") and filters to matching elements. Product Hunt called it ["Spotlight for the macOS user interface"](https://www.producthunt.com/products/homerow?launch=homerow), and that's actually pretty accurate.

Other additions: the Tutor I mentioned, better Electron support, multi-monitor support, and an improved scroll mode with Vim-style keys (`j`/`k` for lines, `u`/`d` for half-pages, `gg`/`G` for top/bottom).

## Why I Paid For It

I know, I know. $50 for a utility app. But here's my thing: I will gleefully pay a one-time price for software that does one job well and doesn't ask me for money again next month. No subscription. No "free tier with upsells." No "sign in with your Google account to continue." You buy it, it's yours, it works.

That business model is increasingly rare, and I think it's worth supporting when you find it. Homerow does one thing - keyboard-driven clicking - and it does it well enough that I forget my mouse exists for long stretches. Not always. Not in every app. But enough that the $50 was worth it the first week.

And the underlying concept - that your OS has this rich, queryable tree of everything on screen, just waiting for someone to build a nice interface on top of it - is genuinely interesting from an engineering perspective. [Multi.app](https://multi.app/blog/building-a-macos-remote-control-engine) used the same API to build a remote control engine. Window managers use it to manipulate window positions. Automation tools use it to script interactions with apps that don't have scripting interfaces.

Homerow just found a particularly elegant use for infrastructure that was already there.

Happy clicking. Or, well. Happy *not* clicking.

---

## Mentioned in this post

- [Homerow](https://www.homerow.app/) - The $50 keyboard-driven clicking app for macOS
- [Vimac](https://github.com/nchudleigh/vimac) - Homerow's open-source predecessor (same developer, GPL-3.0)
- [Apple AXUIElement Documentation](https://developer.apple.com/documentation/applicationservices/axuielement) - The macOS Accessibility API that makes this possible
- [Multi.app - Building a macOS Remote Control Engine](https://multi.app/blog/building-a-macos-remote-control-engine) - Another project using the same accessibility APIs
- [Vimium](https://vimium.github.io/) - The browser extension that inspired the "hint labels on everything" concept
- [HN: Vimac goes proprietary](https://news.ycombinator.com/item?id=34964904) - The discussion when Vimac became Homerow
- [Product Hunt: Homerow launch](https://www.producthunt.com/products/homerow?launch=homerow) - "Spotlight for the macOS user interface"
