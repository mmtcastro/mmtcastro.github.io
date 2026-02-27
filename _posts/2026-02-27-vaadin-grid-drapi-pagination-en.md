---
layout: post
title: "Vaadin Grid with DRAPI lazy pagination: loading thousands of documents without blowing up memory"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, grid]
lang: en
---

When you have thousands of documents in Domino and need to display them in a Vaadin Grid, you cannot load everything into memory at once. DRAPI supports pagination via `start` and `count` parameters and returns the total number of records in the `x-totalcount` header. This post shows how to connect the Vaadin Grid to DRAPI with real lazy loading.

## The concept: DataProvider.fromCallbacks()

The Vaadin Grid supports two loading modes:
- **In-memory**: `grid.setItems(list)` -- loads everything at once
- **Lazy**: `DataProvider.fromCallbacks(fetch, count)` -- fetches on demand

For DRAPI, we use lazy mode with two callbacks:
1. **fetch**: retrieves a page of data (`offset` + `limit`)
2. **count**: returns the total number of records (to size the scrollbar)

## The service: findAllByCodigo with pagination

The method in AbstractService builds the DRAPI URI with pagination parameters and captures the `x-totalcount` header:

```java
public List<T> findAllByCodigo(int offset, int count,
        List<QuerySortOrder> sortOrders, String search,
        Class<T> model, boolean fulltextsearch) {

    List<T> resultados = new ArrayList<>();

    String column = "Codigo";
    String direction = "asc";

    // Ordenacao vinda do Grid
    if (sortOrders != null && !sortOrders.isEmpty()) {
        QuerySortOrder sort = sortOrders.get(0);
        direction = sort.getDirection() == SortDirection.ASCENDING
            ? "asc" : "desc";
    }

    // Busca: prefixo (type-ahead) ou full-text (Enter)
    String searchQuery = "";
    if (search != null && !search.trim().isEmpty()) {
        if (fulltextsearch) {
            searchQuery = "&ftSearchQuery=" + search.trim();
        } else {
            searchQuery = "&startsWith=" + search.trim();
        }
    }

    // Monta URI com paginacao
    String uri = String.format(
        "/lists/%s?dataSource=%s&mode=%s"
        + "&count=%d&start=%d"
        + "&column=%s&direction=%s%s",
        Utils.getListaNameFromModelName(model.getSimpleName()),
        scope, mode,
        count, offset,              // ← paginacao
        column, direction,          // ← ordenacao
        searchQuery);               // ← busca

    // Chamada com captura de headers
    ClientResponse clientResponse = webClient.get()
        .uri(uri)
        .header("Authorization", "Bearer " + getUserToken())
        .exchangeToMono(Mono::just)
        .block();

    String rawResponse = clientResponse.bodyToMono(String.class)
        .block();

    // Deserializa o JSON
    JavaType listType = objectMapper.getTypeFactory()
        .constructCollectionType(List.class, model);
    resultados = objectMapper.readValue(rawResponse, listType);

    // CRITICO: captura x-totalcount do header
    String totalCountHeader = clientResponse.headers()
        .header("x-totalcount")
        .stream().findFirst().orElse(null);

    if (search == null || search.isBlank()) {
        // Sem filtro: usa totalcount da API
        this.totalCount = totalCountHeader != null
            ? Integer.parseInt(totalCountHeader)
            : resultados.size();
    } else {
        // Com filtro: usa tamanho da resposta
        this.totalCount = resultados.size();
    }

    return resultados;
}
```

### The x-totalcount header

DRAPI returns the total number of available documents in the HTTP header:

```
HTTP/1.1 200 OK
x-totalcount: 5432
Content-Type: application/json

[
  { "@unid": "...", "Codigo": "EMP-001", "Nome": "..." },
  { "@unid": "...", "Codigo": "EMP-002", "Nome": "..." },
  ...  (50 registros)
]
```

The Grid uses this value to size the virtual scrollbar -- the user sees a scroll bar proportional to the 5,432 records, but only 50 are in memory.

## The view: Grid with lazy loading

The list view creates the Grid and connects it to the DataProvider:

```java
@PageTitle("Empresas")
@Route(value = "empresas", layout = MainLayout.class)
@RolesAllowed("ROLE_EVERYONE")
public class EmpresasView extends VerticalLayout {

    private final EmpresaService service;
    private final Grid<Empresa> grid = new Grid<>(Empresa.class, false);
    private final TextField searchField = new TextField();

    public EmpresasView(EmpresaService service) {
        this.service = service;
        configureGrid();
        configureSearch();
        add(createToolbar(), grid);
        setSizeFull();
        updateGrid("", false);
    }
```

### Column configuration

```java
    private void configureGrid() {
        grid.addThemeVariants(GridVariant.LUMO_ROW_STRIPES);
        grid.setSizeFull();

        // Coluna Codigo com link para o detalhe
        grid.addColumn(new ComponentRenderer<>(empresa -> {
            Anchor link = new Anchor(
                "empresa/" + empresa.getUnid(),
                empresa.getCodigo());
            link.getElement().setAttribute("router-link", true);
            return link;
        }))
        .setHeader("Codigo")
        .setSortable(true)
        .setKey("codigo")
        .setResizable(true);

        grid.addColumn(Empresa::getNome)
            .setHeader("Nome")
            .setKey("nome");

        grid.addColumn(Empresa::getStatus)
            .setHeader("Status")
            .setKey("status");

        grid.addColumn(Empresa::getEstado)
            .setHeader("UF")
            .setKey("estado");

        // Coluna de data com formatacao
        grid.addColumn(new TextRenderer<>(item ->
            item.getCriacao() != null
                ? item.getCriacao().format(
                    DateTimeFormatter.ofPattern("dd/MM/yyyy"))
                : null))
        .setHeader("Criacao")
        .setKey("criacao");
    }
```

### The DataProvider with two callbacks

```java
    public void updateGrid(String searchText,
            boolean fulltextsearch) {

        // Fetch inicial para popular totalCount
        service.findAllByCodigo(0, 50, List.of(),
            searchText, Empresa.class, fulltextsearch);

        // Cria DataProvider lazy
        DataProvider<Empresa, Void> dataProvider =
            DataProvider.fromCallbacks(

                // Callback 1: busca dados para a pagina atual
                query -> {
                    int offset = query.getOffset();
                    int limit = query.getLimit();
                    List<QuerySortOrder> sortOrders =
                        query.getSortOrders();

                    int totalCount = Optional.ofNullable(
                        service.getTotalCount()).orElse(0);

                    if (totalCount == 0) {
                        return Stream.empty();
                    }

                    // Ajusta limit se offset > total
                    int adjustedLimit = Math.min(
                        limit, totalCount - offset);
                    if (adjustedLimit <= 0) {
                        return Stream.empty();
                    }

                    return service.findAllByCodigo(
                        offset, adjustedLimit, sortOrders,
                        searchText, Empresa.class,
                        fulltextsearch).stream();
                },

                // Callback 2: retorna total para o scrollbar
                query -> Optional.ofNullable(
                    service.getTotalCount()).orElse(0)
            );

        grid.setDataProvider(dataProvider);
    }
```

## Search: prefix vs full-text

The search operates in two modes:

```java
    private void configureSearch() {
        searchField.setPlaceholder("buscar...");
        searchField.setClearButtonVisible(true);
        searchField.setValueChangeMode(ValueChangeMode.LAZY);

        // Modo 1: LAZY (type-ahead) → busca por prefixo
        // Dispara apos o usuario parar de digitar
        searchField.addValueChangeListener(e -> {
            String term = e.getValue() != null
                ? e.getValue().trim() : "";
            updateGrid(term, false);  // startsWith
        });

        // Modo 2: ENTER → busca full-text
        searchField.addKeyPressListener(Key.ENTER, event -> {
            String term = searchField.getValue() != null
                ? searchField.getValue().trim() : "";
            if (!term.isEmpty()) {
                updateGrid(term, true);  // ftSearchQuery
            }
        });
    }
```

| User action | DRAPI parameter | Search type |
|---|---|---|
| Types "TDE" and waits | `&startsWith=TDE` | Prefix (fast, view index) |
| Types "rede computadores" and presses Enter | `&ftSearchQuery=rede computadores` | Full-text (slower, more comprehensive) |

## Toolbar with search and "New" button

```java
    private HorizontalLayout createToolbar() {
        Button newButton = new Button("Nova",
            new Icon(VaadinIcon.PLUS));
        newButton.addThemeVariants(ButtonVariant.LUMO_PRIMARY);
        newButton.addClickListener(e ->
            getUI().ifPresent(ui ->
                ui.navigate("empresa/novo")));

        HorizontalLayout searchLayout = new HorizontalLayout(
            searchField, searchButton);
        HorizontalLayout toolbar = new HorizontalLayout(
            searchLayout, newButton);
        toolbar.setWidthFull();
        toolbar.expand(searchLayout);

        return toolbar;
    }
```

## Alternative: client-side filtering for small datasets

For entities with fewer than ~1,000 records, we load everything into memory and filter on the client:

```java
// Em CargosView — dataset pequeno
private void loadData() {
    cargos = cargoService.getCargos();
    grid.setItems(cargos);
}

private void filterGrid(String filterText) {
    if (filterText == null || filterText.isEmpty()) {
        grid.setItems(cargos);
    } else {
        String filter = filterText.toLowerCase();
        grid.setItems(cargos.stream()
            .filter(c ->
                (c.getCodigo() != null && c.getCodigo()
                    .toLowerCase().contains(filter))
                || (c.getDescricao() != null && c.getDescricao()
                    .toLowerCase().contains(filter)))
            .toList());
    }
}
```

### When to use each approach

| Criterion | In-memory | Lazy (DataProvider) |
|---|---|---|
| Records | < 1,000 | > 1,000 |
| Memory | Loads everything | Loads per page |
| Search | Client-side (stream.filter) | Server-side (DRAPI) |
| Sorting | Client-side | Server-side (column + direction) |
| Initial speed | Slower (loads everything) | Faster (loads 50) |

## The complete flow

```
Grid renderiza 50 linhas visiveis
  │
  ▼  DataProvider.fetch(offset=0, limit=50)
GET /lists/_intraCodigos?dataSource=empresas&count=50&start=0
   &column=Codigo&direction=asc
  │
  ▼  Resposta: 50 docs + header x-totalcount: 5432
Grid mostra scrollbar proporcional a 5432 registros
  │
  ▼  Usuario rola para baixo...
DataProvider.fetch(offset=50, limit=50)
  │
  ▼  GET .../count=50&start=50...
Proximos 50 docs carregados
  │
  ▼  Usuario clica no header "Codigo" ↓
DataProvider.fetch(offset=0, limit=50, sort=desc)
  │
  ▼  GET .../column=Codigo&direction=desc...
Primeiros 50 docs em ordem decrescente
  │
  ▼  Usuario digita "TDE" no campo de busca
DataProvider.fetch(offset=0, limit=50)
  │
  ▼  GET .../startsWith=TDE...
Resultados filtrados por prefixo
```

## Lessons learned

| Challenge | Solution |
|---|---|
| Grid loads everything into memory | DataProvider.fromCallbacks() with server-side pagination |
| Scrollbar doesn't know the total size | DRAPI's x-totalcount header sizes the virtual scroll |
| Slow search on large datasets | Two modes: prefix (fast) and full-text (comprehensive) |
| Offset greater than totalCount causes error | Math.min(limit, totalCount - offset) before the call |
| Sorting needs to be server-side | column + direction parameters in the DRAPI URI |
| Small datasets don't need lazy loading | grid.setItems() with stream.filter() for < 1,000 records |

---

With this pattern, the Vaadin Grid behaves like an infinite virtual table -- the user scrolls freely through thousands of documents, and DRAPI delivers only the necessary pages. Memory usage stays constant regardless of database size.
