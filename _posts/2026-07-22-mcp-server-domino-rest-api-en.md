---
layout: post
title: "Giving AI agents eyes into a 25-year-old Domino ERP — an MCP server over the Domino REST API"
date: 2026-07-22
categories: [domino, drapi, java, spring, spring-ai, mcp, ai, architecture]
---

There is an ERP that has been running on Lotus Notes/Domino for 25 years. It has survived version upgrades, server moves, and several generations of technology — and it is still the operational heart of a real business. Last week, an AI agent asked that system questions in plain natural language — about customers, orders, data history — and got answers. Not a single line of the legacy system was changed to make that happen. The entire bridge was built on the outside, on top of the Domino REST API (formerly Project KEEP), as an MCP server written in Spring Boot.

This post is about how that bridge was built, the three architecture decisions holding it up — and the detail I find most interesting of all: the moment when the *accumulated knowledge about the API* became a tool itself.

## The context: gradual migration, living database

If you follow this blog you know the story: I am gradually migrating a Domino ERP to Spring Boot + Vaadin. The key word is *gradually*. Domino was not shut down or frozen — it is still the live database of the operation, and the new application talks to it through the Domino REST API. Modules migrate one at a time; the data stays where it has always been.

In that setting, a need showed up that I had not anticipated: the AI sessions helping me develop needed to *see* the data. There is no point asking an agent to write the data-access layer for a module if it cannot check which fields a document actually has, what a view returns, or whether that numeric field arrives as a string. I was spending far too much time pasting JSON from REST responses into the chat. The agent needed eyes of its own.

## The build: an MCP server in Spring Boot

The answer was an MCP (Model Context Protocol) server built with Spring Boot and Spring AI, exposing roughly 60 tools over the Domino REST API. I won't list all sixty — they fall into a few categories:

- **Lookup by business key**: find a customer, an order, a document by the code the business actually uses;
- **View queries**: read Domino's native indexes, with pagination;
- **Aggregate analytics**: totals, rankings, trends over time;
- **Introspection**: discover a form's schema, list the available views.

One design pattern paid for itself many times over: the tools live in a **shared module** consumed by two different clients. The MCP server exposes them to external agents (Claude Code, for instance); the internal chat inside the Vaadin application consumes the exact same classes through Spring AI. I write a tool once and it shows up in both worlds — the assistant embedded in the ERP and the development agent on my machine run the same code, with the same descriptions.

The server is light: it boots in about 4.5 seconds and runs as a separate process from the main application (port 8090 in my examples). An agent connects over MCP, discovers the tools, and starts asking.

## Three decisions worth writing down

### 1. Read-only by design

The server writes nothing to Domino. No create, update, or delete tools — and that is an architecture decision, not a limitation.

The reasoning: a generic write tool is *too convenient*. With several AI sessions running in parallel — which is how I work these days — convenient generic writes are a recipe for clobbering: two sessions modifying the same document with stale views of each other. Against a production database holding 25 years of data, that risk is non-negotiable.

When something does need to be written, it happens through **versioned scripts**: code that goes into git, gets reviewed, has a diff, and can be re-run or rolled back deliberately. The agent may *propose* the script; executing it is a deliberate, auditable act. Free reads, ceremonial writes. That asymmetry has turned out to be exactly the right balance.

### 2. A service account with token caching

Authentication against the Domino REST API uses a JWT with an expiry. The naive implementation — authenticate on every call — works for the first few minutes and then starts failing in strange ways. The reason: Domino ships an anti-DDoS protection mechanism that behaves like a *tarpit*. A client hammering the authentication endpoint starts being treated as an attacker, and responses get progressively slower until it amounts to a block. With an AI agent firing dozens of calls per minute, you get there fast.

The fix is a token manager with three behaviors:

```
TokenManager:
  - authenticates once with the service account
  - schedules proactive renewal shortly before the token's exp
  - on an unexpected 401, re-authenticates and retries the call ONCE
```

Authenticate once at startup, renew before expiry (never waiting for the 401), and keep the 401 retry only as a safety net — capped at a single attempt so it can never degenerate into an authentication loop. Since this manager went in, the tarpit has never shown up again.

### 3. Introspection tools — and the traps they avoid

The tools agents use most are not the business ones: they are the introspection ones. Querying a form's schema before writing code that consumes it eliminates an entire class of guessing errors.

Two scar-tissue rules are baked into the tool design:

**Never use the UNID as a business identifier.** The UNID looks like a perfect ID — universal, unique, present on every document. But it can change across replicas. Every lookup tool works with business keys (the customer code, the order number), which survive any replica.

**Always paginate with a cursor.** Queries against the Domino REST API **truncate silently**: you get one page of results with no indication whatsoever that there was more. A tool that returns "this quarter's orders" without cursor pagination actually returns "some of this quarter's orders" — and the agent has no way to know. The query tools expose the cursor, and agents learn to iterate to the end.

## The clever bit: knowledge served as a tool

Here is the part that made me write this post.

After months of working against the Domino REST API, the lessons were everywhere — scattered across the memories of dozens of AI sessions, in multiple workspaces. Every new session started from zero, or close to it: it knew whatever was in that workspace's memory and was ignorant of the rest. The knowledge existed, but it did not circulate.

The solution came in two steps. First, an agent workflow distilled everything: **8 parallel readers** swept the existing memories, a **synthesis** stage consolidated the findings, and a **completeness critic** pointed out what had been missed — feeding the cycle again. The result was a markdown compendium with **15 sections** and roughly **560 facts** about the API: authentication, pagination, data types, pitfalls, query patterns.

The second step is the clever bit: **the compendium itself became an MCP tool.** Three operations — read the index, fetch a section by topic, full-text search. The file is read from disk on every call, which means updating the knowledge is editing a `.md` file. No rebuild. No restart. Learned something new today? I edit the file, and on the very next call every agent already knows it.

The practical effect is hard to overstate: any AI session, in any workspace, connected to the MCP server, now "knows" everything that months of project work taught us about the API. Memory stopped being per-workspace and became infrastructure.

## The proof

Shortly after the tool went live, I ran an honest test: I opened a completely fresh AI session, with zero prior context, and asked it about a subtle numeric-precision bug that had cost us dearly months earlier (the full saga deserves — and will get — its own post).

The session queried the compendium, found the right section, and reconstructed the whole story: the symptom, the cause, the fix that was applied. And then it did something I had not asked for: on its own initiative, it called the schema-introspection tool to verify, against the live environment, that the fix still held. Distilled knowledge plus live data, combined at the agent's own initiative. That was the moment the architecture paid for itself.

## Reflection: MCP as a read layer over legacy

What impresses me most about this story is what did **not** happen: the ERP did not change. No new Notes agent, no new field, no deployment on the Domino server. All of the "agent legibility" was added from the outside — the Domino REST API exposes the data, the MCP server adds semantics and safety, the compendium adds context.

I suspect this is a general pattern, well beyond Domino: **MCP as a read layer over legacy systems** may be the lowest-friction path to combining AI with old software. You do not need to migrate first, or wrap the legacy system in microservices, or wait for the big modernization project to finish. If a read API exists — and for most legacy systems it does, or can — you can give an agent eyes today. Read-only, with introspection, and with the institutional knowledge served as a tool, is enough for a 25-year-old system to start answering questions.

The legacy system did not become modern. It became *legible*. And in practice, that changed everything.

---

The production lessons on the Domino REST API that fed this post — the failure modes, silent drops, and fixes, in Symptom → Cause → Fix format — are published in the [domino-rest-api-guide](https://github.com/mmtcastro/domino-rest-api-guide) repo.
