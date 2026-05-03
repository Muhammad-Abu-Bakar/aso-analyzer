# ASO Analyzer

> **App owners spend $200–500/hour on ASO consultants for audits that take 2 hours.**
> This workflow delivers a comparable audit in 90 seconds for under $0.05 in API costs.

An end-to-end automation that takes any app from the Google Play Store, scrapes its listing, validates and analyzes its screenshots with AI vision (both yours and a competitor's), and emails the user a professional ASO consulting report — written in the voice of "Marcus Chen, Senior ASO Consultant."

Built with n8n, OpenAI (text + vision), Jina AI, Google Sheets, and Gmail. No code beyond a small data-passthrough function.

---

## What it does

A user fills out a form (app name, category, competitor, email, plus screenshots from their app and the competitor's app). 90 seconds later, they have a 1,500-word ASO audit in their inbox covering keyword strategy, title and metadata, description quality, competitor positioning, ratings analysis, side-by-side visual asset comparison, and a prioritized action plan with health scores out of 100.

```
User submits form  →  Audit arrives in inbox  →  Backup saved to Google Sheets
       (5 sec)              (90 sec)                    (automatic)
```

## Sample output

A real audit Marcus Chen wrote for YouTube (compared against SoundCloud):

```
ASO AUDIT REPORT
App: YouTube
Audited by: Marcus Chen, Senior ASO Consultant
Date: May 3, 2026
---

THE BOTTOM LINE
YouTube's current ASO health is mediocre, primarily hindered by text
visibility and organization in visual assets. Your app catches the
eye but struggles to communicate its value effectively, leading to
potential loss of user engagement.

KEYWORD STRATEGY
What's working:
- Brand Recognition: The app name "YouTube" alone carries significant
  weight due to its established brand presence
- Targeted Keywords: Effective targeting of "music video" trends

What's broken:
- Lack of Long-tail Keywords: Missing niche keywords like "live music
  streaming" or "music video editor"
- Absence of Localization: No evidence of multi-language keyword targeting
- Inconsistent Keyword Density: "streaming" appears sparse in metadata

[...]

VISUAL ASSETS COMPARISON
Your App's Visual Score Breakdown:
- Hook Strength: 15/25
- Visual Hierarchy: 20/25
- Feature Communication: 18/25
- Brand Consistency: 22/25
- Total Visual Score: 75/100

Competitor's Visual Score Breakdown:
- Hook Strength: 18/25
- Visual Hierarchy: 20/25
- Feature Communication: 22/25
- Brand Consistency: 21/25
- Total Visual Score: 81/100

Head-to-head verdict:
Competitor SoundCloud is winning by 6 points. The biggest gaps appear
in hook strength and feature communication, where SoundCloud effectively
utilizes striking visuals to guide users.

[full report continues with action plan and 100-point health score]
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
                                                  Attach Screenshots 1
                                                  (re-attach binaries)
                                                            │
                                                            ▼
                                                  Verify App Match
                                                  (GPT-4o-mini Vision)
                                                            │
                                                            ▼
                                                       Match?
                                                            │
                                            (false)─────────┤
                                                │           │ (true)
                                                ▼           ▼
                                          Form Ending   Attach Screenshots 3
                                          (validation    (re-attach binaries)
                                           failed)              │
                                                                ▼
                                                  Analyze Your App
                                                  (GPT-4o-mini Vision)
                                                            │
                                                            ▼
                                                  Attach Screenshots 2
                                                  (re-attach binaries)
                                                            │
                                                            ▼
                                                  Analyze Competitor
                                                  (GPT-4o-mini Vision)
                                                            │
                                                            ▼
                                                  Message a model
                                                  "You are Marcus Chen..."
                                                  (text + both visual JSONs)
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
                                                       (success)
```

Fourteen nodes total. Each node has a single responsibility. Failures at any step are isolated to that step.

The three "Attach Screenshots" Code nodes exist because n8n's OpenAI vision nodes consume binary attachments but don't pass them downstream. Each small Code node re-attaches the original screenshots from the form trigger so the next vision call has access to them. This is a known n8n pattern, not over-engineering — it's the cleanest way to chain multiple vision analyses on the same input.

## The interesting parts

A few design decisions worth calling out, because they came from real iteration:

**Anti-hallucination on dates.** GPT-4o-mini was confidently writing "Date: October 2023" in every report because that's near its training cutoff. The fix wasn't more prompting — it was injecting `{{ $now.format('MMMM d, yyyy') }}` directly into the user prompt and adding an explicit instruction to use that exact value. Date-fabrication dropped to zero.

**Two-stage AI for cost control.** A single GPT-4o-mini call could do everything, but separating visual analysis (`Analyze image` → structured JSON) from report writing (`Message a model` → narrative) lets each prompt be tightly scoped. Visual analysis costs ~$0.002 per call. Report writing costs ~$0.03. Total per audit: roughly $0.045 — and either stage can be swapped for a stronger model independently.

**Screenshot validation before analysis.** Without validation, a user uploading the wrong screenshot for the named app would get a confidently-generated but worthless audit. A separate vision call verifies the uploaded screenshot actually matches the named app before the main analysis runs. Mismatches branch to an error form ending instead of wasting API calls and confusing the user.

**Markdown-to-HTML conversion before Gmail.** Marcus Chen writes in Markdown (`## Heading`, `**bold**`). Sending that to Gmail in HTML mode shows literal asterisks and hashes. A dedicated Markdown node converts it to clean HTML that Gmail renders properly. Small step, dramatic visual difference.

**Binary data passthrough via Code nodes.** n8n's HTTP Request and OpenAI vision nodes both strip binary attachments by default. The user's uploaded screenshots get lost between nodes that don't preserve binary data. Three small Code nodes explicitly re-attach the binaries from the form trigger onto the current item before each vision step. Took two hours to debug, ten lines of JavaScript to fix.

## Tech stack

| Layer | Tool | What it does |
|---|---|---|
| Workflow engine | n8n (cloud) | Orchestration, form, sheets, gmail |
| App existence check | Jina AI Reader | Validates app is on Google Play |
| Scraping | Jina AI Reader | Extracts Google Play listing as Markdown |
| Screenshot validation | OpenAI GPT-4o-mini Vision | Verifies upload matches named app |
| Visual analysis | OpenAI GPT-4o-mini Vision | Scores screenshots, returns structured JSON |
| Text generation | OpenAI GPT-4o-mini | Marcus Chen's audit report |
| Storage | Google Sheets | Audit log + raw report archive |
| Delivery | Gmail (OAuth2) | HTML-formatted email to user |

## Cost per audit

| Step | API call | Approx cost |
|---|---|---|
| Check app exists | Jina Reader | Free tier |
| Scrape Google Play | Jina Reader | Free tier |
| Verify screenshot match | GPT-4o-mini Vision | ~$0.002 |
| Analyze your app's screenshot | GPT-4o-mini Vision | ~$0.002 |
| Analyze competitor's screenshot | GPT-4o-mini Vision | ~$0.002 |
| Generate report | GPT-4o-mini | ~$0.03 |
| Sheets + Gmail | Google APIs | Free |
| **Total** | | **~$0.045** |

For a $5 OpenAI credit, that's ~110 audits.

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
6. Open the form URL from the trigger node and submit a test (you'll need a real Google Play app name plus at least one screenshot from your app and one from a competitor)

## Roadmap

**What's built:**
- [x] Form-triggered workflow with validation
- [x] Google Play scraping via Jina
- [x] Marcus Chen text audit (GPT-4o-mini)
- [x] Screenshot upload + vision analysis
- [x] Visual scoring integrated into final report
- [x] Markdown → HTML email delivery
- [x] Date hallucination fix
- [x] Google Sheets archive
- [x] Competitor screenshot analysis (side-by-side visual comparison)
- [x] AI-powered screenshot validation (rejects mismatched uploads before analysis)

**What's next:**
- [ ] FlutterFlow mobile app frontend
- [ ] Anonymized AI input ("App A" vs "App B") to remove brand bias from vision scoring
- [ ] Reviews scraper for deeper Ratings & Reviews analysis
- [ ] Keyword research API integration for real search volumes
- [ ] Multi-image analysis (full screenshot sequence, not just one per app)
- [ ] Error workflow capturing failures (Jina timeout, scrape fail, OpenAI rate limit, Gmail bounce) into a separate sheet
- [ ] Per-run metrics logging (latency per node, token usage, total cost, success/fail)
- [ ] Versioned prompts in repo (Marcus Chen v1, v2, v3 with changelog)
- [ ] PDF export option for users who want a downloadable deliverable
- [ ] Self-hosted image comparison rendering (visual side-by-side instead of text-only)

## Known limitations

A few things this tool does NOT do well, in the spirit of honesty:

- **Brand bias in vision scoring.** GPT-4o-mini scores well-known apps more generously than indie apps with objectively better screenshots. The roadmap item on anonymized "App A vs App B" prompts addresses this by hiding brand identity until after scoring is complete.

- **Shallow reviews analysis.** Jina's scrape returns the Play Store listing page summary but not the full review pool. The Ratings & Reviews section partially infers from limited data. A dedicated reviews scraper is on the roadmap.

- **Keyword search volumes are AI estimates.** GPT-4o-mini doesn't have access to real keyword research databases. Volume numbers like "5K monthly searches" are pattern-matched guesses, not verified data. Integrating an ASO keyword API (AppFollow, App Annie) is on the roadmap.

- **One screenshot per app.** The current workflow analyzes a single screenshot per side. Real ASO is about the full sequence (storytelling across all slides). Multi-image analysis is on the roadmap.

- **Hallucinated screenshot references.** When given one screenshot, the AI sometimes writes "Screenshot 3" or "Screenshot 4" in the analysis — it pattern-matches to what an audit "should look like." Strict prompt instructions reduce this but don't eliminate it.

- **Validation false positives.** The screenshot match check can occasionally reject legitimate uploads of indie apps the AI doesn't recognize. The prompt errs on the side of leniency, but it's not perfect.

- **No multi-language support.** Reports are English-only.

- **Free-tier rate limits.** Jina's free tier handles testing fine, but a high-traffic deployment would need paid tiers.

## About this project

I'm Muhammad Abubakar — I built this while learning n8n. It started as a "let me see if I can wire up Gmail" exercise and grew into something I'd actually pay $5 for if I were launching an app.

Most of the lessons came from things breaking:
- Gmail showed raw markdown asterisks until I added the Markdown-to-HTML conversion
- The AI invented dates from 2023 until I forced today's date into the prompt
- Vision analysis returned `binary not found` errors until I figured out that HTTP Request and OpenAI vision nodes both drop binary attachments
- The first competitor analysis report had duplicated visual sections because the prompt template still had old single-app instructions alongside the new comparison logic
- Adding screenshot validation broke the existing chain because it consumed binaries without re-attaching them — needed a third Code node

Every node in this workflow exists because something failed and I had to add it. That's the most honest summary of the project.

## License

MIT — do whatever you want with this. If you use it commercially and it makes you money, a credit is appreciated but not required.

## Contact

- GitHub: [Muhammad-Abu-Bakar](https://github.com/Muhammad-Abu-Bakar)
- Email: abubakarg28@gmail.com
- LinkedIn: [muhammad-abubakar](https://www.linkedin.com/in/muhammad-abubakar-b37967115)
