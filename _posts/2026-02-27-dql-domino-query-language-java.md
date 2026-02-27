---
layout: post
title: "DQL via Java: queries avancadas no Domino REST API com projecao de campos"
date: 2026-02-27
categories: [domino, drapi, java, dql]
---

Domino Query Language (DQL) permite executar queries complexas no Domino via `POST /query`. Ao contrario das views tradicionais, DQL suporta filtros por data, combinacoes OR/AND e — criticamente — projecao de campos que reduz o payload em ate 95%. Este post mostra como executamos DQL a partir do Java com Spring WebClient.

## O endpoint: POST /query

```
POST /query?dataSource={scope}&action=execute&mode={mode}&fields={fields}
Authorization: Bearer {token}
Content-Type: application/json

{
  "query": "Form = 'Invoice' and Date >= @dt('2026-01-01T00:00:00')",
  "mode": "default"
}
```

**Parametros criticos:**
- `dataSource`: o scope do banco Domino
- `fields`: lista CSV dos campos a retornar (projecao)
- `mode` no body: **obrigatorio** para o DRAPI serializar os campos corretamente

## O metodo base: executeDql()

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

**Detalhe importante**: usamos `String.format()` em vez do URI builder do WebClient porque o builder codifica as virgulas na lista de campos (`fields=A,B,C` vira `fields=A%2CB%2CC`), o que o DRAPI nao aceita.

## Projecao de campos: 95% menos payload

Sem projecao, cada documento pode ter 2.700+ bytes. Com projecao de 5 campos, cai para ~150 bytes:

```java
// Constante com campos necessarios
private static final String DQL_FIELDS =
    "DataFaturamento,RazaoSocial,TotalNota,Sit,Natureza";

// Sem projecao: ~2776 bytes/doc × 5000 docs = 13.8 MB
// Com projecao: ~150 bytes/doc × 5000 docs = 750 KB
// Reducao: 95%
```

## Exemplo 1: filtro por periodo com log de performance

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

### Log de exemplo

```
DQL: '(InvoiceDate)'.DataFaturamento >= @dt('2026-01-01T00:00:00') and ...
Fields: DataFaturamento,RazaoSocial,TotalNota,Sit,Natureza
Resposta: 384000 chars (375 KB) em 1200ms
Resultado: 2560 docs | API=1200ms | Total=1450ms | ~150 bytes/doc
```

## Exemplo 2: batch query com OR (evita N+1)

Em vez de fazer N chamadas para verificar se cada grupo economico tem negocios, fazemos uma unica query DQL com OR:

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

**Ganho**: 1 chamada DQL em vez de N chamadas individuais. Para 500 grupos, passa de 500 requests HTTP para 1.

**Seguranca**: `.replace("'", "''")` previne injecao de DQL nos valores.

## Views do Domino como indice DQL

DQL pode usar views como indice para performance. A sintaxe `'(ViewName)'.FieldName` forca o DRAPI a usar a view como indice:

```java
// Sem view (scan sequencial): ~3100ms
String dql = "Form = 'Invoice' and Date >= @dt('2026-01-01')";

// Com view como indice: ~1.6ms
String dql = "'(InvoiceDate)'.Date >= @dt('2026-01-01')"
    + " and Form = 'Invoice'";
```

A diferenca pode ser de **1000x** em datasets grandes.

## Tratamento de erro

```java
.onStatus(status -> status.isError(),
    response -> response.bodyToMono(String.class)
        .flatMap(body -> {
            log.error("Erro HTTP na DQL: {}", body);
            return Mono.error(new RuntimeException(
                "Erro ao executar query DQL: " + body));
        }))
```

Erros comuns do DRAPI em DQL:
- `400`: sintaxe DQL invalida
- `401`: token expirado
- `404`: view referenciada nao existe
- `500`: timeout em queries muito grandes

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Payload gigante sem projecao | Parametro `&fields=` reduz 95% do tamanho |
| Virgulas nos fields codificadas incorretamente | String.format() em vez de URI builder |
| "mode" ignorado | Incluir "mode" no body do POST (obrigatorio) |
| N+1 queries para verificar existencia | DQL com OR em uma unica chamada |
| Scan sequencial muito lento | Usar views como indice: `'(ViewName)'.Field` |
| Injecao de DQL | `.replace("'", "''")` nos valores interpolados |
| Respostas gigantes | Buffer de 100MB no WebClient para DQL |

---

DQL e a ferramenta mais poderosa do DRAPI para queries complexas. Combinada com projecao de campos e views como indice, voce consegue performance comparavel a queries SQL em grandes volumes de dados.
