# Building an AI Analyst That Doesn't Make Things Up

An LLM will answer a question it has no real basis for just as confidently as one it actually knows the answer to. That's the single biggest risk in building a chat feature on top of a real dataset, and it shaped nearly every decision in this layer of the platform. The chat system here isn't meant to be a conversational LLM with a database attached. It's meant to be data-grounded: the database returns real, verifiable results first, and the LLM's only job is to explain what's already there — not to invent anything on top of it.

## Splitting retrieval and generation cleanly

The core design decision follows directly from that principle: structured retrieval and narrative generation are kept strictly separate. The database returns a grounded JSON payload before an LLM ever gets involved, and the LLM is explicitly instructed to explain the retrieved data rather than fabricate facts. This sounds like a small distinction, but it changes what the LLM is actually being asked to do — not "answer this question," but "narrate this evidence."

SQL is good at precise filtering: give it a date range, an actor, an event type, a location, and it returns exact matches. What SQL can't do is answer an open-ended question like "what's the broader context around this topic" — that's a semantic question, not a filtering question, and it needed a different tool. That's where ChromaDB comes in. Event summaries and article snippets are embedded using `all-MiniLM-L6-v2` and stored in a persistent collection. When a question calls for background rather than statistics, a vector similarity search returns the most relevant events, complete with source URLs and content snippets. The division of labor stays clean: SQL handles precise, structured questions, ChromaDB handles open-ended semantic ones, and neither tries to do the other's job. The vector index itself — around 50MB of SQLite and HNSW data — ships with the repository, so the chat layer can use it immediately, or rebuild it from scratch if needed.

## Deep Dive Report: three layers running at once

The most complex feature in this layer is the Deep Dive Report, which takes a basic AI summary and enriches it with three additional layers of context:

| Layer | Data source | Content |
|---|---|---|
| Storyline | MySQL events + GKG themes | Timeline, how participants evolved, how locations shifted, theme trends |
| News Coverage | Live scraping of the event's source URL, with a ChromaDB fallback | Original headlines, content summaries, source links |
| GKG Insights | Google BigQuery (the public GDELT GKG dataset) | Related people, organizations, media themes, sentiment trends |

The execution flow is what makes this fast enough to be usable:

```
POST /api/v1/analyze/event-report
   → generate_event_report()
   → Phase 1: three data-gathering tasks run in parallel via asyncio.gather()
        ├─ gather_news_coverage()   → scraper (direct HTTP)
        ├─ gather_related_news()    → scraper (batch)
        └─ gather_gkg_data()        → BigQuery client
   → Phase 2: build the full storyline (timeline + participants + themes)
   → Phase 3: format everything into a single LLM prompt (truncated to 8000 characters)
   → Phase 4: LLM generates the narrative report
   → Phase 5: parse and return as JSON
   → Phase 6: frontend renders the report
```

Running those three data-gathering tasks in parallel, rather than sequentially, is what keeps the whole thing from feeling sluggish — the same instinct behind the parallel-query pattern from a few posts back, applied here to network calls instead of database queries.

The news scraper isn't a naive `requests.get` either. It caches URLs in memory for an hour, caps concurrency at five requests sharing a session, enforces a 12-second total timeout with a 5-second connection timeout, and extracts content intelligently — trying an `<article>` tag first, then `<main>`, then anything marked with a `main` role, falling back to scoring individual paragraphs by their CSS class if nothing more structured is found. If scraping fails outright, it falls back to whatever's already in ChromaDB rather than returning nothing.

## Using a pay-per-scan API without a surprise bill

The GKG Insights layer is the one piece of this system that touches a genuinely dangerous cost model. BigQuery charges by data scanned — about $5 per terabyte — and a query against the GDELT GKG dataset without partition filtering can scan roughly 3.6TB in one shot, which is close to $18 for a single request. On a feature that could get called repeatedly by ordinary usage, that's not a hypothetical risk; it's a bill waiting to happen.

The guardrails built around this are worth naming individually, because each one closes a specific way this could go wrong:

- **Partition filtering is mandatory, not optional.** A query without a `_PARTITIONTIME` filter is rejected outright, before it ever reaches BigQuery.
- **Every query gets a dry run first**, estimating the bytes it would scan before actually running it.
- **Hard caps are enforced**: 1GB per query, 10GB per day — roughly five cents a day at worst.
- **Results get cached for an hour**, so repeated requests for the same insight don't re-trigger a billed query at all.

And when GCP credentials simply aren't present, the system doesn't fail — `available` comes back `false`, the theme evolution section returns empty, and the frontend prompts the user to configure it. But the rest of the Deep Dive Report — the summary, the storyline, the news coverage — still works. That's the right instinct for anything optional: a missing enhancement should degrade the feature, not take down the whole thing with it.

## Three bugs that are worth remembering, because they'll happen again

A few concrete bugs from this layer are worth calling out specifically, because none of them are unique to this project — they're the kind of thing that shows up anywhere data crosses a boundary between two pieces of code that were written with slightly different assumptions.

**A key-naming mismatch silently returned empty data.** Code was looking for a dictionary key called `event_detail`, but the actual key had a numeric suffix — `event_detail_0`. The fix added a lookup function that checks both the exact key and any key matching that prefix.

**A dict was treated like an object.** One function returned a plain dictionary, but the calling code assumed it was an object and tried to call a method on it. The fix made the calling code check the type first: read it as a dictionary if it is one, fall back to the object-style access if it isn't.

**A dependency installed by hand disappeared on restart.** A required package had been `pip install`-ed directly inside a running container to unblock development, but it was never added to the Dockerfile. The next container restart wiped it out, and the feature broke again with no code change at all.

The lesson underneath all three is the same one: data contracts — key names, return types, whether something is a dict or an object — need to be validated at the boundary where two pieces of code meet, not assumed. And a fix applied by hand, inside a running system, isn't actually fixed until it's committed to whatever builds that system from scratch.

---

That closes out the AI analysis layer, and with it, the last of the three deep technical threads in this project — the database, the prediction model, and now the layer that ties both of them into something a person can actually ask questions about. Everything from here forward has been about proving each piece works. What's left is stepping back from all of it: what the platform actually delivers end to end, what its honest limitations are, and what I'd genuinely do differently if I started this project again today. That's the last post in this series.
