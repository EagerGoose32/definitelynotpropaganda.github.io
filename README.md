# Definitely Not Propaganda

**[https://eagergoose32.github.io/definitelynotpropaganda.github.io/]**

A counter-propaganda documentation project tracking coordinated influence operations, hasbara networks, and information warfare in real time. Receipts first, analysis second — every claim is sourced and archived.

## What this is

DNP documents recurring patterns in coordinated online influence campaigns and mainstream media failures: coordinated comment surges, "zone flooding" (burying accountability stories under high news velocity), narrative injection, and the selection (not purchase) of institutional alignment. Entries are cross-referenced against primary sources — NPR, AP, Reuters, NY Times, Washington Post, CNN, ProPublica, The Intercept — and screenshots are archived to the Wayback Machine before accounts can be scrubbed.

## Features

- **Documentation Feed** (`index.html`, `feed.html`, `feed-archive.html`) — a filterable feed of documented entries (Breaking / Hasbara / Disinfo / Analysis), plus a live "Breaking" section that pulls and clusters current RSS headlines by topic, ranked by how many independent outlets are covering a story.
- **The Method** — a methodology section laying out the site's detection patterns: Follow the Invoice, Silence Is a Tell, Zone Flooding, Selection Not Purchase, Receipts First, and Multi-Source Verification.
- **Operations Archive** (`operations.html`) — an archive of documented propaganda campaigns and political influence operations.
- **Blog** (`blog.html`) — editorial posts and dispatches, backed by Firebase Auth/Firestore with email-verified accounts; only the configured admin account can publish, everyone can read (see `BLOG_SETUP.md`).
- **My Account** (`profile.html`) — reader account management for the blog/login system.
- **Submit a Tip** (`submit-tip.html`) — an anonymous, verified tip submission form (with optional image attachments) for readers to report coordinated account behavior or narrative patterns; sensitive submissions can go through Signal or Matrix instead.
- **Anonymous Chatroom** — a Matrix-based chat room (via Element) for real-time discussion, relayed through a Cloudflare Worker.

## Under the hood

- Static site (HTML/CSS/vanilla JS) served via GitHub Pages — no build step.
- A Cloudflare Worker (`WORKER.md`) proxies Matrix chat messages and server-side RSS feed fetches (avoiding CORS/bot-blocking issues when pulling breaking news).
- Firebase (Auth + Firestore) powers blog accounts and post storage (`BLOG_SETUP.md`).
- Tip submissions are relayed via [FormSubmit](https://formsubmit.co/).

## Contributing

Seen a coordinated pattern worth documenting? [Submit a tip](https://definitelynotpropaganda.github.io/submit-tip.html) or join the [Matrix chatroom](https://matrix.to/#/#DNP5:matrix.org).
