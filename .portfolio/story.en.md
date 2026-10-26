## Problem
Terms of service and privacy policies run to thousands of words of legal language, so almost nobody reads them. People accept dozens of them without knowing what they give away, and the only help on offer is pasting the text into a chatbot by hand.

## Solution
PlainSight does the reading at the moment of decision. A Manifest V3 extension, written in plain JavaScript, scans each page for policy links by keyword and adds a small icon coloured by risk. Hovering opens a card drawn inside a Shadow DOM, so no site's CSS can break it. A background worker asks my Hono API to analyse the link, with a five-minute cache. The server fetches the policy page, extracts the text with jsdom, and prompts an OpenAI-compatible language model for tagged sections such as the summary, risks and data collected, then parses them with a fallback so the card never shows raw model output. Accounts use better-auth, and a Drizzle database holds analyses, activity, monitored domains and feedback. A Next.js site carries the landing page, the sign-in and dashboard pages, and the launch waitlist.

## Outcome
Pre-launch and built solo: a working extension (v0.2.1), a live API, and a live landing page with a waitlist, in about 9,000 lines across the extension, server and website. The Chrome Web Store launch is next.
