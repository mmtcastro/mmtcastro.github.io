---
layout: post
title: "DQL with Domino REST API: a practical guide from real-world experience"
date: 2026-02-27
categories: [domino, drapi, dql, spring-boot]
lang: en
---

When we decided to integrate our Spring Boot application with HCL Domino via the Domino REST API (DRAPI), one of our first needs was to execute advanced queries using **Domino Query Language (DQL)**. The official documentation covers the basics, but in practice we ran into a series of details that we only discovered through trial and error.

This post brings together everything we learned so you don't have to repeat our mistakes.

## The /query endpoint

DRAPI exposes the `POST /query` endpoint for executing DQL queries. The basic structure is:

```
POST /api/v1/query?dataSource={scope}&action=execute&mode=default&fields={fields}
```

With the body:

```json
{
  "query": "Form = 'Customer' and Status = 'Active'",
  "mode": "default"
}
```

And the authentication header:

```
Authorization: Bearer {your-jwt-token}
Content-Type: application/json
```

Seems straightforward, right? In practice, there are several details that make the difference between success and obscure errors.

## Lesson 1: "mode" in the request body is mandatory

This was our first obstacle. Without the `"mode": "default"` field in the request body, DRAPI does not serialize fields correctly. The response may come back empty or with missing fields.

```java
// DOES NOT work correctly - fields may come back empty
Map<String, Object> payload = Map.of("query", dql);

// WORKS - mode in the body is mandatory
Map<String, Object> payload = Map.of("query", dql, "mode", "default");
```

This was not clear in the documentation when we started. The `mode` needs to be in **both the URL and the body**.

## Lesson 2: commas in the fields parameter cause problems

When you want to limit the returned fields (essential for performance), the `fields` parameter takes a comma-separated list:

```
/query?dataSource=invoices&action=execute&mode=default&fields=Date,Name,Total,Status
```

The problem: if you use Spring WebClient's URI template variables, commas get encoded as `%2C`, and DRAPI does not interpret them correctly.

```java
// DOES NOT work - WebClient encodes commas as %2C
webClient.post()
    .uri("/query?dataSource={ds}&fields={fields}", scope, "Date,Name,Total")

// WORKS - String.format preserves the commas
String uri = String.format(
    "/query?dataSource=%s&action=execute&mode=%s&fields=%s",
    scope, mode, "Date,Name,Total");
webClient.post().uri(uri)
```

This discovery cost us hours of debugging. The response came back `200 OK` but without the requested fields.

## Lesson 3: date syntax with @dt()

DQL uses the `@dt()` function for date comparisons. The accepted format is ISO with time:

```java
String dql = String.format(
    "Form = 'Invoice' and InvoiceDate >= @dt('%sT00:00:00') and InvoiceDate <= @dt('%sT23:59:59')",
    startDate.toString(), endDate.toString());
```

Key points:
- The format is `@dt('YYYY-MM-DDThh:mm:ss')`
- To cover an entire day, use `T00:00:00` for the start and `T23:59:59` for the end
- Single quotes inside the `@dt()` function are mandatory

## Lesson 4: qualifying fields with view names changes everything for performance

This was our most impactful discovery. DQL allows you to reference fields qualified by a view name:

```java
// WITHOUT view qualification - ~3100ms
String dql = "InvoiceDate >= @dt('2026-01-01T00:00:00') and InvoiceDate <= @dt('2026-12-31T23:59:59')";

// WITH view qualification - ~1.6ms (nearly 2000x faster!)
String dql = "'(InvoicesByDate)'.InvoiceDate >= @dt('2026-01-01T00:00:00') and '(InvoicesByDate)'.InvoiceDate <= @dt('2026-12-31T23:59:59')";
```

The difference is dramatic. When you qualify a field with the view name in single quotes and parentheses, Domino uses the view's index instead of doing a full scan.

**Syntax**: `'(ViewName)'.FieldName`

Whenever possible, create indexed views in Domino for the fields you query via DQL and use qualification.

## Lesson 5: always filter by Form

Even when using qualified views, include `Form = 'FormName'` in your query:

```java
String dql = "'(InvoicesByDate)'.InvoiceDate >= @dt('2026-01-01T00:00:00') "
    + "and '(InvoicesByDate)'.InvoiceDate <= @dt('2026-12-31T23:59:59') "
    + "and Form = 'Invoice'";
```

Without this filter, the query may return documents from other forms that happen to be in the same view, causing deserialization errors on the Java side.

## Lesson 6: OR conditions instead of IN

DQL does not have an `IN` operator. To search for multiple values, use `OR`:

```java
// Building OR conditions dynamically
String conditions = codes.stream()
    .map(c -> "'(ViewName)'.Code = '" + c.replace("'", "''") + "'")
    .collect(Collectors.joining(" or "));

String dql = "Form = 'Deal' and Status = 'Closed' and (" + conditions + ")";
```

Note the `replace("'", "''")` to escape single quotes in values — essential to prevent the DQL equivalent of SQL injection.

## Complete code: DQL query service with Spring WebClient

Here is the pattern we arrived at after all our iterations:

```java
@Service
public class InvoiceService {

    private static final String DQL_FIELDS = "InvoiceDate,CustomerName,Total,Status,Type";

    @Autowired
    private WebClient webClient;

    @Autowired
    private ObjectMapper objectMapper;

    private String scope = "invoices";
    private String mode = "default";

    public List<Invoice> findByDateRange(LocalDate startDate, LocalDate endDate) {
        // 1. Build DQL with view qualification + Form filter
        String dql = String.format(
            "'(InvoicesByDate)'.InvoiceDate >= @dt('%sT00:00:00') "
            + "and '(InvoicesByDate)'.InvoiceDate <= @dt('%sT23:59:59') "
            + "and Form = 'Invoice'",
            startDate.toString(), endDate.toString());

        // 2. Body MUST contain "mode"
        Map<String, Object> payload = Map.of("query", dql, "mode", mode);

        // 3. URI via String.format to avoid comma encoding
        String uri = String.format(
            "/query?dataSource=%s&action=execute&mode=%s&fields=%s",
            scope, mode, DQL_FIELDS);

        try {
            String rawResponse = webClient.post()
                .uri(uri)
                .header("Accept", "application/json")
                .header("Content-Type", "application/json")
                .header("Authorization", "Bearer " + getToken())
                .bodyValue(payload)
                .retrieve()
                .onStatus(status -> status.isError(),
                    response -> response.bodyToMono(String.class)
                        .flatMap(body -> Mono.error(
                            new RuntimeException("DQL error: " + body))))
                .bodyToMono(String.class)
                .block();

            if (rawResponse == null || rawResponse.isBlank()) {
                return Collections.emptyList();
            }

            return objectMapper.readValue(rawResponse,
                objectMapper.getTypeFactory()
                    .constructCollectionType(List.class, Invoice.class));

        } catch (Exception e) {
            log.error("Error executing DQL query: {}", e.getMessage(), e);
            return Collections.emptyList();
        }
    }
}
```

## Bonus tip: response buffer for large volumes

If your DQL query returns many documents, the WebClient default buffer may overflow. Configure a larger buffer:

```java
@Bean
WebClient.Builder webClientBuilder() {
    ExchangeStrategies strategies = ExchangeStrategies.builder()
        .codecs(configurer -> {
            configurer.defaultCodecs().maxInMemorySize(100 * 1024 * 1024); // 100MB
        })
        .build();

    return WebClient.builder()
        .exchangeStrategies(strategies)
        .baseUrl("https://your-domino-server:8443/api/v1/");
}
```

## Bonus tip: select only the fields you need

The payload difference between fetching a complete document and fetching only the required fields can be enormous. In our case, a complete invoice document is approximately 2,776 bytes. By selecting only 5 fields for a list screen, each document drops to about 200 bytes. For a query returning 1,000 documents, that is the difference between 2.7MB and 200KB.

```java
// Minimal fields for the list screen
private static final String DQL_FIELDS = "InvoiceDate,CustomerName,Total,Status,Type";

// Full document - loaded only when the user opens the detail view
Response<Invoice> full = findByUnid(unid);
```

## Summary of lessons learned

| Problem | Solution |
|---|---|
| Empty fields in the response | Include `"mode": "default"` in the body |
| Commas encoded as %2C | Use `String.format()` instead of URI templates |
| Slow queries (seconds) | Qualify fields with a view: `'(View)'.Field` |
| Unwanted documents in results | Always include `Form = 'FormName'` |
| No IN operator | Use OR conditions with `Collectors.joining(" or ")` |
| Buffer overflow on large responses | Configure `maxInMemorySize` on WebClient |
| Excessive payload | Use the `fields` parameter to limit returned fields |

---

These were the main discoveries from our journey with DQL and Domino REST API. If you are on the same path, we hope this guide saves you a few hours. In upcoming posts, we will cover JWT authentication via DRAPI and how to reproduce the Domino ACL model in Spring Security.
