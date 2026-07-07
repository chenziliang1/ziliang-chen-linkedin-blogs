# The One Architecture Principle That Held the Whole Project Together

If I had to explain this project's architecture in one sentence, it wouldn't be about any specific technology. It would be a rule: **keep fast, deterministic data access completely separate from LLM reasoning.** Once I committed to that rule, most of the other design decisions in this platform stopped being separate choices and started being consequences of the same idea.

## Fast paths and slow paths, kept apart on purpose

The Dashboard never touches an LLM. Every query behind it is structured and deterministic, aimed at a response time under 200 milliseconds. There's no round trip to a language model to slow it down, because there's nothing about "show me the last 30 days of events" that benefits from one.

Analyst Chat works differently, because it has to. A natural language question doesn't map cleanly to a database query on its own, so it goes through a planner first: the question gets parsed into structured intent, and only then does that intent get converted into a precise query or handed off for a narrative response. Plan first, execute second.

Reports follow the same instinct toward laziness. When a user opens an analysis, they get the underlying data immediately. The narrative report — the part that actually calls an LLM — only gets generated if the user clicks to request it. Nobody pays the cost of LLM generation until they've asked for it.

The frontend carries this same separation into its own layout: the Dashboard tab looks backward, the Forecast tab looks forward, and neither one tries to do the other's job. That's a small decision, but it's the same principle showing up again — don't let two things that answer different questions blur into one page that answers both badly.

Put together, this separation pays off in three concrete ways. The interface stays responsive, because a user can see data on screen before any LLM call has even started. Debugging gets easier, because data retrieval, planning, and report generation are three distinct stages instead of one tangled process. And hallucination goes down, because the report generator's only job is to explain data that's already been retrieved — it's not being asked to reason about the world from scratch.

---

## One place for every query

The second piece of this principle is about where SQL actually lives. Every analytical query in the platform — Dashboard aggregates, the map, timelines, event search, event detail, semantic news search, even the prediction service's data needs — routes through a single file: `core_queries.py`. It's the one source of truth for how this platform talks to its database.

That wasn't the starting point. Early on, SQL was scattered across individual services, each writing its own version of similar queries. Consolidating it into one shared layer, and then deleting the older service file that used to hold duplicated logic, was a deliberate cleanup step, not an accident of the initial design.

The payoff shows up in a very practical way: when a query pattern needs to be optimized, it only needs to be optimized once. I'll get into a specific example of this in the next couple of posts — a set of slow queries that all shared the same underlying problem, and all got fixed in a single pass, precisely because they lived in the same file instead of five different ones. Without a single source of truth, that would have meant hunting down and fixing the same bug five separate times.

---

## What deployment actually taught me

The platform runs as three services under Docker Compose: MySQL, a FastAPI backend, and a React frontend, each with its own exposed port, plus resource limits on the backend and a mounted secrets directory for external service credentials. Large data volumes — the raw GDELT CSVs, the MySQL data directory — never go into git. They get shared through import scripts or a separate database dump instead.

Two lessons from this stuck with me, and both are the unglamorous kind that only show up once you're actually running the thing instead of just designing it on paper.

The first: changing code on the host machine doesn't do anything by itself if the backend is running inside a container. The container needs an explicit restart before it sees the change. It sounds obvious written down, but it's an easy assumption to make wrong when you're used to a process that reloads on save.

The second, related lesson came from a package installed directly inside a running container to fix an immediate problem. It worked — until the container restarted and the fix disappeared, because it had never been written into the Dockerfile. A temporary fix that isn't committed to the build isn't really a fix. It's a countdown.

---

## Why this principle mattered more than any single technology choice

None of these three things — fast/slow separation, a single SQL source of truth, and a couple of deployment habits — are exotic. What made them matter was applying them consistently, so that a decision made in one part of the system didn't get silently violated somewhere else.

This is also the frame I'd point to if I had to explain why the database optimization work later in this project was able to move as fast as it did. When every query lives in one place, fixing a slow pattern isn't a five-file hunt — it's a single, contained fix that benefits everything at once. The next two posts get into exactly what that looked like in practice, starting with how 15 million rows of raw event data got turned into something queryable in the first place.
