# 🔍 PlainSight

> **Stop skipping. Start understanding.**
>
> A browser extension that detects privacy policies and terms of service on any website, then shows you an AI-powered summary card when you hover over the link. No copy-paste, no law degree, no cloud subscription required.

**[🌐 Live Website](https://plainsight.dev)** &nbsp;|&nbsp; **[⬇️ Add to Chrome](#-try-it)** &nbsp;|&nbsp; **Waitlist:** ~2,800 people &nbsp;|&nbsp; **Status:** Pre-launch

---

> **📌 About this repository:** This is a public project overview and case study — a portfolio piece showing what PlainSight is, how it's built, and why the technical decisions matter. The full source code lives in a private monorepo and is available for review via call or screenshare. [Details →](#-source-code)

---

## 👆 Why This Isn't Just Another AI Wrapper

2026 is drowning in "AI-powered" tools that are really just a text input hitting the OpenAI API. PlainSight is not one of them. Here's what's actually hard about what this does:

**The AI is swappable — the detection and UX are the product.** The extension doesn't just send text to a chatbot. It scans every page you visit, detects policy links via keyword + URL pattern matching (12 keywords, across arbitrary DOM structures), injects analysis icons without breaking the host page's layout, and renders an interactive hover card in an isolated Shadow DOM so no site's CSS can break it. The LLM is one component — you can swap it for OpenAI, Anthropic, DeepSeek, Gemini, a $0 local model via Ollama, or disable it entirely. The detection pipeline, risk grading system, and hover UX don't care which engine is under the hood.

**Structured output, not free text.** The LLM prompt doesn't say "summarize this policy." It uses XML-tagged output (`<summary>`, `<risks>`, `<data_collected>`, `<data_sharing>`, `<user_rights>`, `<key_points>`, `<risk_score>`) parsed by `extractTagContent()` with per-tag type handling — numeric extraction for the risk score, list normalization for array fields. This gives deterministic, parseable results from a non-deterministic model. If the LLM fails to return parseable XML, the system falls back gracefully rather than showing broken UI.

**Three-mode privacy spectrum.** Cloud mode uses the Plainsight server (zero setup, requires an account). Custom mode lets you bring your own API key — policy text goes directly from your browser to your chosen provider and never touches my infrastructure. Or run Ollama locally: fully offline, zero data leaves your device. No other privacy-policy tool offers this range. It's either "send everything to our cloud" or "doesn't exist."

> <!-- IMAGE: app-showcase-hero.png -->
> <!-- CAPTURE: A split-screen composite. Left: a real website (e.g. reddit.com, amazon.com) with a privacy
>      policy link and the PlainSight icon visible next to it. Right: the hover card expanded, showing the
>      grade badge (colored A/B/C pill), risk score, and one section (Data Collected) expanded with 3-4 items.
>      Goal: one image proves the product is real, works on real sites, and delivers on its promise. -->

---

## 🤔 The Problem & The Thesis

**Nobody reads Terms of Service.** Not because they're lazy — because the system is designed to make them unreadable. The average ToS runs 7,000+ words of dense legalese. The average internet user would need **76 workdays per year** to read every policy they encounter. Companies bank on this asymmetry. 90% of users click "Accept" without reading a single word.

I believe **privacy transparency shouldn't require a law degree, a cloud subscription, or blind trust in a company's "we care about your privacy" page.** The internet normalized a world where you sign away your data rights 40 times a day without knowing it. That's not a technical limitation — it's a UX failure. PlainSight is my attempt to fix it.

The bet: if understanding a privacy policy takes 10 seconds instead of 30 minutes, people will actually do it. The product exists to make that bet true.

> <!-- IMAGE: problem-stats.png -->
> <!-- CAPTURE: A clean visual showing the three core stats — "90% skip ToS", "7,000 words avg", "76 workdays/year".
>      Could be a simple text graphic, a screenshot of the landing page's problem-solution section, or a
>      branded stat card. Goal: three numbers that make the problem visceral in one glance. -->

---

## 🧩 Key Decisions & Trade-offs

| Decision | Why I Chose This | Alternatives Considered |
|----------|------------------|-------------------------|
| **Dual AI mode: Cloud + Custom + Local** | Users get frictionless analysis out of the box, but privacy-conscious users can BYO API key (or run Ollama locally) and policy text never touches my server. This resolves the "privacy tool that requires sending your data to a server" paradox. | Cloud-only (privacy risk), Local-only (too slow on consumer hardware for quality models) |
| **Browser extension over standalone app** | Extensions meet users where they already are. The hover-to-reveal UX is only possible as an extension — it injects analysis directly into the page at the decision point. | Standalone web app (friction death — nobody will paste ToS links manually), Mobile app (different problem space) |
| **WXT framework over raw Chrome APIs** | One TypeScript codebase → Chrome MV3, Firefox MV2, Safari MV2 builds. HMR in dev, typed manifest generation, auto-imported browser APIs. | Raw Chrome APIs (3x boilerplate, no cross-browser path), Plasmo (less mature, fewer escape hatches) |
| **Shadow DOM for hover cards** | Injected UI on arbitrary third-party pages is a CSS minefield. Shadow DOM means PlainSight's styles are completely isolated — no style leaks, no specificity wars with whatever CSS the host site uses. | iframe (heavier, worse positioning), Inline DOM (breaks on sites with aggressive CSS resets) |
| **Structured XML-tag LLM prompts over free-text** | The system needs parseable output: a risk score it can display as a number, lists it can render as collapsible sections. XML tags give structure to an unstructured model. If parsing fails, fallback kicks in — the UI never shows raw LLM output. | Free-text summarization (can't extract structured fields), Function calling (adds provider coupling — not all 8 support it equally) |
| **Hono.js over Express** | ~13KB vs ~2MB. First-class TypeScript with type inference for request context. WinterCG-compliant — runs on Node, Deno, Bun, or Cloudflare Workers. | Express (heavier, looser TypeScript), Fastify (overkill for this API surface) |
| **better-auth over custom auth** | Handles email/password, sessions, CSRF, email verification, forgot-password flow. Auth is the most dangerous thing to build from scratch. | Custom JWT (security risk), Auth0/Clerk (vendor lock-in, per-user pricing kills freemium) |
| **Drizzle ORM + libsql (Turso) over PostgreSQL** | Solo-built project that needs to run cheaply. SQLite-compatible with optional cloud replication. Schema-as-TypeScript means DB and app types never drift. | PostgreSQL (needs a server process — overkill for pre-launch), Prisma (heavier, harder to debug) |
| **LLM as engine, not as product** | 8 providers with different API formats, auth schemes, and response shapes — each handled by provider-specific adapters. If a better model launches tomorrow, users can switch via a dropdown. The product is the detection + UX + grading, not the specific LLM. | Hardcoded to one provider (vendor risk), Abstract "AI service" facade (over-abstracted — adapters are simpler and more debuggable) |

---

## 🖼️ Product Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        USER'S BROWSER                             │
│                                                                    │
│  ┌──────────────┐     ┌──────────────────────────────────────┐   │
│  │  Any Website  │     │       PlainSight Extension            │   │
│  │               │     │  ┌────────────────────────────────┐  │   │
│  │  privacy ─────┼────▶│  │  Content Script                 │  │   │
│  │  policy       │     │  │  • DOM scan + MutationObserver  │  │   │
│  │  terms of ◀───┼───  │  │  • 12-keyword link detection    │  │   │
│  │  service      │     │  │  • Icon injection at link site  │  │   │
│  └──────────────┘     │  │  • Shadow DOM hover card render  │  │   │
│                        │  │  • 5-min request cache           │  │   │
│                        │  └──────────┬─────────────────────┘  │   │
│                        │             │                         │   │
│                        │  ┌──────────▼─────────────────────┐  │   │
│                        │  │  Background Service Worker      │  │   │
│                        │  │  • Message router               │  │   │
│                        │  │  • Auth: login/signup/reset     │  │   │
│                        │  │  • Dual-mode analysis dispatch  │  │   │
│                        │  │  • 8-provider adapter layer     │  │   │
│                        │  └──────────┬─────────────────────┘  │   │
│                        │             │                         │   │
│                        │  ┌──────────▼─────────────────────┐  │   │
│                        │  │  Popup UI (React 18)            │  │   │
│                        │  │  • Dashboard + Privacy Health   │  │   │
│                        │  │  • Quick Analyze + Recent Feed  │  │   │
│                        │  │  • Monitored Domains manager    │  │   │
│                        │  │  • 3-tab Settings panel         │  │   │
│                        │  └────────────────────────────────┘  │   │
│                        └──────────────┬───────────────────────┘   │
│                                       │                            │
└───────────────────────────────────────┼────────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                    │
                    ▼                   ▼                    ▼
          ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
          │  CLOUD MODE   │   │ CUSTOM MODE   │   │  LOCAL MODE   │
          │  (Default)    │   │ (BYO API Key) │   │  (Ollama/     │
          │               │   │               │   │   LM Studio)  │
          │  User sends   │   │  Extension    │   │  100% offline │
          │  policy URL   │   │  sends text   │   │  No data      │
          │  to server    │   │  directly to  │   │  leaves device│
          │               │   │  provider     │   │               │
          │  Server       │   │  OpenAI       │   │  Any GGUF     │
          │  fetches +    │   │  Anthropic    │   │  model        │
          │  extracts     │   │  DeepSeek     │   │               │
          │  content      │   │  Gemini       │   │               │
          │               │   │  Groq         │   │               │
          │  DeepSeek LLM │   │  OpenRouter   │   │               │
          │               │   │  HuggingFace  │   │               │
          └──────┬─────────┘   └──────┬────────┘   └──────┬────────┘
                 │                    │                    │
                 ▼                    ▼                    ▼
          ┌──────────────────────────────────────────────────────┐
          │              Structured Analysis Result               │
          │  • Letter grade (A–F) + risk score (0–100)           │
          │  • 7 XML-tagged sections, collapsible in the card    │
          │  • Content-hash cached for change detection          │
          │  • Falls back gracefully if LLM returns unparseable  │
          └──────────────────────────────────────────────────────┘
```

> <!-- IMAGE: architecture-diagram.png -->
> <!-- CAPTURE: A clean Excalidraw or draw.io version of this diagram. Use the PlainSight brand colors
>      (#FF9D00 orange, #E60000 red, #30DF00 green) for the three mode boxes. Show the flow:
>      Browser (left column) → Mode Selection (3 parallel boxes in center) → Result (right column).
>      Add icons for each component (globe for website, puzzle piece for extension, server for cloud,
>      key for custom, house for local). Goal: the 3-mode architecture is the key visual insight
>      that separates this from every other AI tool. -->

---

## 📸 Product Walkthrough

<!-- IMAGE: hover-card-full.png -->
<!-- CAPTURE: A close-up of the PlainSight hover card on a well-known site (e.g. amazon.com/privacy or
     google.com/policies/privacy). Include: the header with PlainSight logo + "AI Privacy Analysis" +
     the grade badge (A/B/C/D/F in a colored pill), the summary text, the risk score bar, one
     collapsible section expanded (Data Collected with 3-4 items visible), and the "Ask anything..."
     input at the bottom. Goal: THE money shot. Every stakeholder scrolls to this. Make it real. -->

### What the User Sees (Step by Step)

| Step | User Action | What Happens Under the Hood |
|------|------------|----------------------------|
| **1. Install** | One click from the extension store. No account required to start browsing. | Extension registers a content script on `<all_urls>`, starts the background service worker, initializes `MutationObserver` for dynamic content |
| **2. Browse** | Visit any website. Policy links automatically get a small PlainSight icon next to them. | Content script scans the DOM for 12 keywords (`privacy`, `policy`, `tos`, `terms`, `conditions`, `legal`, `gdpr`, `ccpa`, etc.) across `<a>` href and text content. Detected links get a 16×16 icon injected via `safeInsertIcon()` with fallback DOM manipulation |
| **3. Hover** | Hover over the icon next to a "Privacy Policy" or "Terms of Service" link. A card appears. | `analyzeUrl()` fires → background script dispatches to the configured AI mode (cloud: 30s timeout, custom: 60s timeout) → result cached for 5 minutes → Shadow DOM hover card renders with a loading spinner, then populates with the structured result |
| **4. Read** | Scan the grade, expand sections, check the risk score. Takes ~10 seconds. | 7 collapsible sections with per-category color coding (red for risks, blue for data collected, purple for data usage, orange for data sharing, green for user rights, indigo for key points). The grade badge and icon color update based on the risk score (green A → red F) |
| **5. Ask** | Type a follow-up question (e.g. "Can they sell my data to advertisers?") | Inline Q&A sends the question + policy context to the configured LLM. Response renders below the input in the card |
| **6. Track** | Open the extension popup to see your Privacy Health Score, recent activity, and monitored domains | Dashboard pulls from `chrome.storage.local` — quick stats, trend indicators, scrollable activity feed (last 50 entries with risk scores), domain monitoring levels (favorite/blocked/watch) |

<!-- IMAGE: extension-popup-dashboard.png -->
<!-- CAPTURE: The full popup window (335px wide) showing the ExtensionDashboard component. Include:
     - Header: PlainSight logo + PRO badge + "Hey, [name]! 👋"
     - 3 quick stat cards (This Week / Saved / Avg Score) with lucide icons
     - The Privacy Health Score card with the gradient bar and trend indicator
     - The "Quick Analyze" input bar with search icon
     Goal: proves there's a full product here, not just the hover card. The dashboard shows this
     isn't a weekend hack — it's a product with state, history, and user-facing depth. -->

<!-- IMAGE: settings-ai-config.png -->
<!-- CAPTURE: The Settings → AI Config tab with the provider dropdown open, showing all options:
     Anthropic, DeepSeek, Gemini, Groq, Hugging Face, LM Studio, Ollama, Open AI, OpenRouter.
     Include the mode selector (Cloud vs Custom toggle), the model selector dropdown, the API key
     input field, and the "Test Connection" button with its success/error state visible.
     Goal: this single screenshot proves "not an AI wrapper" more than any paragraph. 8 providers
     with connection testing = real engineering, not a demo. -->

---

## 🛠️ Tech Stack

### Browser Extension — WXT + React 18

| Layer | Technology | Why |
|-------|------------|-----|
| **Framework** | [WXT](https://wxt.dev) v0.20 | One TypeScript codebase → Chrome MV3, Firefox MV2, Safari MV2 builds. HMR in dev, typed manifest generation, auto-imported browser APIs |
| **Popup UI** | React 18 + TypeScript + Tailwind CSS 3 | Component-based dashboard with Radix UI primitives (DropdownMenu, Select, Switch, Tabs, Dialog) |
| **Content Script** | Vanilla TypeScript (~2,500 lines) | No framework — runs in the host page's DOM and must stay lightweight. Handles: DOM scanning, MutationObserver, Shadow DOM card rendering, 5-position viewport-aware card placement, event propagation control |
| **Style Isolation** | Shadow DOM | Hover cards live in `#plainsight-root` with an attached Shadow DOM. No CSS contamination in either direction. Event propagation is explicitly managed |
| **Storage** | `chrome.storage.local` | Session tokens, AI config, user preferences, activity history (last 50 entries), 5-min analysis cache |
| **Cross-browser** | WXT browser abstraction + `browser-compat.ts` | Normalized API surface. Same source builds for all targets |

### Backend API — Hono.js + Drizzle

| Layer | Technology | Why |
|-------|------------|-----|
| **Server** | [Hono.js](https://hono.dev) v4 | ~13KB, first-class TypeScript, type-safe request context, WinterCG-compliant (runs on Node/Deno/Bun/Workers) |
| **Auth** | [better-auth](https://better-auth.com) v1 | Email/password + sessions + CSRF + email verification. Hono adapter via `auth.handler()` |
| **ORM** | [Drizzle](https://orm.drizzle.team) v0.44 | Type-safe SQL. Schema-as-TypeScript = DB and app types never drift |
| **Database** | libsql (Turso-compatible SQLite) | Free local dev, optional Turso cloud. No server process needed |
| **LLM** | DeepSeek API (pluggable) | Configurable endpoint + model. XML-tag structured prompting with graceful fallback |
| **Content Extraction** | jsdom v26 | Server-side DOM parsing to extract policy text from raw HTML pages |
| **Email** | Resend + Nodemailer | Welcome emails, waitlist confirmations, password reset |
| **Runtime** | tsx | Watch mode for dev with auto-reload; compiles to plain JS for production |

### Landing Page — Next.js 15

| Layer | Technology | Why |
|-------|------------|-----|
| **Framework** | Next.js 15 + TypeScript + Tailwind CSS 4 | SSR for SEO. Same Radix UI primitives as the extension for brand consistency |
| **Deployment** | Vercel | Zero-config. Edge functions proxy waitlist API calls |
| **Brand** | `#FF9D00` → `#E60000` → `#30DF00` gradient system | Warm-to-urgent. Signals "privacy matters" without alarmism |

### Monorepo Orchestration

| Tool | Purpose |
|------|---------|
| **pnpm** v10.18 | Workspaces with shared dependency hoisting |
| **Turborepo** | `turbo run build` builds all 3 apps with correct dependency ordering |
| **3 packages** | `apps/web` (landing) + `apps/extension` (WXT) + `apps/server` (Hono) |

---

## 🧠 AI Provider Architecture

This is the feature that most clearly separates PlainSight from AI wrapper territory. The system has three analysis modes:

### Mode 1: Cloud (Default — zero config)
- Policy URL sent to the Plainsight server
- Server fetches the policy page via jsdom, extracts clean text, sends to the configured LLM
- Structured XML-tag prompt → parsed by `parseLlmResponse()` with per-tag type handling
- Response cached server-side with content hash (SHA of policy text). If the same policy is re-analyzed with a different hash, a `policy_changes` record is created with the risk delta
- **Requires an account. Only the policy URL touches the server — not the user's browsing context.**

### Mode 2: Custom (BYO API Key — your key, your data)
- Policy text extracted client-side from the page's DOM
- Extension constructs a provider-specific request body (each of 8 providers has a different API format, auth scheme, and response shape)
- Sent directly from the browser to the user's chosen API endpoint. API key never touches Plainsight infrastructure.
- Response parsing adapts to each provider's response shape:
  - OpenAI-compatible: `data.choices[0].message.content`
  - Anthropic: `data.content[0].text`
  - Gemini: `data.candidates[0].content.parts[0].text`
  - HuggingFace: `data[0].generated_text`
  - Ollama: `data.message.content`

### Mode 3: Local (Ollama / LM Studio — fully offline)
- Same flow as Custom mode, but the API endpoint is `localhost:11434` (Ollama) or `localhost:1234` (LM Studio)
- No API key required. No internet required after model download
- Works with any GGUF model via LM Studio's OpenAI-compatible endpoint

### Supported Providers (Pre-configured)

| Provider | Auth | API Format | Example Models |
|----------|------|------------|----------------|
| **OpenAI** | `Bearer $KEY` | `/chat/completions` | GPT-5, GPT-4o |
| **Anthropic** | `x-api-key: $KEY` | `/messages` | Claude Sonnet 4.5, Opus 4.1 |
| **DeepSeek** | `Bearer $KEY` | `/chat/completions` | DeepSeek-Chat, DeepSeek-Reasoner |
| **Google Gemini** | `?key=$KEY` (query) | `/models/{m}:generateContent` | Gemini 2.5 Pro, 2.5 Flash |
| **Groq** | `Bearer $KEY` | `/chat/completions` | Llama 3.3 70B (fastest inference) |
| **OpenRouter** | `Bearer $KEY` | `/chat/completions` | Multi-model gateway |
| **HuggingFace** | `Bearer $KEY` | `/{model}` (Inference API) | Llama 3.1, Mistral v0.3 |
| **Ollama** | None | `/api/chat` | Any local model — offline |
| **LM Studio** | None | `/chat/completions` (:1234) | Any GGUF — offline |

Each provider has a **connection test button** in settings — sends a lightweight request using the provider-specific format and reports success/failure before the config is saved.

---

## 🌟 Differentiators

- **Hover-to-reveal, not dashboard-first.** The analysis lives at the policy link, not in a separate app. This is the UX thesis: reduce friction to zero, and people actually read policies.

- **The AI is not the product.** The detection pipeline (12 keywords, MutationObserver, DOM-safe injection), the Shadow DOM hover card, the grading system, and the structured XML-tag output parser all work independently of which LLM is configured. Swap or disable the engine — the product still works.

- **Three-mode privacy spectrum.** Cloud (convenient, zero config), Custom (your API key, your data path), Local (fully offline). No other privacy-policy tool offers this. The user, not the developer, decides where their data goes.

- **8 AI providers, 0 vendor lock-in.** Pre-configured endpoints, models, API formats, and auth schemes for every major provider. Connection test button. Custom endpoint input for any OpenAI-compatible API.

- **Structured LLM output, not free text.** XML-tagged responses mean parseable sections, computable risk scores, and a UI that never shows raw model output to the user. If parsing fails, the fallback layer kicks in — the card never shows garbage.

- **Shadow DOM isolation.** Hover cards are immune to any site's CSS. No style leaks in either direction. Event propagation is explicitly managed. This was non-negotiable — injecting UI into arbitrary pages without isolation is a support nightmare.

- **Content-hash change detection.** Every analyzed policy is hashed. When a monitored domain's policy changes, the `policy_changes` table records the delta. One-shot analysis becomes ongoing monitoring.

- **16 database tables for a pre-launch product.** The schema models users, sessions, analyses, policy changes, monitored domains, user risk thresholds, compliance actions, education progress, and structured feedback with categories. This isn't a prototype — it's designed for the product I want in 12 months.

---

## 📊 Project Metrics

| Metric | Value |
|--------|-------|
| **Waitlist Signups** | ~2,847 (pre-launch, organic) |
| **Extension Version** | v0.2.1 |
| **Total Codebase** | ~8,000 TypeScript/TSX across 3 apps |
| **Extension** | ~4,000 lines (background service worker, content script, React popup, utilities) |
| **Server** | ~1,500 lines (routes, services, controllers, 16-table Drizzle schema) |
| **Landing Page** | ~2,500 lines (9 page sections, 7 shadcn/ui primitives) |
| **AI Providers** | 8 cloud + 2 local, each pre-configured with models, endpoints, and connection testing |
| **Database Tables** | 16 |
| **Browser Targets** | Chrome/Chromium MV3, Firefox MV2, Safari MV2 — single WXT codebase |
| **Content Script** | ~2,500 lines: DOM scanning, MutationObserver, Shadow DOM, hover cards, event management |
| **Request Cache** | 5 minutes (client-side), content-hash-based (server-side) |
| **Pricing** | Free tier (core features forever) + Premium ($5/month) |

---

## 📁 Project Structure

The PlainSight monorepo contains three applications — a browser extension, a backend API server, and a landing page — managed with pnpm workspaces and Turborepo:

```
tos/
├── apps/
│   ├── extension/                # WXT + React 18 browser extension
│   │   └── src/
│   │       ├── entrypoints/
│   │       │   ├── background.ts       # Auth, 8-provider dispatch, message routing, activity tracking
│   │       │   ├── content.ts          # Link detection, Shadow DOM hover cards, MutationObserver
│   │       │   └── popup/              # React 18 popup: Dashboard, Auth, Settings
│   │       ├── types/ai-config.ts      # 8 provider definitions (models, endpoints, API formats)
│   │       └── utils/                  # analysis-service, ai-config-storage, content-fetcher,
│   │                                   # preferences-storage, auth, browser-compat
│   │
│   ├── server/                   # Hono.js + Drizzle + better-auth
│   │   └── src/
│   │       ├── index.ts                # App setup, CORS, route mounting, auth middleware
│   │       ├── db/schema/index.ts      # 16 Drizzle table definitions
│   │       ├── routes/                 # analysis, feedback, waitlist, earlyAccess
│   │       └── services/               # llm.service (XML-tag parsing), analysis.service
│   │
│   └── web/                      # Next.js 15 landing page + waitlist
│       ├── app/                  # App Router: home, privacy-policy, terms-of-service
│       └── components/           # hero, problem-solution, how-it-works, privacy-analysis-card,
│                                  # waitlist, trust-proof, product-hunt-teaser, faq, + ui primitives
│
├── pnpm-workspace.yaml
└── turbo.json
```

---

## 🔒 Source Code

The PlainSight source code lives in a private monorepo. I keep it private because:

- The extension contains proprietary detection heuristics and UX patterns that are competitive advantages pre-launch
- The backend includes infrastructure configuration and API keys management logic
- The waitlist contains ~2,800 real email addresses (hashed, but still sensitive)

**For recruiters and hiring managers:** I'm happy to walk through the full codebase on a call. Book 30 minutes and I'll show you the architecture, the most interesting code paths, the test setup, and the CI pipeline — nothing is off limits. [Email me](mailto:your-email@plainsight.dev) or find me on [LinkedIn](#).

**For fellow developers:** The architecture is described in detail above. The trade-off table, AI provider adapter pattern, Shadow DOM approach, and structured XML-tag prompt engineering are the interesting pieces — and they're all documented here. If you're building something similar, I'm happy to chat. Steal any idea that's useful.

**For potential users:** You don't need the source code to use PlainSight. The extension is free to install, and its behavior is fully transparent — you can inspect what data it sends where from the settings panel. No trust required.

---

## ⬇️ Try It

| Platform | Link | Status |
|----------|------|--------|
| **Landing Page** | [plainsight.dev](https://plainsight.dev) | Live |
| **Chrome Web Store** | *Coming — join the waitlist for launch notification* | Pre-launch |
| **Firefox Add-ons** | *Coming — built and tested, pending store review* | Pre-launch |
| **Safari Extension** | *Planned — WXT build exists, needs Apple review process* | Roadmap |

---

## 🔮 Roadmap

- [ ] **Chrome Web Store launch** — Coordinate with ~2,800 waitlist subscribers for launch day momentum
- [ ] **Product Hunt launch** — Launch page ready, notification system in place
- [ ] **Chat-style follow-up questions** — The "Ask anything..." input exists in the card; backend has scaffolding
- [ ] **Policy change alerts** — `policy_changes` table is populated on content hash mismatch; notification delivery is next
- [ ] **Historical privacy timeline** — Each domain's risk score over time, from existing `userAnalyses` and `policyChanges` tables
- [ ] **Compliance action templates** — Auto-fill GDPR deletion/access/portability requests from the policy's own language
- [ ] **Firefox Add-ons launch** — Build is ready; need to complete the Mozilla review process
- [ ] **Mobile Safari** — WXT supports Safari MV2; need iOS WebExtension validation
- [ ] **Multi-language policy support** — Keyword detection + LLM prompts currently optimized for English

---

## 📝 Credits

Built solo by **Baki**. I design, I ship, I write. I care about internet transparency, software that earns trust instead of assuming it, and building products where the business model aligns with the user's interest — not against it.

- **Icons:** [Lucide](https://lucide.dev)
- **UI Primitives:** [shadcn/ui](https://ui.shadcn.com) + [Radix UI](https://www.radix-ui.com)
- **Extension Framework:** [WXT](https://wxt.dev)
- **Backend:** [Hono.js](https://hono.dev) + [better-auth](https://better-auth.com) + [Drizzle ORM](https://orm.drizzle.team)
- **Database:** [Turso](https://turso.tech) (libsql)

---

## 📬 Connect

- **Website:** [plainsight.dev](https://plainsight.dev)
- **Twitter/X:** [@plainsight](https://x.com/plainsight)
- **Email:** [your-email@plainsight.dev](mailto:your-email@plainsight.dev)
- **Source code review:** Available on request — [book a call](mailto:your-email@plainsight.dev)

---

<p align="center">
  <b>Stop skipping. Start understanding.</b><br>
  <sub>Privacy transparency shouldn't require a law degree, a cloud subscription, or blind trust.</sub>
</p>
