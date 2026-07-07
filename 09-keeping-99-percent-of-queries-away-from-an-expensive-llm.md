# Keeping 99% of Queries Away From an Expensive LLM

A few posts back, I wrote about walking away from an MCP-centered architecture mid-project, because tool-call overhead and glue-layer bugs were eating more of my time than the actual product. I said the replacement was a LangChain hybrid planner, and left the details for later. This is later. Here's exactly what took MCP's place, and why it's held up.

## Three tiers, ordered by cost

The planner routes every incoming question through three mechanisms, cheapest first:

```
User's natural language question
       │
       ▼
① Local Ollama router (qwen2.5:3b)
   Extracts structured fields: location / date_start / date_end
                  / event_type / query_text / intent_category
       │
       ▼
② Rule-based fast path
   Switches on intent_category, calls the data service directly
   → handles 99%+ of queries without touching a remote LLM
       │
       ▼ (only reached if the rules can't handle it)
③ Remote LLM (LangChain's ChatOpenAI)
   Generates a narrative report for genuinely complex questions
```

The design premise is the same triage logic as a support desk: most requests get resolved by whoever's cheapest to staff at the front line, and only the genuinely hard cases get escalated to a specialist. Here, that means most questions don't actually need an expensive remote model to answer at all. Rule-based routing resolves in under a millisecond. The local Ollama extraction step takes about 1.5-2 seconds. The remote LLM is there as a backstop, not a default. This also picks up small usability details that used to be dead ends — natural language dates like "May 1 2024," "last week," or "Q1 2024," and location shorthand like "NYC" or "DC" getting normalized to full names before they ever reach a query.

## Replacing 200 lines of regex with a 3B-parameter model

Before this design, the planner leaned on a large block of regular expressions — over 200 lines of it — to pull location, date, and event type out of a raw question. It worked, but only for the phrasing patterns someone had already thought to write a regex for. Anything slightly different fell through.

The fix was to stop trying to out-write regex for natural language and instead hand that job to a small local model, `qwen2.5:3b`, and let the rule engine focus purely on routing once the structure was already extracted. Coverage improved immediately, for the obvious reason: a language model generalizes across phrasing in a way a fixed set of patterns never will.

That said, handing structured extraction to a small model came with its own failure mode, and it's worth walking through because the fix generalizes well beyond this one project.

**The bug:** event fingerprints look like `EVT-2024-02-29-1160747286` — an ID that happens to contain what looks like a date. Ollama would occasionally parse that embedded date straight into the `date_start` / `date_end` fields, treating an identifier as if it were a date range the user had actually asked about.

**The fix had two layers, and neither one alone was enough:**

1. A prompt-level rule: the system prompt was updated to instruct that when the input is an event ID, only `intent_category=detail` and `query_text` (the full ID) should be set — every other field should come back null.
2. A code-level fallback: regardless of what the model returns, if `query_text` matches the `EVT-YYYY-MM-DD-NNNNNNNNNN` pattern or looks like a plain numeric ID, the code forces `intent=detail` and clears every other field itself.

The lesson that came out of this: don't fully trust a small model's structured output, even after you've told it exactly what you want. For anything with a strict, checkable format, back the prompt instruction with a code-level check that enforces the same rule deterministically. Prompt constraint plus code-level fallback is double insurance — the prompt handles the common case cheaply, and the code catches the model when it doesn't listen.

## Three small optimizations that added up

A few more practical adjustments came out of this layer, all aimed at reducing tokens and calls rather than adding new capability:

| Optimization | Approach | Effect |
|---|---|---|
| Lazy LLM loading | Delay initializing the LLM client via a `@property` | The rule-based path makes zero API calls, avoiding stray errors from an uninitialized client |
| Report pre-formatting | Convert raw JSON into a narrative-style block before handing it to the LLM | Roughly 70% fewer tokens, and a faster, more stable response |
| Simplified LLM input | Pass a simplified structured intent plus a short reference, instead of the full raw query | Less token overhead, and less room for the model to misinterpret the request |

The report pre-formatting step is worth showing directly, because the shape of the change matters more than the code itself — it's the difference between handing a model a data dump and handing it something already organized like a paragraph:

```
=== PRIMARY EVENT ===
Date: 2024-05-01 | Location: New York | Title: ... | Summary: ... | Articles: 42 | Tone: conflict (-8.5)

=== RELATED EVENTS ===
- Date: ... | Location: ... | Title: ...
```

Each change is modest on its own. Together, they're the difference between a chat feature that feels expensive to run and one that mostly doesn't touch the expensive path at all.

---

This is what actually replaced MCP: a router that costs a couple of seconds instead of a full tool-calling round trip, a rule engine that resolves the overwhelming majority of requests instantly, and a remote LLM held in reserve for the cases that genuinely need it. It's a smaller, more debuggable system than what it replaced, and it's the reason 99%+ of queries in this platform never touch a paid API call at all.

But routing a question correctly is only half the job. The other half is making sure that when the LLM does get involved — generating an actual narrative report — it explains real data instead of inventing plausible-sounding details. That's a different kind of engineering discipline, and it's the subject of the next post.
