# Definitely Not Propaganda

**[eagergoose32.github.io/definitelynotpropaganda.github.io](https://eagergoose32.github.io/definitelynotpropaganda.github.io/)**

A counter-propaganda documentation project tracking coordinated influence operations, hasbara networks, and information warfare in real time. Receipts first, analysis second — every claim is sourced and archived.

## What this is

DNP documents recurring patterns in coordinated online influence campaigns and mainstream media failures: coordinated comment surges, "zone flooding" (burying accountability stories under high news velocity), narrative injection, and the selection (not purchase) of institutional alignment. Entries are cross-referenced against primary sources — NPR, AP, Reuters, NY Times, Washington Post, CNN, ProPublica, The Intercept — and screenshots are archived to the Wayback Machine before accounts can be scrubbed.

AI disclosure: I use Claude, an AI model made by Anthropic, to help gather information and draft articles. Claude does not pick the stories or decide the conclusions. AI also gets things wrong, so every claim links to its source for readers to check, and readers can report mistakes through the tip form. The full process is on the [How DNP Works](how-it-works.html) page.

## How to read an entry

Investigations label their claims so you can tell what's proven from what isn't:

- **Unconfirmed** — not officially established. Entries say so up front, before any argument starts.
- **Not Established** — spells out what the evidence does *not* prove, so a connection isn't implied where the record doesn't show one.
- **Secondhand** — a lead resting on someone else's reporting, with the outlet named. Treated as something to confirm, not a finding.
- **What to Pull Next** — the open questions, records requests, and filings that would settle it: pages to archive before they change, FOIA requests to file, and public records anyone can look up. You can check the work or finish it.

When a claim rests on a single source, the entry flags it and adds a dated update note when the reporting changes.

## Features

Listed in the same order as the site's menu.

- **Home** (`index.html`) — opens with the latest investigation and a quick guide to the labels above, then a filterable Documentation Feed (Breaking / Hasbara / Disinfo / Analysis). Its live Breaking section pulls current RSS headlines, clusters them by topic, and ranks them by how many independent outlets are covering each story.
- **How We Work** (`how-it-works.html`) — how the research gets done, how Claude (an AI model) is used and what it does not do, how every claim is verified, what the tags mean, and how corrections work.
- **The Method** (on the homepage) — the site's detection patterns: Follow the Invoice, Silence Is a Tell, Zone Flooding, Selection Not Purchase, Receipts First, Multi-Source Verification, and How We Use AI.
- **News Feed** (`feed.html`) — a live, searchable feed from non-Western and independent outlets (Al Jazeera, TRT World, Anadolu, Middle East Eye, +972, CGTN, Press TV), with US/Western outlets (NPR, NY Times, Washington Post, CNN) shown side by side and flagged so you can compare framing.
- **Operations Archive** (`operations.html`) — long-form investigations into propaganda campaigns and influence operations. Each one is broken into numbered sections that follow the money, map the network, lay out the paper trail, and end with full sources.
- **Blog** (`blog.html`) — editorial posts and dispatches, backed by Firebase Auth/Firestore. Everyone can read and search posts. Readers with a verified email can comment and subscribe to get an email when a new post goes up. Only the configured admin account can publish or edit (see `BLOG_SETUP.md`).
- **Feed Archive** (`feed-archive.html`) — a searchable archive of entries built from the live feeds, auto-tagged by topic, 15 per page.
- **Submit a Tip** (`submit-tip.html`) — an anonymous, verified tip form (with optional image attachments) for reporting coordinated account behavior, narrative patterns, or mistakes on the site; sensitive submissions can go through Signal or Matrix instead.
- **Chatroom** — a Matrix room (via Element) for real-time discussion, relayed through a Cloudflare Worker.
- **My Account** (`profile.html`) — reader account management for the blog/login system.

## Under the hood

- Static site (HTML/CSS/vanilla JS) served via GitHub Pages — no build step.
- A Cloudflare Worker (`WORKER.md`, config in `wrangler.jsonc`) relays Matrix chat messages and fetches RSS feeds server-side from an allowlist of news hosts, which avoids CORS and bot-blocking problems when pulling breaking news.
- Firebase (Auth + Firestore) powers blog accounts, posts, comments, and subscriptions (`BLOG_SETUP.md`). New-post emails are queued in a Firestore `mail` collection for the Firebase Trigger Email extension to send.
- Tip submissions are relayed via [FormSubmit](https://formsubmit.co/).
- Cloudflare Web Analytics runs on every page.
- Every page shares a favicon; the main pages (home, Operations, How We Work, Blog) carry Open Graph/Twitter link preview tags.

## Contributing

Seen a coordinated pattern worth documenting? [Submit a tip](https://eagergoose32.github.io/definitelynotpropaganda.github.io/submit-tip.html) or join the [Matrix chatroom](https://matrix.to/#/#DNP5:matrix.org).
