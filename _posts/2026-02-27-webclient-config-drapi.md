---
layout: post
title: "Configurando Spring WebClient para o Domino REST API: buffer, error handling e token JWT"
date: 2026-02-27
categories: [domino, drapi, java, spring, webclient]
---

O Spring WebClient e o cliente HTTP reativo que usamos para toda comunicacao com o DRAPI. Mas a configuracao padrao nao funciona bem com o Domino — respostas grandes de DQL estouram o buffer de 1MB, erros precisam ser mapeados para exceptions tipadas, e o token JWT precisa ser injetado em cada request. Este post mostra como configuramos o WebClient para o DRAPI.

## O problema do buffer padrao

O WebClient do Spring tem um buffer padrao de **256KB** (ou 1MB dependendo da versao). Quando uma query DQL retorna 50MB de JSON, voce recebe:

```
org.springframework.core.io.buffer.DataBufferLimitException:
Exceeded limit on max bytes to buffer: 262144
```

## Configuracao do WebClient: buffer de 100MB

```java
@Configuration
public class WebClientConfig {

    @Bean
    WebClient.Builder webClientBuilder(
            WebClientProperties properties) {

        // Buffer de 100MB para respostas grandes do DRAPI
        ExchangeStrategies strategies =
            ExchangeStrategies.builder()
                .codecs(configurer -> configurer
                    .defaultCodecs()
                    .maxInMemorySize(
                        100 * 1024 * 1024)) // 100MB
                .build();

        return WebClient.builder()
            .exchangeStrategies(strategies)
            .baseUrl(properties.getBaseUrl());
    }

    @Bean
    WebClient webClient(WebClient.Builder builder) {
        return builder.build();
    }
}
```

**Por que 100MB?** Queries DQL sem projecao de campos podem retornar payloads enormes (81MB em producao para 5000 documentos completos). Com projecao de campos, raramente passa de 1MB — mas o buffer grande e uma rede de seguranca.

## Properties de configuracao

```java
@Validated
@Component
@ConfigurationProperties(prefix = "webclient.properties")
public class WebClientProperties {

    @NotNull @NotEmpty
    private String baseUrl;

    @NotNull @NotEmpty
    private List<String> baseUrls;  // Para load balancing

    @NotNull @NotEmpty
    private String username;

    @NotNull @NotEmpty
    private String password;
}
```

```properties
# application.properties
webclient.properties.baseUrl=https://domino-server:8443/api/v1/
webclient.properties.baseUrls=https://server1:8443/api/v1/,https://server2:8443/api/v1/
webclient.properties.username=app-user
webclient.properties.password=${DOMINO_API_PASSWORD:changeme}
```

O password usa `${DOMINO_API_PASSWORD:changeme}` — le da variavel de ambiente, com fallback para desenvolvimento.

## WebClientService: wrapper com token

```java
@Service
public class WebClientService {

    private final WebClient webClient;
    private String token;
    private static final int BUFFER_SIZE =
        100 * 1024 * 1024;

    public WebClientService(WebClient webClient) {
        this.webClient = webClient.mutate()
            .codecs(configurer -> configurer
                .defaultCodecs()
                .maxInMemorySize(BUFFER_SIZE))
            .build();
    }

    public WebClient getWebClient() {
        return webClient;
    }

    public String getToken() { return token; }
    public void setToken(String token) {
        this.token = token;
    }
}
```

## Error handling: de HTTP para exceptions tipadas

### ErrorResponse: mapeando erros do DRAPI

O DRAPI retorna erros em formato JSON estruturado:

```json
{
  "status": 404,
  "message": "Document not found",
  "details": "UNID ABC123 does not exist in scope empresas",
  "errorId": "ERR-12345"
}
```

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public class ErrorResponse {

    @JsonProperty("status")
    private int status;

    @JsonProperty("message")
    private String message;

    @JsonProperty("details")
    private String details;

    @JsonProperty("errorId")
    private String errorId;
}
```

### CustomWebClientException: exception tipada

```java
public class CustomWebClientException
        extends RuntimeException {

    private final int statusCode;
    private final ErrorResponse errorResponse;

    public CustomWebClientException(String message,
            int statusCode, ErrorResponse errorResponse) {
        super(message);
        this.statusCode = statusCode;
        this.errorResponse = errorResponse;
    }
}
```

### A cadeia de error handling

```java
// No AbstractService:

// Metodo GET com error handling
private String doGet(String uri) {
    return webClient.get()
        .uri(uri)
        .header("Content-Type",
            "application/json; charset=UTF-8")
        .header("Authorization",
            "Bearer " + getUserToken())
        .retrieve()
        .onStatus(HttpStatusCode::isError,
            clientResponse -> clientResponse
                .bodyToMono(ErrorResponse.class)
                .flatMap(err -> Mono.error(
                    new CustomWebClientException(
                        err.getMessage(),
                        err.getStatus(), err))))
        .bodyToMono(String.class)
        .block();
}
```

### Tratamento no nivel do service

```java
private Response<T> getAndPopulaModelo(
        String uri, boolean carregarAnexos) {
    try {
        String raw = doGet(uri);
        T model = mapRawToModel(raw, carregarAnexos);
        return new Response<>(model,
            "Documento carregado.", 200, true);

    } catch (CustomWebClientException e) {
        ErrorResponse err = e.getErrorResponse();

        // Fallback: se erro e "MIME part not found",
        // tenta sem MIME
        if (err != null && err.getMessage()
                .contains("mime part not found")) {
            Response<T> fallback =
                tryFallbackWithoutMime(uri);
            if (fallback != null) return fallback;
        }

        return new Response<>(null,
            err.getMessage(), err.getStatus(), false);

    } catch (Exception e) {
        return new Response<>(null,
            "Erro: " + e.getMessage(), 500, false);
    }
}
```

### Fallback: retry sem MIME

Alguns documentos Domino tem problemas com encoding MIME. O fallback tenta novamente sem MIME:

```java
private Response<T> tryFallbackWithoutMime(
        String uri) {
    try {
        String fallbackUri = uri
            .replace("richTextAs=mime", "richTextAs=none")
            .replace("mode=" + mode, "mode=nomime");

        String raw = doGet(fallbackUri);
        T model = mapRawToModel(raw, false);
        return new Response<>(model,
            "Carregado (sem MIME).", 200, true);
    } catch (Exception e) {
        return null;
    }
}
```

## Token JWT: injecao em cada request

O token e obtido da sessao do usuario e adicionado como header Authorization:

```java
// No AbstractService:
protected String getUserToken() {
    User user = UtilsSession.getCurrentUser();
    if (user == null || user.getToken() == null) {
        throw new IllegalStateException(
            "Usuario nao autenticado");
    }
    return user.getToken();
}

// Uso em cada chamada:
webClient.get()
    .uri("/document/" + unid
        + "?dataSource=" + scope)
    .header("Authorization",
        "Bearer " + getUserToken())
    .retrieve()
    .bodyToMono(String.class)
    .block();
```

## Save: POST e PUT com error handling

```java
// Documento novo: POST
String rawResponse = webClient.post()
    .uri("/document?dataSource=" + scope
        + "&richTextAs=mime")
    .header("Authorization",
        "Bearer " + getUserToken())
    .header("Content-Type", "application/json")
    .bodyValue(json)
    .retrieve()
    .onStatus(HttpStatusCode::isError,
        this::handleError)
    .bodyToMono(String.class)
    .block();

// Documento existente: PUT com revision
String rawResponse = webClient.put()
    .uri("/document/" + unid
        + "?dataSource=" + scope
        + "&richTextAs=mime"
        + "&mode=" + mode
        + "&revision=" + model.getRevision())
    .header("Authorization",
        "Bearer " + getUserToken())
    .header("Content-Type", "application/json")
    .bodyValue(json)
    .retrieve()
    .onStatus(HttpStatusCode::isError,
        this::handleError)
    .bodyToMono(String.class)
    .block();
```

## O Response wrapper

Todas as operacoes retornam um `Response<T>` consistente:

```java
public class Response<T> {
    private T model;
    private String message;
    private int statusCode;
    private boolean success;

    public boolean isSuccess() { return success; }
}
```

Isso permite que views tratem erros de forma uniforme:

```java
Response<Empresa> response =
    service.findByUnid(unid);

if (response.isSuccess()) {
    model = response.getModel();
    binder.setBean(model);
} else {
    Notification.show(response.getMessage());
}
```

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Buffer de 256KB estoura com DQL | ExchangeStrategies com 100MB |
| Erros do DRAPI sao JSON, nao texto | ErrorResponse mapeado com Jackson |
| Exception generica dificulta tratamento | CustomWebClientException com statusCode e ErrorResponse |
| Documentos com MIME corrompido | Fallback automatico com richTextAs=none |
| Token precisa estar em cada request | getUserToken() da VaadinSession |
| Password hardcoded no properties | ${DOMINO_API_PASSWORD:changeme} com variavel de ambiente |
| Virgulas nos campos DQL codificadas | String.format() em vez de URI builder |

---

Com essa configuracao, o WebClient funciona de forma robusta com o DRAPI — buffers grandes para DQL, error handling tipado, fallback para MIME, e token JWT transparente. O service nunca precisa se preocupar com detalhes HTTP.
