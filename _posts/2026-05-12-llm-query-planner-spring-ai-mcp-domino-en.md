---
layout: post
title: "The LLM as query planner: Spring AI tools + MCP over a Domino database"
date: 2026-05-12
categories: [domino, drapi, java, spring, spring-ai, mcp, ai, llm, agents, architecture]
lang: en
---

Imagine asking *"which books were borrowed more than five times last quarter"* and getting the answer from a database designed before the Web was common. It's not text-to-SQL. It's not RAG over a document dump. It's an LLM choosing, among dozens of annotated Java methods, which one to call — and the method reads a native Domino view via REST. This post describes the architecture that makes this possible, why it is simpler than it looks, and what shifts mentally once the LLM becomes your query planner.

## The unlikely combination that works

Domino was designed in 1989. Schemaless, multivalue, hierarchical documents, views as materialized indexes. Everything a modern query planner *dislikes*. Spring AI and the MCP (Model Context Protocol) are from 2024–2025. Everything a legacy database *doesn't know exists*.

The bridge has three pieces:

```
┌──────────────────────────┐
│ LLM (Claude / GPT / etc) │
└────────────┬─────────────┘
             │  tool calls (JSON-RPC)
             ▼
┌──────────────────────────┐
│ MCP Server (Spring Boot) │
└────────────┬─────────────┘
             │  Spring AI ToolCallbackProvider
             ▼
┌──────────────────────────┐
│ @Tool methods (Java)     │
└────────────┬─────────────┘
             │  WebClient
             ▼
┌──────────────────────────┐
│ Domino REST API (DRAPI)  │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ HCL Domino — NSF + views │
└──────────────────────────┘
```

The part that looks like magic — the LLM picking the right function, assembling arguments, interpreting the response — is Spring AI doing standardized tool calling. The part that looks impossible — a 1989 database speaking structured JSON — is DRAPI: a REST server HCL ships, which exposes views, forms and CRUD operations over any NSF. What you're left to build is the middle layer: the `@Tool` methods.

## @Tool is where the intelligence lives

The LLM does not see your database. It sees only what you describe in prose, inside the `@Tool` annotation. Each method becomes an atomic operation the model can invoke:

```java
@Component
public class CatalogTools {

    private final DrapiClient drapi;

    @Tool(description = """
        Lists books in the catalog filtered by genre.
        Use when the user asks for books in a specific category
        (e.g. "science fiction", "biography", "children").
        Returns only title, author and year — full fields are
        available via consultBookByCode.
        """)
    public List<BookSummary> listBooksByGenre(
            @ToolParam(description = "Genre in lowercase") String genre,
            @ToolParam(description = "How many results, max 50") int limit) {

        return drapi.queryViewEntries(
                "library", "(BooksByGenre)",
                genre, Math.min(limit, 50));
    }
}
```

Two important things live here, and neither is the body of the method:

**The `@Tool` description is prompt engineering.** The model decides when to use this function by reading that string. "Use when the user asks for…" steers selection; "Returns only X — full fields via Y" trains the model to chain calls rather than load unnecessary data. Sloppy phrases here cause hallucinations or the wrong tool choice. Precise phrases become a *narrative contract* between you and the model.

**The `@ToolParam` description is prompt too.** "Genre in lowercase" is not validation — it's an instruction for how the model normalizes the human input before calling. You write less `if (input.isUpperCase())` in Java and more prose in the annotation.

## The dual mode: views or documents

DRAPI exposes two ways to read a view, and the performance gap is glaring:

- `documents=false` — only the columns the view publishes (fast)
- `documents=true` — loads the full document for each entry (slow, but everything is there)

On a view of hundreds of items, the second mode can be orders of magnitude more expensive. And the LLM has no way to know which to use — *unless you teach it*. The pattern that unlocks this is exposing two tools with opinionated descriptions:

```java
@Tool(description = """
    **PREFER THIS TOOL.** Lists entries of a view, returning
    ONLY the columns the view publishes. Use in ~80% of cases.
    Fast.
    """)
public String queryEntries(String view, String key, int limit) { ... }

@Tool(description = """
    **USE SPARINGLY.** Loads the full document for each entry.
    Slower. Use ONLY when you need a field that is NOT exposed
    in the view (e.g. body text, attachments).
    """)
public String queryDocuments(String view, String key, int limit) { ... }
```

The bold and the adjectives are deliberate. Models follow emphasized instructions. On a sample of a few hundred real calls, the "fast" tool was chosen by itself in nearly every situation that warranted it — and the "slow" one only appeared when the request genuinely required a field outside the view.

## Dynamic discovery via ToolCallbackProvider

In a small system you list tools by hand. In a system that grows, that becomes liability. Spring AI solves it with a `ToolCallbackProvider` that scans the Spring context and exposes every `@Bean` with `@Tool` methods:

```java
@Configuration
public class ToolsConfig {

    @Bean
    public ToolCallbackProvider catalogTools(
            CatalogTools catalog,
            LoansTools loans,
            MembersTools members) {

        return MethodToolCallbackProvider.builder()
                .toolObjects(catalog, loans, members)
                .build();
    }
}
```

Adding a new domain area is creating a `@Component` class, annotating methods with `@Tool`, and including it in the provider. No route, no DTO, no decorator. The LLM "discovers" on the next handshake.

## Why a separate MCP server, and not embedded tools

You can embed Spring AI directly into the app that handles chat — and in many cases that is enough. But standing up the tool layer as an **independent MCP server** (also Spring Boot, its own port) buys you three things:

1. **Reuse across hosts.** Claude Desktop, Cursor, OpenAI Agents SDK, IDE plugins — they all speak MCP. A single tool surface serves all of them.
2. **Decoupled life cycle.** The tool server can go up, down, scale independently of the app that consumes it.
3. **Explicit security boundary.** The MCP server is the single place to enforce scope, rate limit, audit. None of that scattered across N hosts.

The MCP wrapper itself is short — it registers `@Tool`s in the MCP shape, exposes them over stdio or HTTP, and that's it. The intellectual weight stays in the tools.

## What shifts mentally

Three displacements this pattern produces, worth naming:

### 1. The description is the API

In traditional REST, the contract is the URL plus the JSON schema. The human client reads documentation. In the `@Tool` pattern, the client is an LLM and the contract is prose. You don't write OpenAPI; you write conditional instructions ("use when…", "prefer… unless…"). This is programming, only the target language is natural-language semantics. Treating the description as first-class code — review, versioning, A/B tests with prompts — is the way.

### 2. The LLM is the new query planner

A modern relational database accepts SQL and runs an expensive optimizer to pick a plan. In this pattern, the "plan" is the sequence of tool calls the LLM emits. Your tools are the physical operators. The descriptions are the statistics. The LLM rarely fails out of lack of intelligence — it fails out of lack of information in the catalog. Caring for the catalog is caring for the optimizer.

### 3. Legacy databases become first-class citizens again

The instinct to discard pre-Web databases under the banner of "modernization" assumes the interface is the problem. It is not — the indexed-view read engine in Domino is, in many cases, faster than a naive query against a modern relational schema. What was missing was a client that spoke to humans. Spring AI + MCP delivers that client without touching the engine.

## What remains as work

This is not friction-free. Where most time goes:

- **Calibrating descriptions.** Ambiguous phrases cause the wrong tool to be picked. The loop is: run representative prompts, watch what the model chose, adjust prose. It's a loop, not a one-shot.
- **Scoping reads first.** The first version exposes read-only. Write tools demand a different order of care — human confirmation, audit log, idempotency. Don't board everything at once.
- **Cache and cost.** LLMs make tool calls that can be repetitive. A short in-memory cache (seconds) at the DRAPI edge cuts cost for duplicate calls within the same conversation turn.

But the skeleton — the three layers, the annotation, the dynamic discovery — is small. A few hundred lines of Java to put a decades-old database in natural-language conversation with frontier models. The fascinating part is that the target stack didn't need to modernize: it's the stack around it that became smart enough to take advantage of it as it is.
