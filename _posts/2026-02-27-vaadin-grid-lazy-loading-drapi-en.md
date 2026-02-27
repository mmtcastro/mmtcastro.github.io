---
layout: post
title: "Lazy loading Vaadin Grids with Domino REST API"
date: 2026-02-27
categories: [vaadin, drapi, spring-boot, performance]
lang: en
---

When you have a Domino database with thousands of documents and need to display them in a grid, loading everything at once is not an option. Vaadin offers a **lazy loading** mechanism via `DataProvider.fromCallbacks()` that loads only the records visible on screen. The Domino REST API supports server-side pagination with the `start` and `count` parameters. The question is: how do you connect the two?

This post explains how we made this integration work, including the details the documentation does not cover.

## The problem

Imagine a Domino view with 10,000 documents. Without lazy loading, you would need to:

1. Load all 10,000 documents in a single HTTP call
2. Transfer megabytes of JSON over the network
3. Keep all objects in memory on the Java server
4. The user waits seconds (or minutes) before seeing any data

With lazy loading, the grid loads only **50 documents at a time** — exactly what fits on screen. As the user scrolls down, the grid fetches the next page automatically.

## How Vaadin's DataProvider works

The Vaadin Grid uses a `DataProvider` to obtain data. For lazy loading, we use `DataProvider.fromCallbacks()` with two callbacks:

- **Fetch callback**: receives `offset` and `limit`, returns the records for that page
- **Count callback**: returns the total number of records (to calculate the scrollbar)

```java
DataProvider<MyEntity, Void> dataProvider = DataProvider.fromCallbacks(
    // Fetch: "give me records from offset to offset+limit"
    query -> {
        int offset = query.getOffset();  // e.g.: 0, 50, 100, 150...
        int limit = query.getLimit();    // e.g.: 50
        return service.findAll(offset, limit).stream();
    },
    // Count: "how many records exist in total?"
    query -> service.getTotalCount()
);

grid.setDataProvider(dataProvider);
```

When the user opens the page, Vaadin calls fetch with `offset=0, limit=50`. When they scroll down, it calls with `offset=50, limit=50`, then `offset=100, limit=50`, and so on.

## Mapping to DRAPI parameters

The Domino REST API `/lists` endpoint accepts pagination parameters:

```
GET /api/v1/lists/{viewName}?dataSource={scope}&start={offset}&count={limit}
```

| Vaadin Grid | DRAPI /lists | Description |
|---|---|---|
| `query.getOffset()` | `start` | Starting position (0-based) |
| `query.getLimit()` | `count` | Number of records per page |
| Sort column | `column` | Sort column name |
| Sort direction | `direction` | `asc` or `desc` |

The mapping is straightforward — `offset` becomes `start`, `limit` becomes `count`:

```java
public List<T> findAll(int offset, int count, List<QuerySortOrder> sortOrders,
                       String search, boolean fulltextsearch) {

    String column = "Codigo";      // Default sort column
    String direction = "asc";

    if (sortOrders != null && !sortOrders.isEmpty()) {
        QuerySortOrder sort = sortOrders.get(0);
        direction = sort.getDirection() == SortDirection.ASCENDING ? "asc" : "desc";
    }

    String searchParam = "";
    if (search != null && !search.isBlank()) {
        if (fulltextsearch) {
            searchParam = "&ftSearchQuery=" + search;
        } else {
            searchParam = "&startsWith=" + search;
        }
    }

    String uri = String.format(
        "/lists/%s?dataSource=%s&mode=%s&count=%d&start=%d&column=%s&direction=%s%s",
        viewName, scope, mode, count, offset, column, direction, searchParam);

    // Make the HTTP call and parse the JSON...
}
```

## The x-totalcount header: the missing piece

Here is the crucial detail: **Vaadin needs to know the total number of records** to render the scrollbar correctly. Without it, the grid does not know when to stop requesting more data.

DRAPI returns this total in the HTTP header `x-totalcount`:

```
HTTP/1.1 200 OK
Content-Type: application/json
x-totalcount: 3847

[{...}, {...}, {...}, ...]
```

We capture this header from the response:

```java
ResponseEntity<String> entity = webClient.get()
    .uri(uri, params)
    .header("Authorization", "Bearer " + getUserToken())
    .retrieve()
    .toEntity(String.class)  // toEntity to access headers
    .block();

// Capture total record count from header
String totalCountHeader = entity.getHeaders().getFirst("x-totalcount");
if (totalCountHeader != null) {
    this.totalCount = Integer.parseInt(totalCountHeader);
}

// Parse the body normally
String rawResponse = entity.getBody();
List<T> result = objectMapper.readValue(rawResponse,
    objectMapper.getTypeFactory().constructCollectionType(List.class, modelClass));
```

Note the use of `.toEntity(String.class)` instead of `.bodyToMono(String.class)`. This is necessary to access both the response headers and the body.

The `totalCount` is stored in the service and accessed by the grid's count callback:

```java
// In the service
protected Integer totalCount;

public Integer getTotalCount() {
    return totalCount;
}
```

## The complete view: connecting everything

Here is the full pattern we use in all list views:

```java
@Route(value = "invoices", layout = MainLayout.class)
public class InvoicesView extends VerticalLayout {

    private final InvoiceService service;
    private final Grid<Invoice> grid = new Grid<>(Invoice.class, false);
    private final TextField searchField = new TextField("Search");

    public InvoicesView(InvoiceService service) {
        this.service = service;
        setSizeFull();

        configureSearch();
        configureGrid();
        add(createToolbar(), grid);

        // Initial load
        updateGrid("", false);
    }

    private void configureSearch() {
        searchField.setPlaceholder("search...");
        searchField.setClearButtonVisible(true);

        // Auto-search while typing (prefix)
        searchField.setValueChangeMode(ValueChangeMode.LAZY);
        searchField.addValueChangeListener(e -> {
            String term = e.getValue() != null ? e.getValue().trim() : "";
            updateGrid(term, false);  // startsWith
        });

        // ENTER triggers fulltext search
        searchField.addKeyPressListener(Key.ENTER, event -> {
            String term = searchField.getValue() != null ? searchField.getValue().trim() : "";
            if (!term.isEmpty()) {
                updateGrid(term, true);  // ftSearchQuery
            }
        });
    }

    private void configureGrid() {
        grid.addThemeVariants(GridVariant.LUMO_ROW_STRIPES);
        grid.setSizeFull();

        grid.addColumn(Invoice::getCodigo).setHeader("Code").setSortable(true);
        grid.addColumn(Invoice::getCustomerName).setHeader("Customer");
        grid.addColumn(Invoice::getStatus).setHeader("Status");
        grid.addColumn(invoice -> {
            if (invoice.getTotal() != null) {
                return NumberFormat.getCurrencyInstance(Locale.US)
                    .format(invoice.getTotal());
            }
            return "";
        }).setHeader("Total");
    }

    public void updateGrid(String searchText, boolean fulltextsearch) {
        // 1. Initial fetch to populate totalCount in the service
        service.findAll(0, 50, List.of(), searchText, fulltextsearch);

        // 2. Create DataProvider with lazy loading
        DataProvider<Invoice, Void> dataProvider = DataProvider.fromCallbacks(
            query -> {
                int offset = query.getOffset();
                int limit = query.getLimit();
                List<QuerySortOrder> sortOrders = query.getSortOrders();

                int totalCount = Optional.ofNullable(service.getTotalCount()).orElse(0);

                if (totalCount == 0) {
                    return Stream.empty();
                }

                // Adjust limit to not exceed total
                int adjustedLimit = Math.min(limit, totalCount - offset);
                if (adjustedLimit <= 0) {
                    return Stream.empty();
                }

                return service.findAll(offset, adjustedLimit, sortOrders,
                    searchText, fulltextsearch).stream();
            },
            query -> Optional.ofNullable(service.getTotalCount()).orElse(0)
        );

        grid.setDataProvider(dataProvider);
        grid.getDataProvider().refreshAll();
    }
}
```

## Two search modes: prefix and fulltext

DRAPI offers two search mechanisms that we integrated with the search field:

| Mode | DRAPI parameter | Triggered by | Behavior |
|---|---|---|---|
| Prefix | `startsWith=comp` | Typing (LAZY) | Filters by the index column |
| Fulltext | `ftSearchQuery=computers` | Pressing ENTER | Searches across all fields |

```properties
# Prefix: searches the view's index column (fast)
/lists/Invoices?...&startsWith=comp

# Fulltext: searches all document fields (more comprehensive)
/lists/Invoices?...&ftSearchQuery=computers
```

An important detail: **when fulltext search is active, the `x-totalcount` header may not be returned**. In that case, we use the returned list size as a fallback:

```java
if (search == null || search.isBlank()) {
    // No filter: use x-totalcount
    if (totalCountHeader != null) {
        this.totalCount = Integer.parseInt(totalCountHeader);
    }
} else {
    // With filter: use response size
    this.totalCount = result.size();
}
```

## The limit adjustment: avoiding unnecessary requests

A detail that prevents errors: when offset + limit exceeds the total number of records, we need to adjust:

```java
int adjustedLimit = Math.min(limit, totalCount - offset);
if (adjustedLimit <= 0) {
    return Stream.empty();
}
```

Without this adjustment, DRAPI may return an empty array or error, and the grid may loop trying to load more data.

## Visual flow

```
User opens the page
    |
    v
updateGrid("", false)
    |
    ├── service.findAll(0, 50, ...)  ← Initial fetch: populates totalCount
    |       |
    |       └── DRAPI: /lists/View?start=0&count=50
    |                  Response: [...50 docs...]
    |                  Header: x-totalcount: 3847
    |                  → this.totalCount = 3847
    |
    └── DataProvider.fromCallbacks(fetch, count)
            |
            ├── count() → 3847  ← Vaadin knows the scrollbar size
            |
            └── fetch(offset=0, limit=50) → [...50 docs...]
                    ← Grid renders the first 50 records

User scrolls down
    |
    └── fetch(offset=50, limit=50)
            |
            └── DRAPI: /lists/View?start=50&count=50
                       → [...next 50 docs...]

User scrolls more
    |
    └── fetch(offset=100, limit=50)
            |
            └── DRAPI: /lists/View?start=100&count=50
                       → [...next 50 docs...]

...and so on until offset 3847
```

## Lessons learned

| Challenge | Solution |
|---|---|
| Grid does not know when to stop loading | Capture `x-totalcount` from DRAPI header |
| Using `.bodyToMono()` loses the headers | Use `.toEntity()` to access headers + body |
| Fulltext search does not return x-totalcount | Fallback to `result.size()` |
| Offset exceeds totalCount | Adjust limit with `Math.min(limit, total - offset)` |
| Grid sorting | Map `QuerySortOrder` to `column` + `direction` in DRAPI |
| Search needs two modes | LAZY → `startsWith` (prefix), ENTER → `ftSearchQuery` (fulltext) |
| Initial call for totalCount | First fetch with `offset=0, limit=50` before creating the DataProvider |

---

In the next post, we will explore how we implemented the Domino ACL (Access Control List) model in Spring Security, controlling view, edit, and delete permissions per user and group.
