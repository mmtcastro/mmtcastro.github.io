---
layout: post
title: "Lazy loading no Vaadin Grid com Domino REST API"
date: 2026-02-27
categories: [vaadin, drapi, spring-boot, performance]
---

Quando voce tem uma base Domino com milhares de documentos e precisa exibi-los em um grid, carregar tudo de uma vez nao e opcao. O Vaadin oferece um mecanismo de **lazy loading** via `DataProvider.fromCallbacks()` que carrega apenas os registros visiveis na tela. O Domino REST API suporta paginacao server-side com os parametros `start` e `count`. A questao e: como conectar os dois?

Este post explica como fizemos essa integracao funcionar, incluindo os detalhes que a documentacao nao cobre.

## O problema

Imagine uma view do Domino com 10.000 documentos. Sem lazy loading, voce precisaria:

1. Carregar todos os 10.000 documentos em uma unica chamada HTTP
2. Transferir megabytes de JSON pela rede
3. Manter todos os objetos em memoria no servidor Java
4. O usuario espera segundos (ou minutos) ate ver qualquer dado

Com lazy loading, o grid carrega apenas **50 documentos por vez** — exatamente o que cabe na tela. Conforme o usuario rola para baixo, o grid busca a proxima pagina automaticamente.

## Como funciona o DataProvider do Vaadin

O Vaadin Grid usa um `DataProvider` para obter dados. Para lazy loading, usamos `DataProvider.fromCallbacks()` com dois callbacks:

- **Fetch callback**: recebe `offset` e `limit`, retorna os registros daquela pagina
- **Count callback**: retorna o total de registros (para calcular a barra de rolagem)

```java
DataProvider<MyEntity, Void> dataProvider = DataProvider.fromCallbacks(
    // Fetch: "me de os registros de offset ate offset+limit"
    query -> {
        int offset = query.getOffset();  // ex: 0, 50, 100, 150...
        int limit = query.getLimit();    // ex: 50
        return service.findAll(offset, limit).stream();
    },
    // Count: "quantos registros existem no total?"
    query -> service.getTotalCount()
);

grid.setDataProvider(dataProvider);
```

Quando o usuario abre a tela, o Vaadin chama o fetch com `offset=0, limit=50`. Quando rola para baixo, chama com `offset=50, limit=50`, depois `offset=100, limit=50`, e assim por diante.

## Mapeamento para os parametros do DRAPI

O endpoint `/lists` do Domino REST API aceita parametros de paginacao:

```
GET /api/v1/lists/{viewName}?dataSource={scope}&start={offset}&count={limit}
```

| Vaadin Grid | DRAPI /lists | Descricao |
|---|---|---|
| `query.getOffset()` | `start` | Posicao inicial (base 0) |
| `query.getLimit()` | `count` | Quantidade de registros por pagina |
| Sort column | `column` | Coluna de ordenacao |
| Sort direction | `direction` | `asc` ou `desc` |

O mapeamento e direto — `offset` vira `start`, `limit` vira `count`:

```java
public List<T> findAll(int offset, int count, List<QuerySortOrder> sortOrders,
                       String search, boolean fulltextsearch) {

    String column = "Codigo";      // Coluna padrao de ordenacao
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

    // Fazer a chamada HTTP e parsear o JSON...
}
```

## O header x-totalcount: a peca que faltava

Aqui esta o detalhe crucial: **o Vaadin precisa saber o total de registros** para renderizar a barra de rolagem corretamente. Sem isso, o grid nao sabe quando parar de pedir mais dados.

O DRAPI retorna esse total no header HTTP `x-totalcount`:

```
HTTP/1.1 200 OK
Content-Type: application/json
x-totalcount: 3847

[{...}, {...}, {...}, ...]
```

Capturamos esse header na resposta:

```java
ResponseEntity<String> entity = webClient.get()
    .uri(uri, params)
    .header("Authorization", "Bearer " + getUserToken())
    .retrieve()
    .toEntity(String.class)  // toEntity para ter acesso aos headers
    .block();

// Capturar o total de registros do header
String totalCountHeader = entity.getHeaders().getFirst("x-totalcount");
if (totalCountHeader != null) {
    this.totalCount = Integer.parseInt(totalCountHeader);
}

// Parsear o body normalmente
String rawResponse = entity.getBody();
List<T> result = objectMapper.readValue(rawResponse,
    objectMapper.getTypeFactory().constructCollectionType(List.class, modelClass));
```

Note o uso de `.toEntity(String.class)` em vez de `.bodyToMono(String.class)`. Isso e necessario para ter acesso aos headers da resposta alem do body.

O `totalCount` fica armazenado no service e e acessado pelo count callback do grid:

```java
// No service
protected Integer totalCount;

public Integer getTotalCount() {
    return totalCount;
}
```

## A view completa: conectando tudo

Aqui esta o padrao completo que usamos em todas as views de listagem:

```java
@Route(value = "invoices", layout = MainLayout.class)
public class InvoicesView extends VerticalLayout {

    private final InvoiceService service;
    private final Grid<Invoice> grid = new Grid<>(Invoice.class, false);
    private final TextField searchField = new TextField("Buscar");

    public InvoicesView(InvoiceService service) {
        this.service = service;
        setSizeFull();

        configureSearch();
        configureGrid();
        add(createToolbar(), grid);

        // Carga inicial
        updateGrid("", false);
    }

    private void configureSearch() {
        searchField.setPlaceholder("buscar...");
        searchField.setClearButtonVisible(true);

        // Busca automatica ao digitar (prefixo)
        searchField.setValueChangeMode(ValueChangeMode.LAZY);
        searchField.addValueChangeListener(e -> {
            String term = e.getValue() != null ? e.getValue().trim() : "";
            updateGrid(term, false);  // startsWith
        });

        // ENTER aciona fulltext search
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

        grid.addColumn(Invoice::getCodigo).setHeader("Codigo").setSortable(true);
        grid.addColumn(Invoice::getCustomerName).setHeader("Cliente");
        grid.addColumn(Invoice::getStatus).setHeader("Status");
        grid.addColumn(invoice -> {
            if (invoice.getTotal() != null) {
                return NumberFormat.getCurrencyInstance(new Locale("pt", "BR"))
                    .format(invoice.getTotal());
            }
            return "";
        }).setHeader("Total");
    }

    public void updateGrid(String searchText, boolean fulltextsearch) {
        // 1. Busca inicial para popular o totalCount no service
        service.findAll(0, 50, List.of(), searchText, fulltextsearch);

        // 2. Criar DataProvider com lazy loading
        DataProvider<Invoice, Void> dataProvider = DataProvider.fromCallbacks(
            query -> {
                int offset = query.getOffset();
                int limit = query.getLimit();
                List<QuerySortOrder> sortOrders = query.getSortOrders();

                int totalCount = Optional.ofNullable(service.getTotalCount()).orElse(0);

                if (totalCount == 0) {
                    return Stream.empty();
                }

                // Ajustar limit para nao ultrapassar o total
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

## Dois modos de busca: prefixo e fulltext

O DRAPI oferece dois mecanismos de busca que integramos ao campo de pesquisa:

| Modo | Parametro DRAPI | Quando ativa | Comportamento |
|---|---|---|---|
| Prefixo | `startsWith=comp` | Digitar (LAZY) | Filtra pela coluna de indice |
| Fulltext | `ftSearchQuery=computers` | Pressionar ENTER | Busca em todos os campos |

```properties
# Prefixo: busca na coluna de indice da view (rapido)
/lists/Invoices?...&startsWith=comp

# Fulltext: busca em todos os campos do documento (mais abrangente)
/lists/Invoices?...&ftSearchQuery=computers
```

Um detalhe importante: **quando fulltext search esta ativo, o header `x-totalcount` pode nao vir na resposta**. Nesse caso, usamos o tamanho da lista retornada como fallback:

```java
if (search == null || search.isBlank()) {
    // Sem filtro: usar x-totalcount
    if (totalCountHeader != null) {
        this.totalCount = Integer.parseInt(totalCountHeader);
    }
} else {
    // Com filtro: usar tamanho da resposta
    this.totalCount = result.size();
}
```

## O ajuste de limit: evitando requisicoes desnecessarias

Um detalhe que evita erros: quando o offset + limit ultrapassa o total de registros, precisamos ajustar:

```java
int adjustedLimit = Math.min(limit, totalCount - offset);
if (adjustedLimit <= 0) {
    return Stream.empty();
}
```

Sem esse ajuste, o DRAPI pode retornar um array vazio ou erro, e o grid pode ficar em loop tentando carregar mais dados.

## Fluxo visual

```
Usuario abre a tela
    |
    v
updateGrid("", false)
    |
    ├── service.findAll(0, 50, ...)  ← Busca inicial: popula totalCount
    |       |
    |       └── DRAPI: /lists/View?start=0&count=50
    |                  Response: [...50 docs...]
    |                  Header: x-totalcount: 3847
    |                  → this.totalCount = 3847
    |
    └── DataProvider.fromCallbacks(fetch, count)
            |
            ├── count() → 3847  ← Vaadin sabe o tamanho da barra de rolagem
            |
            └── fetch(offset=0, limit=50) → [...50 docs...]
                    ← Grid renderiza os 50 primeiros registros

Usuario rola para baixo
    |
    └── fetch(offset=50, limit=50)
            |
            └── DRAPI: /lists/View?start=50&count=50
                       → [...proximos 50 docs...]

Usuario rola mais
    |
    └── fetch(offset=100, limit=50)
            |
            └── DRAPI: /lists/View?start=100&count=50
                       → [...proximos 50 docs...]

...e assim por diante ate o offset 3847
```

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Grid nao sabe quando parar de carregar | Capturar `x-totalcount` do header DRAPI |
| Usar `.bodyToMono()` perde os headers | Usar `.toEntity()` para acessar headers + body |
| Fulltext search nao retorna x-totalcount | Fallback para `result.size()` |
| Offset ultrapassa totalCount | Ajustar limit com `Math.min(limit, total - offset)` |
| Ordenacao no grid | Mapear `QuerySortOrder` para `column` + `direction` no DRAPI |
| Busca precisa de dois modos | LAZY → `startsWith` (prefixo), ENTER → `ftSearchQuery` (fulltext) |
| Chamada inicial para totalCount | Primeira busca com `offset=0, limit=50` antes de criar o DataProvider |

---

No proximo post, vamos explorar como implementamos o modelo de ACL (Access Control List) do Domino no Spring Security, controlando permissoes de visualizacao, edicao e exclusao por usuario e grupo.
