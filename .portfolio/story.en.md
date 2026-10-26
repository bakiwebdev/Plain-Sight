## Problem
Terms of service and privacy policies run to thousands of words of legal language, so almost nobody reads them. People agree to dozens of them a day without knowing what they give away, and the only help on offer is copying text into a chatbot by hand.

## Solution
PlainSight is a browser extension that does the reading at the moment of decision. A content script finds policy links on any page and adds a small icon beside them. Hovering the icon opens a card, drawn inside a Shadow DOM so no site's CSS can break it, with a letter grade, a risk score and collapsible sections for what data is collected, shared and what rights you have. The AI is an interchangeable engine behind structured, parsed output: the cloud server, your own API key for one of eight providers, or a local model through Ollama, so the user chooses where their data goes. A Hono and Drizzle backend with better-auth handles accounts and caching, and records a change whenever a policy's content hash changes. A Next.js landing page collects the waitlist. The whole product lives in one Turborepo monorepo.

## Outcome
Built solo and pre-launch: extension v0.2.1, about 8,000 lines of TypeScript across the extension, server and landing page, and about 2,800 people on the waitlist. The Chrome Web Store launch is next.
