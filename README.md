# ASO Analyzer

> **App owners spend $200–500/hour on ASO consultants for audits that take 2 hours.**
> This workflow delivers a comparable audit in 90 seconds for under $0.05 in API costs.

An end-to-end automation that takes any app from the Google Play Store, scrapes its listing, analyzes its screenshots with AI vision, and emails the user a professional ASO consulting report — written in the voice of "Marcus Chen, Senior ASO Consultant."

Built with n8n, OpenAI (text + vision), Jina AI, Google Sheets, and Gmail. No code beyond a small data-passthrough function.

---

## What it does

A user fills out a 6-field form. 90 seconds later, they have a 1,500-word ASO audit in their inbox covering keyword strategy, title and metadata, description quality, competitor positioning, ratings analysis, visual asset comparison, and a prioritized action plan with health scores out of 100.

```
User submits form  →  Audit arrives in inbox  →  Backup saved to Google Sheets
       (5 sec)              (90 sec)                    (automatic)
```

## Sample output

A real audit Marcus Chen wrote for Spotify (compared against SoundCloud):

```
ASO AUDIT REPORT
App: Spotify
Audited by: Marcus Chen, Senior ASO Consultant
Date: May 1, 2026
---

THE BOTTOM LINE
Spotify's ASO health is currently mediocre, primarily hindered by
keyword strategy and visual clutter. Improvements are necessary to
boost visibility and conversion rates.

KEYWORD STRATEGY
What's working:
- Brand Recognition: Spotify has strong brand recognition...
- Competitor Comparison: Effectively positioning against SoundCloud...

What's broken:
- Limited Keyword Variety: Overly focused on generic terms like "music"
  without long-tail keywords like "music streaming" or "music discovery"
- Missing User Intent: "playlist maker" or "offline music" aren't being utilized
- Lack of Locale Optimization: No keyword diversity for international users

Keywords you're completely missing:
- "Music streaming app" (5,000 searches)
- "Podcast player" (3,600 searches)
- "Offline music player" (2,900 searches)
[...8 more]

Keyword Score: 4/10

[full report continues with title rewrites, description audit,
competitor analysis, ratings breakdown, visual scoring,
3-tier action plan, and final 100-point health score]
```

Full sample reports are in the [`/samples`](./samples) directory.

## Why it exists

ASO consulting is expensive and slow. A senior consultant runs roughly $200–500/hour. A proper audit takes 1.5–3 hours. That's $300–$1,500 for one report, and a turnaround of several days once you factor in scheduling.

Most app developers — especially indie devs, solo founders, and small studios — never get a professional ASO audit. They guess at keywords, copy what competitors are doing, and watch their downloads stall.

This tool brings the cost down to roughly $0.05 per audit and the turnaround down to 90 seconds. The output isn't quite a $500 senior consultant deliverable, but it's defensibly close on the analytical core: it identifies real keyword gaps, flags genuine description weaknesses, and surfaces concrete competitor advantages.

## How it works

```
On form submission  ──▶  Check App Exists (Jina)  ──▶  App Exists?
                                                            │
                                            (false)─────────┤
                                                │           │ (true)
                                                ▼           ▼
                                          Form Ending   Scrape Google Play (Jina)
                                                            │
                                                            ▼
                                                      Attach Screenshots
                                                            │
                                                            ▼
                                                  Analyze image (GPT-4o-mini Vision)
                                                            │
                                                            ▼
                                                  Message a model (GPT-4o-mini)
                                                  "You are Marcus Chen..."
                                                            │
                                                            ▼
                                                  Append row to Google Sheet
                                                            │
                                                            ▼
                                                  Markdown to HTML
                                                            │
                                                            ▼
                                                       Send Gmail
                                                            │
                                                            ▼
                                                       Form Ending
```

Eleven nodes total. Each node has a single responsibility. Failures at any step are isolated to that step (no cascading silent failures — see the error workflow below).

## The interesting parts

A few design decisions worth calling out, because they came from real iteration:

**Anti-hallucination on dates.** GPT-4o-mini was confidently writing "Date: October 2023" in every report because that's near its training cutoff. The fix wasn't more prompting — it was injecting `{{ $now.format('MMMM d, yyyy') }}` directly into the user prompt and adding an explicit instruction to use that exact value. Date-fabrication dropped to zero.

**Two-stage AI for cost control.** A single GPT-4o-mini call could do everything, but separating visual analysis (`Analyze image` → structured JSON) from report writing (`Message a model` → narrative) lets each prompt be tightly scoped. Visual analysis costs ~$0.002. Report writing costs ~$0.03. Total per audit: roughly $0.04 — and either stage can be swapped for a stronger model independently.

**Markdown-to-HTML conversion before Gmail.** Marcus Chen writes in Markdown (`## Heading`, `**bold**`). Sending that to Gmail in HTML mode shows literal asterisks and hashes. A dedicated Markdown node converts it to clean HTML that Gmail renders properly. Small step, dramatic visual difference.

**Binary data passthrough via Code node.** n8n's HTTP Request nodes (used for Jina scraping) strip binary attachments by default. The user's uploaded screenshots get lost between the form trigger and the Analyze image node. A small Code node explicitly re-attaches the binaries from the form trigger onto the current item before the vision step. Took two hours to debug, ten lines of JavaScript to fix.

## Tech stack

| Layer | Tool | What it does |
|---|---|---|
| Workflow engine | n8n (cloud) | Orchestration, form, sheets, gmail |
| App existence check | Jina AI Reader | Validates app is on Google Play |
| Scraping | Jina AI Reader | Extracts Google Play listing as Markdown |
| Text generation | OpenAI GPT-4o-mini | Marcus Chen's audit report |
| Vision | OpenAI GPT-4o-mini Vision | Screenshot analysis with structured JSON output |
| Storage | Google Sheets | Audit log + raw report archive |
| Delivery | Gmail (OAuth2) | HTML-formatted email to user |

## Cost per audit

| Step | API call | Approx cost |
|---|---|---|
| Check app exists | Jina Reader | Free tier |
| Scrape Google Play | Jina Reader | Free tier |
| Analyze screenshot | GPT-4o-mini Vision | ~$0.002 |
| Generate report | GPT-4o-mini | ~$0.03 |
| Sheets + Gmail | Google APIs | Free |
| **Total** | | **~$0.04** |

For a $5 OpenAI credit, that's ~125 audits.

## Setup

### Prerequisites
- An n8n instance (cloud or self-hosted)
- OpenAI API key with credits
- Google account with Sheets and Gmail enabled
- Jina AI account (free tier is fine)

### Steps
1. Import the workflow JSON from [`/workflow`](./workflow) into n8n
2. Create credentials for: OpenAI, Google Sheets, Gmail (OAuth2)
3. Create a Google Sheet titled "ASO Reports" with these columns: `App Name`, `Category`, `Competitor`, `Date`, `ASO Report`
4. Update the `Append row in sheet` node to point to your sheet
5. Activate the workflow
6. Open the form URL from the trigger node and submit a test

The full setup guide with screenshots is in [`/docs/SETUP.md`](./docs/SETUP.md).

## Roadmap

What's built:
- [x] Form-triggered workflow with validation
- [x] Google Play scraping via Jina
- [x] Marcus Chen text audit (GPT-4o-mini)
- [x] Screenshot upload + vision analysis
- [x] Visual scoring integrated into final report
- [x] Markdown → HTML email delivery
- [x] Date hallucination fix
- [x] Google Sheets archive
- [x] Competitor screenshot analysis (side-by-side visual comparison)

What's next:
- [ ] Anonymized AI input ("App A" vs "App B") to remove brand bias from vision scoring
- [ ] Error workflow that captures every failure (Jina timeout, scrape fail, OpenAI rate limit, Gmail bounce) into a separate sheet
- [ ] Per-run metrics logging (latency per node, token usage, total cost, success/fail)
- [ ] Versioned prompts in repo (Marcus Chen v1, v2, v3 with changelog)
- [ ] PDF export option for users who want a downloadable deliverable
- [ ] Self-hosted image comparison rendering (move from text-only email to visual side-by-side comparison)

## Known limitations

A few things this tool does NOT do well, in the spirit of honesty:

- **Brand bias in vision scoring.** GPT-4o-mini sometimes scores recognizable brands more generously than indie apps with objectively better screenshots. The roadmap item on anonymization addresses this.
- **One screenshot at a time.** The current vision step analyzes a single screenshot per app. Real ASO is about the whole screenshot sequence (storytelling across slides). Multi-image analysis is on the roadmap.
- **Hallucinated screenshot references.** When given one screenshot, the AI sometimes writes "Screenshot 3" or "Screenshot 4" in the analysis — it pattern-matches to what an audit "should look like." Strict prompt instructions reduce this but don't eliminate it.
- **No multi-language support.** Reports are English-only.
- **Free-tier rate limits.** Jina's free tier handles testing fine, but a high-traffic deployment would need paid tiers.

## About this project

I'm Muhammad Abubakar — I built this while learning n8n. It started as a "let me see if I can wire up Gmail" exercise and grew into something I'd actually pay $5 for if I were launching an app.

Most of the lessons came from things breaking:
- Gmail showed raw markdown asterisks until I added the Markdown-to-HTML conversion
- The AI invented dates from 2023 until I forced today's date into the prompt
- Vision analysis returned `binary not found` errors until I figured out that HTTP Request nodes drop attachments
- Marcus Chen wrote with em-dashes everywhere until I prompted explicitly for em-dash-free output

Every node in this workflow exists because something failed and I had to add it. That's the most honest summary of the project.

## License

MIT — do whatever you want with this. If you use it commercially and it makes you money, a credit is appreciated but not required.

## Contact

- GitHub: Muhammad-Abu-Bakar
- Email: abubakarg28@gmail.com
- LinkedIn: www.linkedin.com/in/muhammad-abubakar-b37967115
