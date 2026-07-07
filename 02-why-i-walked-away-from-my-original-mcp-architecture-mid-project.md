# Why I Walked Away From My Original MCP Architecture Mid-Project

At some point in the middle of this project, I noticed something uncomfortable: I was spending more time fixing errors in my tool-calling plumbing than I was writing actual features. Not edge cases. Not rare failures. Routine, load-bearing parts of the system kept breaking in ways that had nothing to do with the business logic I was trying to build.

That was the signal. The architecture I'd committed to at the start of the project — the one written into my original proposal — wasn't the one I ended up shipping. This post is about that decision: what the original design looked like, what specifically went wrong, and why I chose to change course instead of pushing through.

---

## What I started with

The proposal-stage architecture was built around MCP — the Model Context Protocol. Every capability in the system was exposed as an MCP tool, and the client drove everything through tool calls. On top of that sat a router:

```
Client (mcp_app)
  └─ Router (Ollama + qwen2.5:3b): intent classification, tool preselection, safety filtering
       └─ LLM (Kimi / Claude): complex reasoning + tool execution
            └─ MCP Server: all tools (with caching)
                 └─ MySQL 8.0
```

On paper, this made sense. MCP is a genuine standard for exposing capabilities to a model in a structured way, and a local router in front of it meant not every request had to hit an expensive remote LLM. I could point to the design and explain why each layer existed.

## What actually happened

In practice, three problems kept surfacing, and none of them were about the business logic.

**Tool-call overhead was heavy, even for simple queries.** A request that should have been a single database lookup still had to go through the full tool-calling chain — routed, dispatched, executed, returned — before an answer came back. The protocol added a layer of ceremony that a simple query didn't need.

**The glue layer produced its own category of bugs.** I kept running into errors like `tool_call_id is not found`, along with empty or malformed tool call IDs, and duplicate tool registrations from older tool files whose functionality overlapped with newer ones. None of these were bugs in what the system was supposed to do. They were bugs in the machinery connecting the pieces together.

**Frontend and backend integration wasn't direct.** Debugging a broken response meant tracing back through the tool-call chain to figure out where in that chain things had gone wrong, instead of just looking at a request and a response.

Individually, each of these was fixable. Together, they added up to a pattern: I was spending my engineering time on protocol plumbing instead of the platform itself.

---

## The decision

I switched the main branch to a LangChain hybrid planner. A local Ollama router still extracts structure from a request — that part of the original design was worth keeping. But instead of routing everything through MCP tool calls, a rule-based fast path now handles simple requests directly, and a remote LLM is reserved only for generating complex reports. Structured queries go through a shared SQL and data-service layer instead of a tool-calling chain.

I want to be precise about what this pivot actually was, because "we abandoned MCP" makes it sound more dramatic than it was. The `mcp_server/` directory is still in the repository — it's kept as a legacy reference layer. What changed is that the core application no longer depends on it to run. The ideas that were working — a router for extracting intent, caching, intent-driven design — survived the pivot intact. What I replaced was a single assumption: that MCP tool calls should be the execution backbone for the whole system.

The result, in plain terms: a simpler system, lower tool-call overhead, and frontend/backend integration that's more direct to debug. I'll get into exactly how the new planner is structured — the routing tiers, the local model, the fallback logic — in a later post.

---

## What this decision actually taught me

The lesson I keep coming back to is this: **a protocol being more sophisticated doesn't mean it's a better engineering fit for the problem in front of you.** MCP is a legitimate, well-designed standard. It just wasn't the right execution model for this particular platform, at this particular stage, with this particular set of requirements.

The harder part wasn't identifying the problems — the errors were loud and repeatable. The harder part was accepting that an architecture I'd committed to in writing, before I understood the problem as well as I eventually would, was worth walking away from. When the bugs in your glue layer start outnumbering the bugs in your actual business logic, and routine operations are being forced through unnecessarily complex paths, that's usually a sign to revisit the assumptions — even the ones that made it into the original proposal.

That principle — keep the parts that work, and don't be precious about the parts that don't — ended up shaping more than just this one decision. The next post covers the architectural philosophy that came out of this pivot, and it's the thread that runs through almost everything else in this series.
