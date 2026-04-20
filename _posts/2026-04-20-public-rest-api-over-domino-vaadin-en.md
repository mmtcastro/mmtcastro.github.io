---
layout: post
title: "Building a Public REST API Over Domino + Vaadin in One Day"
date: 2026-04-20
categories: [domino, drapi, java, spring, vaadin, api, rest, security, api-keys]
lang: en
---

Our internal portal runs Vaadin over Domino REST API (DRAPI). It works well for human operators with LDAP login. But external integrators — partners, third-party systems, automations — need something else: standard REST, Swagger, stateless auth. This post describes the day we split the public API from the internal portal, from zero to the first `curl` responding 200. Six hours, 3,058 lines, four gotchas catalogued.

## Why split

The internal portal has Vaadin sessions, CSRF, LDAP, stateful cookies. Shoehorning external integration in there means explaining `v-r=init`, forcing browser simulation, exposing internal topology over HTTP. The operational and cognitive cost isn't worth it.

We split into a new module — isolated Spring Boot, its own port, **reusing the existing DRAPI layer** as an internal library. The old module doesn't know the new one exists.

```
public-api (:8083)          internal-portal (:8081)
     │                             │
     └──────┬──────────────────────┘
            │
        DrapiClient (shared lib)
            │
        DRAPI / Domino
```

## Authentication: X-API-Key over NSF document

Storage decision: keys live in the same NSF as other cross-cutting data (`sistema.nsf`), as `ApiKey` docs. No extra relational database. Doc schema:

| Field | Content |
|-------|---------|
| `Id` | `sistema_ApiKey_<uuid>` (platform convention) |
| `Codigo` | **SHA-256 hex of the raw key** (O(1) lookup via `_intraCodigos`) |
| `Descricao` | Human label ("Partner Portal X — Production") |
| `Scopes` | `empresas:READ,WRITE;noc:READ` |
| `Ativo` | `true` / `false` (soft revoke) |

The interesting bit is `Codigo = hash`. Violates the "Código is human-readable" convention but eliminates the need for a custom view — the `_intraCodigos` that exists in every base becomes our key index for free. Nobody will ever look at these docs via Notes UI, so the ugliness of hash-as-code doesn't charge much.

The auth filter is a `OncePerRequestFilter`:

```java
String raw = req.getHeader("X-API-Key");
String hash = sha256(raw);
apiKeyDao.findByHash(hash).ifPresent(apiKey -> {
    ApiKeyPrincipal principal = new ApiKeyPrincipal(apiKey);
    SecurityContextHolder.getContext().setAuthentication(
        new UsernamePasswordAuthenticationToken(principal, null, List.of()));
});
```

The raw key is never persisted — only the hash. At emission time, via the admin portal, we show the raw key exactly once in a non-dismissible dialog ("copy now, won't be shown again"). After that only the hash lives in DRAPI.

## Enforcement: @RequireScope + Interceptor

For granular authorization, an annotation + a `HandlerInterceptor`:

```java
@GetMapping("/sobreavisos")
@RequireScope(base = "noc", permission = Permission.READ)
public List<SobreavisoResponse> list(...) { ... }
```

```java
public class ScopeCheckInterceptor implements HandlerInterceptor {
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        RequireScope req = ((HandlerMethod) handler).getMethodAnnotation(RequireScope.class);
        if (req == null) return true;
        ApiKeyPrincipal p = (ApiKeyPrincipal) SecurityContextHolder.getContext()
                .getAuthentication().getPrincipal();
        if (!p.allows(req.base(), req.permission()))
            throw new AccessDeniedException("missing permission " + req.base() + ":" + req.permission());
        return true;
    }
}
```

We initially tried `@Aspect` for this — more elegant. But `spring-boot-starter-aop:4.0.5` doesn't exist in the public repo at the time of writing. The interceptor is equivalent and needs no extra starter.

## The day's gotchas

Legacy platforms have gray zones that only surface when you push. Here are the four the day found — each one became a memory to avoid falling into again.

### 1. DRAPI silently drops fields outside the schema

We tried writing a new `Id` field on legacy docs via PATCH. Status **200 OK**. Response even omits the field. Subsequent GET: empty.

Cause: DRAPI validates the payload against the **Form schema declared in the scope**. Missing fields get dropped — no error, no warning, no trace. Notes itself is schemaless, the field may even reach the NSF, but DRAPI filters on the read path.

Fix: add the field to the schema before writing. Safe helper (read-modify-write with backup):

```bash
python drapi_add_id_field.py --scope noc \
       --forms Sobreaviso,DiaSobreaviso,AnotacaoTrabalho,TagAnotacao --apply
```

### 2. `documents=true` vs entries — a performance decision

DRAPI reads a view two ways:
- `documents=true` — loads the **full document** for each entry (including RichText/MIME)
- `documents=false` — just the **columns published by the view**

For views with the right columns, the full doc is pure overhead. On views with hundreds of docs it becomes a bottleneck.

We refactored `DrapiClient` into two explicit methods:

```java
drapi.queryViewEntries(scope, view, key, count)     // fast
drapi.queryViewDocuments(scope, view, key, count)   // loads doc
```

And updated the `@Tool` descriptions (the codebase uses internal agents) to guide the decision:

```java
@Tool(description = """
    **PREFER THIS TOOL** — queries view returning ONLY columns (entries).
    Use when the fields you need are exposed as view columns.
    Sufficient in ~80% of cases.
    """)
public String queryViewEntries(...) { ... }

@Tool(description = """
    **USE SPARINGLY** — loads the full document, slower.
    Use ONLY when you need a field NOT exposed in the view.
    """)
public String queryViewDocuments(...) { ... }
```

### 3. Menu is not `@Menu`

Took us a while to figure out why a new `@Route("sistema/apikeys")` with `@Menu(order=110)` wouldn't show up in the drawer. Cold restart, dev-bundle cleanup, slug change, Playwright tests — nothing.

Playwright trace: the item **wasn't even in the DOM**. The cause: the custom `MainLayout` populates the drawer by iterating a hardcoded `ModuleNavigation.MODULES` class, not Vaadin's `MenuConfiguration` registry. `@Menu` is ignored.

Trivial fix: add a `NavEntry` to the right list. But figuring this out burned 40 minutes of debugging. Saved as session memory so we won't trip on it again.

### 4. UNID is a proprietary identifier

Domino uses UNID (32 hex chars) as the physical document identifier. The day you migrate to SQL, every UNID becomes a pumpkin. The project convention is to persist `Id = scope_Form_uuid` as the logical identity. But legacy views still pass UNID in URLs (`/sobreaviso/<UNID>`) and service calls.

We tackled it in layers:

**(a) "ById" family in `AbstractService`** — `findById` already existed; we added `deleteById`, attachments by Id, with a 5s cache for Id→UNID resolution (DRAPI internally only deletes by UNID).

**(b) Smart route** in `AbstractViewDoc.beforeEnter`:

```java
String param = event.getRouteParameters().get("unid").orElse("novo");
// Logical Id contains '_' (scope_Form_uuid), UNID is just 32 hex chars
Response<T> response = param.contains("_")
    ? getService().findById(param)
    : getService().findByUnid(param);
```

Full backward compat. Old bookmark `/sobreaviso/<UNID hex>` keeps working.

**(c) Pilot on the listing view**: swap `s.getUnid()` for `s.getId()` with UNID fallback. Playwright smoke test confirmed — 5 of 5 links generated with `noc_Sobreaviso_<uuid>`, zero UNID.

**(d) Backfill of legacy docs**: many old docs never had `Id`. Idempotent Python script iterates `_intraForms` and PATCHes `scope_Form_uuid` into those missing it. Dry-run by default:

```bash
python drapi_backfill_ids.py --scope noc              # dry-run
python drapi_backfill_ids.py --scope noc --apply      # persist
```

Day's result: 32 NOC docs populated (Sobreaviso, DiaSobreaviso). The other 1,025 already had one.

## Endpoint returning 200 by end of day

```bash
# health — no auth
curl http://localhost:8083/v1/public/health
# {"status":"UP"}

# with noc:READ key
curl -H "X-API-Key: sk_..." \
     "http://localhost:8083/v1/noc/sobreavisos?ano=2024&limit=2"
# [{"id":"noc_Sobreaviso_750eb284-...","codigo":"2024/02",
#   "ano":"2024","mesExtenso":"February",...}]

# no key
curl http://localhost:8083/v1/noc/sobreavisos
# 403
```

Swagger UI at `/docs`, OpenAPI JSON at `/v3/api-docs`.

## What's next

- Bucket4j rate limiting (dependency in pom, missing wiring)
- Persist `apiKey.id + LastUsed + latency` on every call (billing/auditing)
- Continue UNID→Id area by area (NOC → invoices → sales → …)
- Granular permissions splitting CREATE/UPDATE/DELETE instead of current `WRITE`

## Four mental models I'm taking away

1. **Public APIs and internal UIs have different lifecycles.** Splitting early is cheaper than refactoring later.
2. **Schemaless hides friction.** One day the schema will charge you — via strict DRAPI, via a client expecting consistent types, via a SQL migration. Declaring the schema early pays negative interest.
3. **Stack conventions surface late.** Document them where the next work session (yours or an AI's) will trip over them. Shared memory across sessions is as valuable as the code.
4. **Identifier migrations need transparent heuristics, not feature flags.** Route accepts both → backfill → remove old code when coverage is high. Never big-bang.

Three clean commits, push to `master`, 43 new files, 3,058 lines added, 32 legacy docs recovered. And memories logged for four gotchas the next session won't have to re-learn from scratch.
