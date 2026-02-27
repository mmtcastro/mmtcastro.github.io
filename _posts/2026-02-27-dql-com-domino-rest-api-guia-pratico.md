---
layout: post
title: "DQL com Domino REST API: um guia pratico de quem apanhou ate funcionar"
date: 2026-02-27
categories: [domino, drapi, dql, spring-boot]
---

Quando decidimos integrar nossa aplicacao Spring Boot com o HCL Domino via Domino REST API (DRAPI), uma das primeiras necessidades foi executar consultas avancadas usando **Domino Query Language (DQL)**. A documentacao oficial cobre o basico, mas na pratica encontramos uma serie de detalhes que so descobrimos por tentativa e erro.

Este post reune tudo o que aprendemos para que voce nao precise repetir nossos erros.

## O endpoint /query

O DRAPI expoe o endpoint `POST /query` para executar consultas DQL. A estrutura basica e:

```
POST /api/v1/query?dataSource={scope}&action=execute&mode=default&fields={campos}
```

Com o body:

```json
{
  "query": "Form = 'Customer' and Status = 'Active'",
  "mode": "default"
}
```

E o header de autenticacao:

```
Authorization: Bearer {seu-jwt-token}
Content-Type: application/json
```

Parece simples, certo? Na pratica, existem varios detalhes que fazem a diferenca entre funcionar e retornar erros obscuros.

## Descoberta 1: o "mode" no body e obrigatorio

Este foi nosso primeiro obstaculo. Sem o campo `"mode": "default"` no body da requisicao, o DRAPI nao serializa os campos corretamente. A resposta pode vir vazia ou com campos faltando.

```java
// NAO funciona corretamente - campos podem vir vazios
Map<String, Object> payload = Map.of("query", dql);

// FUNCIONA - mode no body e obrigatorio
Map<String, Object> payload = Map.of("query", dql, "mode", "default");
```

Isso nao estava claro na documentacao quando comecamos. O `mode` precisa estar **tanto na URL quanto no body**.

## Descoberta 2: virgulas no parametro fields causam problemas

Quando voce quer limitar os campos retornados (essencial para performance), o parametro `fields` recebe uma lista separada por virgulas:

```
/query?dataSource=invoices&action=execute&mode=default&fields=Date,Name,Total,Status
```

O problema: se voce usa URI template variables do Spring WebClient, as virgulas sao codificadas como `%2C`, e o DRAPI nao interpreta corretamente.

```java
// NAO funciona - WebClient codifica virgulas como %2C
webClient.post()
    .uri("/query?dataSource={ds}&fields={fields}", scope, "Date,Name,Total")

// FUNCIONA - String.format preserva as virgulas
String uri = String.format(
    "/query?dataSource=%s&action=execute&mode=%s&fields=%s",
    scope, mode, "Date,Name,Total");
webClient.post().uri(uri)
```

Essa descoberta nos custou horas de debug. A resposta voltava `200 OK` mas sem os campos solicitados.

## Descoberta 3: sintaxe de datas com @dt()

O DQL usa a funcao `@dt()` para comparacoes de data. O formato aceito e ISO com horario:

```java
String dql = String.format(
    "Form = 'Invoice' and InvoiceDate >= @dt('%sT00:00:00') and InvoiceDate <= @dt('%sT23:59:59')",
    startDate.toString(), endDate.toString());
```

Pontos importantes:
- O formato e `@dt('YYYY-MM-DDThh:mm:ss')`
- Para cobrir o dia inteiro, use `T00:00:00` no inicio e `T23:59:59` no fim
- Aspas simples dentro da funcao `@dt()` sao obrigatorias

## Descoberta 4: qualificar campos com nome da view muda tudo na performance

Esta foi a descoberta mais impactante. O DQL permite referenciar campos qualificados pela view:

```java
// SEM qualificacao de view - ~3100ms
String dql = "InvoiceDate >= @dt('2026-01-01T00:00:00') and InvoiceDate <= @dt('2026-12-31T23:59:59')";

// COM qualificacao de view - ~1.6ms (quase 2000x mais rapido!)
String dql = "'(InvoicesByDate)'.InvoiceDate >= @dt('2026-01-01T00:00:00') and '(InvoicesByDate)'.InvoiceDate <= @dt('2026-12-31T23:59:59')";
```

A diferenca e brutal. Quando voce qualifica o campo com o nome da view entre aspas simples e parenteses, o Domino usa o indice da view em vez de fazer full scan.

**Sintaxe**: `'(NomeDaView)'.NomeDoCampo`

Sempre que possivel, crie views indexadas no Domino para os campos que voce consulta via DQL e use a qualificacao.

## Descoberta 5: filtre sempre pelo Form

Mesmo usando views qualificadas, inclua `Form = 'NomeDoForm'` na query:

```java
String dql = "'(InvoicesByDate)'.InvoiceDate >= @dt('2026-01-01T00:00:00') "
    + "and '(InvoicesByDate)'.InvoiceDate <= @dt('2026-12-31T23:59:59') "
    + "and Form = 'Invoice'";
```

Sem esse filtro, a query pode retornar documentos de outros forms que estejam na mesma view, causando erros de desserializacao no lado Java.

## Descoberta 6: OR conditions em vez de IN

O DQL nao tem operador `IN`. Para buscar multiplos valores, use `OR`:

```java
// Construindo OR conditions dinamicamente
String conditions = codes.stream()
    .map(c -> "'(ViewName)'.Code = '" + c.replace("'", "''") + "'")
    .collect(Collectors.joining(" or "));

String dql = "Form = 'Deal' and Status = 'Closed' and (" + conditions + ")";
```

Note o `replace("'", "''")` para escapar aspas simples nos valores — essencial para evitar SQL injection equivalente no DQL.

## Codigo completo: servico de consulta DQL com Spring WebClient

Aqui esta o padrao que chegamos apos todas as iteracoes:

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
        // 1. Montar DQL com view qualificada + filtro de Form
        String dql = String.format(
            "'(InvoicesByDate)'.InvoiceDate >= @dt('%sT00:00:00') "
            + "and '(InvoicesByDate)'.InvoiceDate <= @dt('%sT23:59:59') "
            + "and Form = 'Invoice'",
            startDate.toString(), endDate.toString());

        // 2. Body DEVE conter "mode"
        Map<String, Object> payload = Map.of("query", dql, "mode", mode);

        // 3. URI via String.format para evitar encoding de virgulas
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

## Dica extra: buffer de resposta para grandes volumes

Se sua consulta DQL retorna muitos documentos, o WebClient pode estourar o buffer padrao. Configure um buffer maior:

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

## Dica extra: selecione apenas os campos necessarios

A diferenca de payload entre trazer o documento completo e trazer apenas os campos necessarios pode ser enorme. Em nosso caso, um documento de fatura completo tem ~2.776 bytes. Selecionando apenas 5 campos para uma tela de listagem, cada documento cai para ~200 bytes. Em uma consulta que retorna 1.000 documentos, isso e a diferenca entre 2.7MB e 200KB.

```java
// Campos minimos para a tela de listagem
private static final String DQL_FIELDS = "InvoiceDate,CustomerName,Total,Status,Type";

// Documento completo - carregado apenas quando o usuario abre o detalhe
Response<Invoice> full = findByUnid(unid);
```

## Resumo das licoes aprendidas

| Problema | Solucao |
|---|---|
| Campos vazios na resposta | Incluir `"mode": "default"` no body |
| Virgulas codificadas como %2C | Usar `String.format()` em vez de URI templates |
| Queries lentas (segundos) | Qualificar campos com view: `'(View)'.Field` |
| Documentos indesejados no resultado | Sempre incluir `Form = 'NomeDoForm'` |
| Sem operador IN | Usar OR conditions com `Collectors.joining(" or ")` |
| Buffer overflow em grandes respostas | Configurar `maxInMemorySize` no WebClient |
| Payload excessivo | Usar parametro `fields` para limitar campos retornados |

---

Estas foram as principais descobertas da nossa jornada com DQL e Domino REST API. Se voce esta no mesmo caminho, esperamos que este guia economize algumas horas do seu tempo. Nos proximos posts, vamos abordar autenticacao JWT via DRAPI e como reproduzir o modelo de ACL do Domino em Spring Security.
