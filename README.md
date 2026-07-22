# AI Competitor News Monitor & Weekly Digest

An n8n workflow that tracks FocusKPI's competitors, summarises the week's news with an LLM, and posts a digest to Slack every Monday at 9 AM.

Built as a take-home assignment.

## What it does

Every Monday morning it pulls news about five competitors (Fractal Analytics, Tredence, Quantiphi, Tiger Analytics, LatentView) plus the wider enterprise-AI market. It keeps the last seven days, removes duplicate headlines, and asks Llama 3.3 70B for 4 to 5 takeaways aimed at leadership. Each takeaway has a "Why it matters" line and a source citation. The digest goes to the `#competitor-news` channel.

A typical run looks at 16 articles and finishes in about 4 seconds.

<img width="1045" height="635" alt="Weekly digest posted to Slack" src="https://github.com/user-attachments/assets/40475ba5-58d6-4784-ace8-47b42fd5172c" />

<img width="1045" height="617" alt="Resolved source list with real article links" src="https://github.com/user-attachments/assets/9084ca1d-ccd2-4494-a056-22ef26462f11" />

## Pipeline

```
Schedule Trigger (Mon 09:00, America/New_York)
        |
        ├──► HTTP Request: competitor feed (5 rivals by name)
        └──► HTTP Request: industry feed (enterprise AI / analytics)
                    |
                 Merge (append)
                    |
        Code: Parse & Prepare
        - parse the RSS XML
        - keep the last 7 days
        - drop duplicate headlines
        - cap 8 items per feed
        - build the numbered list and prompt
                    |
        LLM: Llama 3.3 70B via Groq (temp 0.3)
        - analyst framing, cites [n], never URLs
                    |
        Code: Format Digest
        - turn [n] back into real links
        - fix bold for Slack
                    |
        Slack: #competitor-news
```

## Why I built it this way

**Real competitors, not generic AI news.** The brief said competitor news, so I first worked out who FocusKPI actually competes with instead of pointing at a general tech feed. A digest about the AI industry is a newsletter. A digest about your rivals is competitive intelligence, and that felt like the point.

**Google News RSS instead of company blogs.** Consulting firms post to their blogs irregularly and some have no feed at all. A Google News query is stable, needs no login, and picks up third-party coverage, which is usually where funding, IPO, and hiring news actually shows up. Adding another competitor is a one-line change to the query.

**Two separate feeds.** Each article is tagged with its category, so the model can tell "a rival did something" apart from "the market moved."

**Clean up the data before the AI sees it.** Aggregators repeat the same story across a lot of outlets. Filtering and deduping in code keeps the token cost down and stops one heavily-covered story from pushing out four real ones.

**The model never sees a URL.** It gets a numbered list of headlines and can only cite `[n]`. A later Code node turns those numbers back into the real links I captured while parsing. Because the model has no link to work with, it can't invent a fake one. I liked this better than just telling it "don't make up links."

**Feeds fail on their own.** Both HTTP nodes keep going if they error, so if one source is down the digest still ships with whatever the other one returned.

## Prompt design

A few things I put in the prompt on purpose:

- Framed it as an analyst writing for leadership, so I get implications instead of summaries.
- Required a "Why it matters" line on every point. Without it the model just lists headlines.
- Told it what to prioritise (funding, acquisitions, launches, client wins, market shifts) and what to ignore (stock-price noise).
- Gave it a fixed reply for a quiet week so it doesn't invent significance out of nothing.
- Spelled out Slack's formatting, with a code cleanup step as a backup in case it slips.

## Things that broke while building

**Slack showed literal asterisks.** I'd written the formatter for normal Markdown, but Slack uses its own format (`*bold*`, links as `<url|label>`). I fixed the prompt and added a cleanup pass so the output doesn't rely on the model getting it perfect.

**The bullet clashed with the bold marks.** After that fix, headlines came out italic because the model was writing `* *Headline*` and Slack paired the asterisks the wrong way. I switched the bullet to `•`.

**The citations changed shape.** Once I told the model to merge duplicate stories, it started writing `[1, 2, 4]`. My code only understood single numbers, so the source list quietly dropped to one entry while the text cited six. I fixed both sides: the prompt now uses separate brackets, and the parser handles either version. The lesson I took from it is that an LLM's output format will drift, so the code reading it should be forgiving.

## Cost and scale

| Measure | Value |
|---|---|
| Articles per run | 16 (capped at 8 per feed after dedupe) |
| Tokens per run | roughly 1,400 in, 450 out |
| Runs per year | 52 |
| LLM cost per year | under $0.10 at current Groq rates |
| Run time | about 4 seconds |

Because the article count is capped before the LLM step, cost stays flat even as the feeds get noisier.

## What I'd do next

- **Smarter dedupe.** Right now it catches near-identical headlines but not the same story reworded across outlets. Embeddings would handle that.
- **Memory between runs.** A story running two weeks in a row shows up twice. Saving which articles I've already sent would let it say "still developing" instead.
- **A second news source.** Fine for a weekly internal brief, but a real version would add a backup provider and warn me if a run comes back empty.
- **A feedback loop.** Slack reactions could tell me which takeaways leadership actually finds useful.

## Running it yourself

1. Import `workflow.json` into n8n (⋯ menu, then Import from URL or File)
2. Add your own Groq API key to the LLM node
3. Add a Slack bot token (with `chat:write`) to the Slack node
4. Create a `#competitor-news` channel in your workspace
5. Set the timezone under ⋯ then Settings
6. Publish

The export has no credentials in it, only credential IDs, which mean nothing outside my own account. So you'll need to add your own keys.

## On AI use

I designed the workflow and made the calls myself: which competitors to track, cleaning the data before the AI step, and having the model cite numbers instead of links. I used Claude to speed up the JavaScript and to sanity-check my prompt wording. I built, connected, and debugged every node, and the three bugs above I found by running it and reading the output.

Chandra Narayana Darapaneni
