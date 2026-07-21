# AI Competitor News Monitor & Weekly Digest

An n8n workflow that tracks FocusKPI's named competitors, summarises the week's coverage with an LLM, and posts a digest to Slack every Monday at 9 AM.

Built as a take-home assignment, July 2026. Chandra (Bitu) Darapaneni.

---

## What it does

Every Monday at 9 AM it pulls news about five specific competitors — Fractal Analytics, Tredence, Quantiphi, Tiger Analytics, LatentView — plus the broader enterprise-AI market, filters to the trailing seven days, dedupes repeated coverage, and asks Llama 3.3 70B to produce 4–5 takeaways written for leadership. Each takeaway carries a "Why it matters" line and a citation. The digest lands in `#competitor-news`.

Typical run: 16 articles analysed, ~4 seconds end to end.

## Pipeline

```
Schedule Trigger (Mon 09:00, America/New_York)
        │
        ├──► HTTP Request — competitor feed (5 rivals by name)
        └──► HTTP Request — industry feed (enterprise AI / analytics)
                    │
                 Merge (append)
                    │
        Code — Parse & Prepare
        · regex-parse RSS XML
        · filter to trailing 7 days
        · dedupe near-identical headlines
        · cap 8 items per feed
        · emit numbered list + prompt
                    │
        LLM — Llama 3.3 70B via Groq (temp 0.3)
        · analyst framing, cites [n], never URLs
                    │
        Code — Format Digest
        · resolve [n] → real URLs
        · normalise bold to Slack mrkdwn
                    │
        Slack — #competitor-news
```

## Design decisions

**Competitors by name, not "AI news."** The brief said *competitor* news, so the first step was identifying who FocusKPI actually competes with rather than pointing at a generic tech feed. A digest about the AI industry is a newsletter; a digest about your rivals is competitive intelligence.

**Google News RSS over company blogs.** Consultancies publish irregularly and several have no usable feed. A News query is stable, needs no auth, and captures third-party coverage — which is where funding, IPO, and executive-hire news actually breaks. Adding a competitor is a one-string edit.

**Two feeds kept separate.** Each article carries a category label so the model can distinguish "a rival did something" from "the market moved."

**Filter, dedupe, and cap before the LLM.** Aggregators repeat the same story across outlets. Deduping in code controls token spend and stops one syndicated story from crowding out four real ones.

**The model cannot produce URLs.** It receives a numbered list of headlines and may cite only `[n]`. A downstream Code node maps cited numbers back to the real links captured during parsing. A fabricated citation cannot render as a link because the model has no link to fabricate — the failure mode is designed out rather than discouraged by prompt.

**Feeds fail independently.** Both HTTP nodes continue on error, so one dead source degrades the digest instead of killing the run.

## Prompt design

- Role framing: competitive-intelligence analyst writing for leadership, so the output is implications rather than summaries
- Enforced structure: bold headline, then a mandatory "Why it matters" line — without it the model returns a headline list
- Explicit priorities: funding, acquisitions, launches, client wins, market shifts; ignore stock-price noise
- Defined empty state: a quiet week returns one fixed sentence rather than manufacturing significance
- Slack mrkdwn constraints, with a normalisation pass in code as a fallback

## Things that broke, and why

**Slack rendered literal asterisks.** I wrote the formatter for standard Markdown; Slack uses mrkdwn — `*bold*`, links as `<url|label>`. Fixed the prompt rules and added a normalisation pass so output doesn't depend on the model getting syntax right.

**Bullet character collided with the bold delimiter.** The model emitted `* *Headline*` and Slack paired the asterisks unexpectedly, rendering italic. Switched the bullet to `•`.

**Citation format drifted.** After adding a rule to merge duplicate coverage, the model started citing `[1, 2, 4]`. My resolver matched only single-number brackets, so the Sources list silently collapsed while the body cited six. Fixed on both sides: the prompt specifies separate brackets, and the parser now accepts either form. An LLM's output format is an interface that will drift, so the consumer should be tolerant by default.

## Cost and scale

| Measure | Value |
|---|---|
| Articles per run | 16 (capped 8/feed, post-dedupe) |
| Tokens per run | ~1,400 in / ~450 out |
| Runs per year | 52 |
| Annual LLM cost | under $0.10 at current Groq rates |
| Run time | 3.8–4.4 seconds |

Article volume is capped before the LLM, so cost stays flat as feeds get noisier rather than scaling with them.

## Known limitations

- **Deduplication is lexical.** Near-identical headlines are caught; the same story reworded across outlets is not. Embedding-based clustering would fix this.
- **No memory between runs.** A story running two weeks appears twice. Persisting seen-article IDs would let the digest say "developing."
- **Single news provider.** Fine for a weekly internal brief; production would add a second source and alert on empty results.
- **No feedback loop.** Slack reactions as a signal for what leadership finds useful is the natural v2.

## Running it yourself

1. Import `workflow.json` into n8n (⋯ → Import from File)
2. Add a Groq API credential to the LLM node
3. Add a Slack credential (bot token with `chat:write`) to the output node
4. Create `#competitor-news` in your workspace
5. Set the workflow timezone under ⋯ → Settings
6. Publish

No credentials are included in the export — only credential IDs, which are meaningless outside the original account.

## On AI assistance

I used Claude (Opus 4.8) as a design partner: pressure-testing the feed strategy, reviewing error handling, and accelerating the RSS-parsing JavaScript. I built, configured, credentialed, and debugged every node myself. The three problems above were found by running the workflow and reading the output.

---

Chandra Narayana Darapaneni .
