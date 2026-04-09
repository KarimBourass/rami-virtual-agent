# Rami Virtual Agent - Delivery Architecture Decision

## Overview

Rami Virtual Agent needs to **control a browser** - navigate pages, click buttons, fill forms, scrape data, and interact with platforms like LinkedIn, job boards, and CRMs that lack APIs. This document evaluates three delivery architectures, provides benchmarks, and presents a recommended approach.

---

## Option 1 - Web App with Server-Side Browser

The agent runs entirely on the server. When it needs to browse, it launches a headless browser (Playwright/Puppeteer) on the backend.

**How it works:**
User opens the web UI → builds a workflow → workflow engine triggers a "Browse" node → backend launches a headless Chromium instance → navigates, clicks, extracts → returns results to the next node.

**Advantages:**

- Zero installation - users access via URL
- Full control over the environment (browser versions, proxies, stealth config)
- Scales horizontally - add more browser instances on demand
- Runs 24/7 unattended - works even when the user's machine is off
- Full audit trail of every browser action
- Consistent behavior across all users

**Disadvantages:**

- Cannot access pages behind the user's personal login (LinkedIn, Gmail, CRM) without their credentials or OAuth tokens
- Sites with aggressive bot detection (LinkedIn, Indeed) will block server IPs quickly
- High infrastructure cost - each browser instance consumes significant resources
- CAPTCHAs and proxy rotation become operational burdens

### Platform Benchmarks - Web App Competitors

| Platform | Starter Price | Auth Browsing | AI/LLM | Best For |
| --- | --- | --- | --- | --- |
| **n8n** | $24/mo (free self-hosted) | Via plugins | Yes (LangChain, multi-model) | Technical teams, workflow orchestration |
| **Browse AI** | $19/mo | Yes (cookies/credentials) | Yes (scraping AI) | Web scraping, price monitoring |
| **PhantomBuster** | $69/mo | Yes (session cookies) | Yes (GPT-powered) | LinkedIn/social media automation |

---

## Option 2 - Chrome Extension

The agent runs partly on the server (workflow engine, AI) but browser actions execute **inside the user's actual Chrome browser**.

**How it works:**
User installs a Chrome extension → when a workflow needs to browse, the extension receives commands from the backend via WebSocket → it navigates, clicks, reads the DOM in the user's real browser → sends results back → workflow continues.

**Advantages:**

- Runs in the user's authenticated session - accesses LinkedIn, Gmail, any logged-in tool without credentials
- Indistinguishable from a real user - same cookies, IP, and browser fingerprint
- No server-side browser infrastructure needed
- User can watch the agent work in real-time (transparency and trust)

**Disadvantages:**

- Only works when the user's browser is open
- Chrome-only (no Firefox, Safari, or mobile)
- Manifest v3 limits background script capabilities (service workers go idle, WebSocket connections drop)
- Fragile if user navigates away or closes the tab mid-action
- Chrome Web Store review process adds friction to updates (1–3 day review cycle)
- Cannot easily run parallel browser sessions

### Platform Benchmarks - Chrome Extension Competitors

| Platform | Starter Price | Auth Browsing | AI/LLM | Best For |
| --- | --- | --- | --- | --- |
| **Axiom.ai** | $15/mo | Yes (strong) | Yes (multi-LLM) | No-code browser automation |
| **Bardeen** | $10/mo | Yes (local browser) | Yes (ChatGPT-powered) | Lead generation, browser scraping |
| **Harpa AI** | $12/mo | Yes (local browser) | Yes (GPT, Claude, Gemini, DeepSeek) | AI-assisted browsing, content generation |

---

## Option 3 - Desktop App (Electron/Tauri)

A standalone application on the user's machine with its own embedded browser engine, plus the ability to control Chrome via DevTools Protocol.

**How it works:**
User installs the desktop app → app embeds a browser engine for autonomous browsing AND can control Chrome via DevTools Protocol → workflow engine runs partially locally, partially on the server → app communicates with the backend for AI and workflow state.

**Advantages:**

- Runs in the background even when Chrome is closed
- Can access the user's authenticated sessions
- Automates beyond the browser - file system, local apps, clipboard, system notifications
- No Chrome extension sandbox restrictions
- Can run multiple browser sessions in parallel
- Works offline for local processing, syncs when online

**Disadvantages:**

- Highest development and maintenance cost - Windows, Mac, and Linux support
- Significant installation friction - users must download and install
- Requires auto-update mechanism, code signing certificates per OS
- Antivirus false positives common for apps that automate browsers
- Electron apps are heavy (150–300 MB); Tauri is lighter but has a smaller ecosystem
- Enterprise IT departments may block installation

### Platform Benchmarks - Desktop App Competitors

| Platform | Starter Price | Auth Browsing | AI/LLM | Best For |
| --- | --- | --- | --- | --- |
| **UiPath** | ~$25/mo (POC only), ~$100K+/yr enterprise | Yes (strong) | Yes (AI Center, Autopilot) | Enterprise RPA, legacy systems |
| **Power Automate Desktop** | Free / $15/user/mo | Yes (strong) | Yes (Copilot, AI Builder) | Microsoft ecosystem, desktop RPA |
| **Automation Anywhere** | ~$750/mo | Yes (strong) | Yes (AI Agent Studio) | Enterprise-scale RPA |

---

## Comparison Matrix

| Factor | Web App | Chrome Extension | Desktop App |
| --- | --- | --- | --- |
| **Build complexity** | Medium-high | High | Very high |
| **Installation friction** | None (just a URL) | Low (Chrome Web Store) | High (download + install) |
| **Access to user's login sessions** | No - needs credentials | Yes - native | Yes - with workarounds |
| **Works when user is offline** | Yes - 24/7 on server | No - browser must be open | Partial - app must be running |
| **Parallel browsing sessions** | Easy - spin up instances | Hard - one browser | Medium - embedded browser |
| **Multi-OS support** | All (web-based) | Chrome only | Windows + Mac + Linux |
| **Enterprise IT approval** | Easy - just a URL | Medium - extension install | Hard - app install |

---

## Recommended Approach - Phased Delivery

Rather than committing to a single architecture upfront, a phased rollout reduces risk and delivers value incrementally.

### Phase 1 - Web App (Core Platform)

Build the workflow engine, visual builder, AI layer, and all non-browser nodes. For browsing, use server-side headless browsers for **public pages only** - job boards, company websites, public profiles, news sites. This covers roughly 60% of browsing use cases and ships the fastest.

### Phase 2 - Chrome Extension (Authenticated Browsing)

Once users need the agent to access authenticated platforms (LinkedIn, CRM, email), ship the Chrome extension. It connects to the existing backend via WebSocket. The workflow engine stays the same - a new execution mode is added to the "Browse" node: **server-side** (headless) vs **client-side** (extension). Users select the mode per node.

### Phase 3 - Desktop App (If Market Demands)

Only pursue this if enterprise clients specifically request background automation, local file access, or multi-app control. Most users will be well-served by the web app + extension combination.

**Key architectural benefit:** The workflow engine remains the same across all three phases. The "Browse" node gains a `mode` config - `server`, `extension`, or `desktop` - while the data flow model, expressions, and all other components remain unchanged.
