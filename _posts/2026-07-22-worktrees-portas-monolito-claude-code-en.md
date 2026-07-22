---
layout: post
title: "Twenty worktrees, twenty ports, one monolith: parallel development with Claude Code"
date: 2026-07-22
categories: [claude-code, java, spring, vaadin, git, maven, ai, workflow]
---

On an ordinary working afternoon, this is what my desk looks like: one Claude Code session is pushing the sales module migration forward. Another is working on finance. A third is refactoring shared components, and a fourth is finishing an HR feature. All of them in the **same** Java/Vaadin **monolith** — each one compiling, each one with the full application running and logged in in my browser, none of them stepping on the others.

There are no microservices in this story. No container per session, no exotic tooling. There is one monolithic repository, `git worktree` — a feature that has existed since 2015 — and three isolation decisions that cost a few painful collisions before they stood up straight. This post opens the blog's Claude Code section by explaining how the whole arrangement works.

## The bottleneck moved

If you follow this blog you know the setting: I am migrating a 25-year-old Domino ERP to Spring Boot + Vaadin, module by module — registry, sales, finance, purchasing, each with its dependencies. The repository is one, and the application is one; modules are packages of a monolith, not services.

Working alone, one module at a time, that is fine. But AI agents changed the economics of development: writing code stopped being the bottleneck. One Claude Code session can carry a whole migration front with light supervision — so why not several fronts at once? The honest answer from the first attempt: because they collide. Two sessions in the same directory fight over the working tree. Two simultaneous builds corrupt the Maven cache. Two instances of the application fight for the same port, and two logged-in tabs knock each other's sessions out. The bottleneck had moved: it was no longer writing code — it was **isolation**.

## The foundation: one worktree per front

The base layer is `git worktree`: the same repository, materialized into several sibling directories, each on its own branch.

```
git/
├── platform/                    ← master
├── platform-sales/              ← branch sales-migration
├── platform-finance/            ← branch finance-migration
├── platform-registry/           ← branch registry-migration
└── ...                          (~20 active worktrees today)
```

Every front of work — a module migration, a cross-cutting feature, a core-hardening effort — lives in its own worktree, and every worktree has **a dedicated Claude Code session**, loaded with that module's context and nothing else. There are about twenty today. Worktrees solve the first collision (each session gets its own working tree and branch), but by themselves they solve nothing else: the twenty directories contain the *same* code, the same `application.properties`, the same `pom.xml`. Start two of them "raw" and they collide on the same port. What makes the arrangement work are the two identity layers on top.

## Runtime isolation: a port, a cookie, and a `.env` that never enters git

The first layer gives each worktree an **execution identity**. The convention: each front boots the application on its own port in the 3nnn range — sales on 3000, finance on 3001, registry on 3002, and so on. Memorable, predictable, and the collisions disappeared.

But the port was the obvious part. The pain I had not anticipated: open the sales module in one tab and finance in another, log into the second and... get logged out of the first. Same host, same session cookie name — the browser keeps only one. The fix is to give each worktree **a session cookie with its own name**: `JSESSIONID_APP_SALES`, `JSESSIONID_APP_FINANCE`. With that, twenty logged-in applications coexist in the same browser, each with its own session.

Where does that identity live? In a **gitignored** `.env` at the root of each worktree:

```
PORT=3000
WORKSPACE_LABEL=sales
SESSION_COOKIE_NAME=JSESSIONID_APP_SALES
```

A dotenv loader reads the file at boot, and `application.properties` — identical across all worktrees — holds only placeholders with defaults: `server.port=${PORT:8081}`, `server.servlet.session.cookie.name=${SESSION_COOKIE_NAME:JSESSIONID}`. The versioned code does not know worktrees exist; the identity is local, per directory, and **never enters git** — which also means no merge, however messy, can ever swap a workspace's identity.

## Build isolation: one `.m2` per worktree

The second layer was the most expensive lesson. Twenty worktrees sharing the global `~/.m2` means simultaneous builds writing to the same artifact cache — and a multi-module monolith's reactor installs its internal modules into the local repository. Session A runs `mvn install` while session B compiles: B resolves a half-written internal artifact, and the error message says nothing about any of that. Worse: the application *running* in session C may be serving classes from a jar that was just overwritten under it.

The fix fits in one line, in each worktree's `.mvn/maven.config`:

```
-Dmaven.repo.local=../.m2repo
```

The **relative** path is the trick: each worktree resolves it to its *own* Maven repository directory. Simultaneous builds stop seeing each other. It costs disk — twenty copies of the dependencies — and disk is the cheapest thing in this entire arrangement.

To this layer also belongs a hygiene rule learned the hard way: when killing an application, **kill by the PID owning the specific port**, never by a broad command-line filter ("kill everything that looks like spring-boot"). A broad filter takes down all twenty workspaces at once — including the ones whose sessions were mid-test.

## Consolidation: "let's sync everything"

Twenty fronts moving in parallel raise the inevitable question: how does this become *one* system again? The deploy is monolithic — it packages one branch, one app. Merging master into each front does not solve it: that carries the common base out to each branch, but none of them ends up containing the others' work. You need one point with **everything**.

The cadence that emerged: fronts work independently, and periodically — with all worktrees "quiet", meaning each at a buildable, coherent point with everything committed — I say "let's sync everything" and a custom Claude Code slash command runs the flow:

1. **An integration branch off master.** Master is not touched until the very end — everything reversible.
2. **A `--no-ff` merge of each front, one by one.** The classic conflict is always the same, and it has become routine: the main navigation/menu class, which every front edits to add its entries. The resolution is mechanical — the *union* of both sides' entries — but the step stays in the flow as an explicit stop, because every so often a conflict is not mechanical: it is a product decision disguised as a diff.
3. **A full-reactor build.** Even skipping test *execution*, the tests still **compile** — and that is where cross-front API breaks surface: the signature front A changed and front B still consumes. The integrated build is the first moment the twenty worlds actually meet.
4. **Master adopts via `--ff-only`** — master only ever moves forward, never absorbs a conflicted merge — and the deploy ships from it.
5. **Sync back:** each worktree fast-forwards to master. Since master now contains every branch, all of them are fast-forward-eligible — and the twenty fronts, master, and production end up **on the same commit**. The cycle restarts from zero, everyone identical.

The flow runs autonomously, but with **two intentional stops** agreed upfront: human confirmation before the production deploy, and a mandatory halt if a conflict stops being mechanical. That is not a tool limitation — it is the contract. Autonomy along the way, human decisions at the irreversible points.

## The monolith did not have to become microservices

The reflection that stays with me: when AI agents collapsed the cost of writing code, the development bottleneck migrated to isolation and integration — and the answer required no new tooling at all. `git worktree` is from 2015. Dotenv is trivial. A port convention is a table in a file. Merge discipline is discipline. The monolith stayed a monolith; it only gained a **per-workspace identity** — port, cookie, build cache — which is what genuinely parallel development actually demands.

And that parallelism explains a decision from an earlier post: it is because twenty simultaneous AI sessions exist that the [MCP server over the ERP]({% post_url 2026-07-22-mcp-server-domino-rest-api-en %}) was born **read-only by design** — free reads for every session, writes only through versioned, reviewable scripts. The two architectures are the same principle at different layers: generous parallelism on reads, ceremony on writes. So far, it is the arrangement that lets twenty fronts move fast without any of them breaking what the others are doing.
