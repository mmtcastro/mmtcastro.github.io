---
layout: post
title: "DQL via Java: advanced queries on Domino REST API with field projection"
date: 2026-02-27
categories: [domino, drapi, java, dql]
lang: en
---

Domino Query Language (DQL) allows you to execute complex queries on Domino via `POST /query`. Unlike traditional views, DQL supports date filters, OR/AND combinations and -- critically -- field projection that reduces payload by up to 95%. This post shows how we execute DQL from Java with Spring WebClient.

## The endpoint: POST /query

```
POST /query?dataSource={scope}&action=execute&mode={mode}&fields={fields}
Authorization: Bearer {token}
Content-Type: application/json

{
  "query": "Form = 'Invoice' and Date >= @dt('2026-01-01T00:00:00')",
  "mode": "default"
}
```

**Critical parameters:**
- `dataSource`: the Domino database scope
- `fields`: CSV list of fields to return (projection)
- `mode` in the body: **required** for DRAPI to serialize the fields correctly

## The base method: executeDql()

```java
private String executeDql(String dataSource, String dql,
        String fields) {
    try {
        // "mode" no body e OBRIGATORIO
        Map<String, Object> payload = Map.of(
            "query", dql,
            "mode", mode);

        // URI via String.format para evitar encoding de virgulas
        String uri = String.format(
            "/query?dataSource=%s&action=execute"
            + "&mode=%s&fields=%s",
            dataSource, mode, fields);

        return webClient.post()
            .uri(uri)
            .header("Authorization",
                "Bearer " + getUserToken())
            .header("Content-Type", "application/json")
            .bodyValue(payload)
            .retrieve()
            .bodyToMono(String.class)
            .block();

    } catch (Exception e) {
        log.error("Erro ao executar DQL '{}': {}",
            dql, e.getMessage());
        return null;
    }
}
```

**Important detail**: we use `String.format()` instead of WebClient's URI builder because the builder encodes commas in the field list (`fields=A,B,C` becomes `fields=A%2CB%2CC`), which DRAPI does not accept.

## Field projection: 95% less payload

Without projection, each document can have 2,700+ bytes. With projection of 5 fields, it drops to ~150 bytes:

```java
// Constante com campos necessarios
private static final String DQL_FIELDS =
    "DataFaturamento,RazaoSocial,TotalNota,Sit,Natureza";

// Sem projecao: ~2776 bytes/doc × 5000 docs = 13.8 MB
// Com projecao: ~150 bytes/doc × 5000 docs = 750 KB
// Reducao: 95%
```

## Example 1: date range filter with performance logging

```java
public List<Invoice> findByDateRange(
        LocalDate startDate, LocalDate endDate) {

    String dql = String.format(
        "'(InvoiceDate)'.DataFaturamento >= @dt('%sT00:00:00')"
        + " and '(InvoiceDate)'.DataFaturamento <= @dt('%sT23:59:59')"
        + " and Form = 'Invoice'",
        startDate.toString(), endDate.toString());

    log.info("DQL: {}", dql);
    log.info("Fields: {}", DQL_FIELDS);

    long startTime = System.currentTimeMillis();

    Map<String, Object> payload = Map.of(
        "query", dql, "mode", mode);

    String uri = String.format(
        "/query?dataSource=%s&action=execute"
        + "&mode=%s&fields=%s",
        scope, mode, DQL_FIELDS);

    String rawResponse = webClient.post()
        .uri(uri)
        .header("Authorization",
            "Bearer " + getUserToken())
        .header("Content-Type", "application/json")
        .bodyValue(payload)
        .retrieve()
        .onStatus(status -> status.isError(),
            response -> response.bodyToMono(String.class)
                .flatMap(body -> Mono.error(
                    new RuntimeException(
                        "Erro DQL: " + body))))
        .bodyToMono(String.class)
        .block();

    long apiTime = System.currentTimeMillis() - startTime;

    if (rawResponse == null || rawResponse.isBlank()) {
        log.warn("Resposta vazia ({}ms)", apiTime);
        return Collections.emptyList();
    }

    log.info("Resposta: {} chars ({} KB) em {}ms",
        rawResponse.length(),
        rawResponse.length() / 1024,
        apiTime);

    List<Invoice> result = objectMapper.readValue(
        rawResponse,
        objectMapper.getTypeFactory()
            .constructCollectionType(
                List.class, Invoice.class));

    long totalTime = System.currentTimeMillis() - startTime;

    log.info("Resultado: {} docs | API={}ms | Total={}ms | ~{} bytes/doc",
        result.size(), apiTime, totalTime,
        result.isEmpty() ? 0
            : rawResponse.length() / result.size());

    return result;
}
```

### Log example

```
DQL: '(InvoiceDate)'.DataFaturamento >= @dt('2026-01-01T00:00:00') and ...
Fields: DataFaturamento,RazaoSocial,TotalNota,Sit,Natureza
Resposta: 384000 chars (375 KB) em 1200ms
Resultado: 2560 docs | API=1200ms | Total=1450ms | ~150 bytes/doc
```

## Example 2: batch query with OR (avoids N+1)

Instead of making N calls to check whether each economic group has deals, we make a single DQL query with OR:

```java
public Set<String> findGroupsWithDeals(Set<String> codes) {
    Set<String> result = new HashSet<>();

    if (codes == null || codes.isEmpty()) {
        return result;
    }

    // Constroi DQL com OR: cod1 OR cod2 OR cod3...
    String dql = codes.stream()
        .map(c -> "'(DealsByGroup)'.GroupCode = '"
            + c.replace("'", "''") + "'")
        .collect(Collectors.joining(" or "));

    String rawResponse = executeDql(
        "sales", dql, "GroupCode");

    if (rawResponse == null || rawResponse.isBlank()
            || rawResponse.equals("[]")) {
        return result;
    }

    JsonNode root = objectMapper.readTree(rawResponse);
    if (root.isArray()) {
        for (JsonNode node : root) {
            String code = node.has("GroupCode")
                ? node.get("GroupCode").asText() : null;
            if (code != null && !code.isBlank()) {
                result.add(code);
            }
        }
    }

    return result;
}
```

**Gain**: 1 DQL call instead of N individual calls. For 500 groups, it goes from 500 HTTP requests to 1.

**Security**: `.replace("'", "''")` prevents DQL injection in the values.

## Domino views as DQL indexes

DQL can use views as indexes for performance. The `'(ViewName)'.FieldName` syntax forces DRAPI to use the view as an index:

```java
// Sem view (scan sequencial): ~3100ms
String dql = "Form = 'Invoice' and Date >= @dt('2026-01-01')";

// Com view como indice: ~1.6ms
String dql = "'(InvoiceDate)'.Date >= @dt('2026-01-01')"
    + " and Form = 'Invoice'";
```

The difference can be **1000x** on large datasets.

## Error handling

```java
.onStatus(status -> status.isError(),
    response -> response.bodyToMono(String.class)
        .flatMap(body -> {
            log.error("Erro HTTP na DQL: {}", body);
            return Mono.error(new RuntimeException(
                "Erro ao executar query DQL: " + body));
        }))
```

Common DRAPI errors in DQL:
- `400`: invalid DQL syntax
- `401`: expired token
- `404`: referenced view does not exist
- `500`: timeout on very large queries

## Lessons learned

| Challenge | Solution |
|---|---|
| Huge payload without projection | `&fields=` parameter reduces size by 95% |
| Commas in fields encoded incorrectly | String.format() instead of URI builder |
| "mode" ignored | Include "mode" in the POST body (required) |
| N+1 queries to check existence | DQL with OR in a single call |
| Sequential scan too slow | Use views as indexes: `'(ViewName)'.Field` |
| DQL injection | `.replace("'", "''")` on interpolated values |
| Huge responses | 100MB buffer in WebClient for DQL |

---

DQL is the most powerful tool in DRAPI for complex queries. Combined with field projection and views as indexes, you can achieve performance comparable to SQL queries on large data volumes.
