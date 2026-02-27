---
layout: post
title: "Vaadin Grid com paginacao lazy do DRAPI: carregando milhares de documentos sem estourar a memoria"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, grid]
---

Quando voce tem milhares de documentos no Domino e precisa exibi-los em um Grid no Vaadin, nao pode carregar tudo na memoria de uma vez. O DRAPI suporta paginacao via parametros `start` e `count`, e retorna o total de registros no header `x-totalcount`. Este post mostra como conectar o Vaadin Grid ao DRAPI com lazy loading real.

## O conceito: DataProvider.fromCallbacks()

O Vaadin Grid suporta dois modos de carregamento:
- **In-memory**: `grid.setItems(lista)` — carrega tudo de uma vez
- **Lazy**: `DataProvider.fromCallbacks(fetch, count)` — busca sob demanda

Para o DRAPI, usamos o modo lazy com dois callbacks:
1. **fetch**: busca uma pagina de dados (`offset` + `limit`)
2. **count**: retorna o total de registros (para dimensionar o scrollbar)

## O service: findAllByCodigo com paginacao

O metodo no AbstractService constroi a URI do DRAPI com parametros de paginacao e captura o header `x-totalcount`:

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

### O header x-totalcount

O DRAPI retorna o total de documentos disponiveis no header HTTP:

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

O Grid usa esse valor para dimensionar o scrollbar virtual — o usuario ve a barra de rolagem proporcional aos 5.432 registros, mas so 50 estao na memoria.

## A view: Grid com lazy loading

A view de listagem cria o Grid e conecta ao DataProvider:

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

### Configuracao das colunas

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

### O DataProvider com dois callbacks

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

## Busca: prefixo vs full-text

A busca opera em dois modos:

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

| Acao do usuario | Parametro DRAPI | Tipo de busca |
|---|---|---|
| Digita "TDE" e espera | `&startsWith=TDE` | Prefixo (rapido, view index) |
| Digita "rede computadores" e pressiona Enter | `&ftSearchQuery=rede computadores` | Full-text (mais lento, mais abrangente) |

## Toolbar com busca e botao "Novo"

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

## Alternativa: filtro client-side para datasets pequenos

Para entidades com menos de ~1.000 registros, carregamos tudo na memoria e filtramos no cliente:

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

### Quando usar cada abordagem

| Criterio | In-memory | Lazy (DataProvider) |
|---|---|---|
| Registros | < 1.000 | > 1.000 |
| Memoria | Carrega tudo | Carrega por pagina |
| Busca | Client-side (stream.filter) | Server-side (DRAPI) |
| Ordenacao | Client-side | Server-side (column + direction) |
| Velocidade inicial | Mais lenta (carrega tudo) | Mais rapida (carrega 50) |

## O fluxo completo

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

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Grid carrega tudo na memoria | DataProvider.fromCallbacks() com paginacao server-side |
| Scrollbar nao sabe o tamanho total | Header x-totalcount do DRAPI dimensiona o scroll virtual |
| Busca lenta em grandes datasets | Dois modos: prefixo (rapido) e full-text (abrangente) |
| Offset maior que totalCount causa erro | Math.min(limit, totalCount - offset) antes da chamada |
| Ordenacao precisa ser server-side | Parametros column + direction na URI do DRAPI |
| Datasets pequenos nao precisam de lazy | grid.setItems() com stream.filter() para < 1.000 registros |

---

Com esse pattern, o Vaadin Grid se comporta como uma tabela virtual infinita — o usuario rola livremente por milhares de documentos, e o DRAPI entrega apenas as paginas necessarias. A memoria fica constante independente do tamanho do banco.
