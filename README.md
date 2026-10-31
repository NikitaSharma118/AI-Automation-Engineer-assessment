# PlacidWay Daily Facebook Post: n8n Workflow

One fresh, on-brand Facebook post per day, generated from public PlacidWay pages, checked by guardrails, then published, scheduled, or sent for human approval.

## Architecture

```
Schedule (daily 09:00) ─┐
Manual Run ─────────────┴─> Config ─> Fetch sitemap index ─> Pick Child Sitemap (rotates daily)
  ─> Wait 10s ─> Fetch Child Sitemap ─> Parse & Select (filter, dedupe, pick 1)
  ─> Wait 10s ─> Fetch Page ─> Extract Page (title, summary, body, og:image)
  ─> Build Prompt ─> LLM Generate (Gemini, JSON: hook / lines / cta / hashtags)
  ─> Guardrails Rules (code) ─> LLM Verify (2nd pass) ─> Gate ─> Passed?
        ├─ no  ─> Format Alert ─> Send Alert (email)
        └─ yes ─> Approval Mode?
              ├─ on  ─> Email Reviewer ─> Wait (webhook, 24h) ─> Approved? ─(no/timeout)─> stop
              └─ off ───────────────────────────────────────────────┐
                                    (approved) ─> Prepare Publish ─> Dry Run?
                                                   ├─ yes ─> Simulate ─┐
                                                   └─ no  ─> Facebook Graph API ─┴─> Log Run
Error Trigger ─> Format Alert ─> Send Alert   (any node failure after retries)
```

## Setup

1. Import `placidway_daily_post.workflow.json` (n8n: Workflows → Import from file).
2. Create credentials:
   - **Gemini** (Header Auth): name `x-goog-api-key`, value = your Google AI Studio API key (free tier). Attach to *LLM Generate* and *LLM Verify*.
   - **Facebook** (Header Auth): name `Authorization`, value `Bearer <PAGE_ACCESS_TOKEN>`. Attach to *Publish to Facebook*. Use a long-lived Page token for your **own test Page** (permissions: `pages_manage_posts`, `pages_read_engagement`).
   - **SMTP**: attach to *Email Reviewer* and *Send Alert*.
3. Edit the **Config** node (all settings in one place):

| Key | Meaning |
|---|---|
| `INDEX_URL` | `https://www.placidway.com/sitemap.xml` (sitemap index, listed in robots.txt) |
| `CONTENT_SITEMAPS` | Child sitemaps to use; one is chosen per day by rotation |
| `CRAWL_DELAY_SECONDS` | 10, per robots.txt `Crawl-delay` |
| `URL_INCLUDE` | Optional regex to further filter URLs |
| `FORCE_URL` | Test one specific page |
| `APPROVAL_MODE` | `true` = human approves via email link before posting |
| `DRY_RUN` | `true` = simulate, never call Facebook |
| `SCHEDULE_HOURS_AHEAD` | `>0` = schedule on the Page instead of posting now |
| `FB_PAGE_ID` | Your test Page ID (never a PlacidWay page) |
| `MODEL`, `*_EMAIL` | LLM model and email addresses |

4. Open the workflow settings and set **Error workflow** to this same workflow (the Error Trigger inside it handles alerts).
5. Set a real contact email in the `User-Agent` header of the two fetch nodes.
6. Test: *Manual Run* with `DRY_RUN: true`, then `false` against your test Page, then Activate for the daily schedule.

## Design decisions and trade-offs

- **Extraction: two-level sitemap + plain HTTP + meta tags.** `sitemap.xml` is an index; the workflow picks one child sitemap (articles, packages, news, testimonials, experiences) by day rotation, then a URL from it. 3 requests per run. Cheapest, most polite, no headless browser. `og:title`, `og:description`, `og:image` give the title, summary and image reliably; body text is a tag-stripped fallback. Trade-off: JS-rendered pages would need a headless browser.
- **Politeness:** robots.txt allows these paths for all agents with `Crawl-delay: 10`, so the workflow waits 10s between requests (stricter than the 1 req/s assessment limit). Disallowed paths (`/uploads/`, `/api/`, `/ajax/`…) are never fetched. descriptive User-Agent, only public pages, run once a day.
- **Selection and dedupe:** random choice among unposted URLs of the day's content type. Posted URLs are stored in n8n workflow static data and added only after a real publish. *Limitation:* static data persists only for production (activated) runs, not manual test executions. For a production system I would use an n8n Data Table or Google Sheet.
- **Structured generation:** the LLM returns JSON (hook, lines, CTA, hashtags) and code assembles the post, so format, link placement and length are enforced by code, not trusted to the model.
- **Guardrails, two layers:**
  1. *Rules (deterministic):* ≤500 chars before the link, 2–4 lines, 3–5 hashtags, banned claim words (cure, guarantee, best, safe, painless, permanent, proven…), any price not in the source, any number not found in the source text.
  2. *LLM verifier (semantic):* a separate call acts as a strict compliance reviewer and checks the post against the source for medical claims, promises and unsupported facts.
  Fail = nothing is posted and an alert with the issues and the draft is emailed. Trade-off: no auto-regenerate loop (one retry would be the next improvement) and the rules can false-positive, which is the safe direction.
- **Approval mode:** emails approve/reject links using n8n's resume webhook. Timeout or rejection means no post (fail-closed).
- **Errors:** HTTP nodes retry 3x (publish 2x, to limit duplicate-post risk). Anything still failing triggers the Error Trigger, which emails node name, error and execution link, covering site down, API error and expired Facebook token (error code 190).
- **Logging:** each run logs timestamp, URL, post text, Facebook post ID and status (`published`/`scheduled`/`simulated`) in static data (last 200) plus the n8n execution history.

## Sample output

See `SAMPLE_OUTPUTS.md` (generated by real runs).

## Known limitations / next steps

Persistent store (Data Table/Sheets), regenerate-on-fail loop, image branding/generation, Instagram/LinkedIn cross-post, engagement read-back via Graph `/insights`.

## Final hardening notes

- **Failure alerts work in manual runs too.** Every fetch, parse, LLM and publish node routes its error output to *Format Alert* → *Send Alert*. The Error Trigger remains as a safety net for scheduled/production runs.
- **Blocked and failed runs are logged** (status `blocked` / `error`) in addition to published/scheduled/simulated.
- **Retries:** n8n caps wait-between-tries at 5 s. LLM nodes retry 4x, fetches 3x, publish 2x (low, to limit duplicate-post risk).
- **Run Now (Webhook):** a GET webhook starts a production-mode run, which persists the dedupe store and triggers the Error Trigger. Activate the workflow first; deactivate or delete the node after the demo.
- **Demo switches (Config, VIDEO ONLY, keep `false`):** `DEMO_BAD_POST` injects an unsafe line so the guardrails block the post; `DEMO_FB_FAIL` sends an invalid token (nothing is ever posted). Turn both off afterwards and the workflow behaves normally. No nodes or credentials need editing.
