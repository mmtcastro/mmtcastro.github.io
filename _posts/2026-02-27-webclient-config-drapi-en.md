---
layout: post
title: "Configuring Spring WebClient for the Domino REST API: buffer, error handling and JWT token"
date: 2026-02-27
lang: en
categories: [domino, drapi, java, spring, webclient]
---

Spring WebClient is the reactive HTTP client we use for all communication with DRAPI. But the default configuration does not work well with Domino — large DQL responses overflow the 1MB buffer, errors need to be mapped to typed exceptions, and the JWT token needs to be injected into each request. This post shows how we configured WebClient for DRAPI.

## The default buffer problem

Spring's WebClient has a default buffer of **256KB** (or 1MB depending on the version). When a DQL query returns 50MB of JSON, you get:

```
org.springframework.core.io.buffer.DataBufferLimitException:
Exceeded limit on max bytes to buffer: 262144
```

## WebClient configuration: 100MB buffer

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

**Why 100MB?** DQL queries without field projection can return enormous payloads (81MB in production for 5000 complete documents). With field projection, it rarely exceeds 1MB — but the large buffer is a safety net.

## Configuration properties

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

The password uses `${DOMINO_API_PASSWORD:changeme}` — it reads from the environment variable, with a fallback for development.

## WebClientService: wrapper with token

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

## Error handling: from HTTP to typed exceptions

### ErrorResponse: mapping DRAPI errors

DRAPI returns errors in a structured JSON format:

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

### CustomWebClientException: typed exception

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

### The error handling chain

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

### Handling at the service level

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

### Fallback: retry without MIME

Some Domino documents have MIME encoding issues. The fallback retries without MIME:

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

## JWT token: injection in each request

The token is obtained from the user session and added as an Authorization header:

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

## Save: POST and PUT with error handling

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

## The Response wrapper

All operations return a consistent `Response<T>`:

```java
public class Response<T> {
    private T model;
    private String message;
    private int statusCode;
    private boolean success;

    public boolean isSuccess() { return success; }
}
```

This allows views to handle errors uniformly:

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

## Lessons learned

| Challenge | Solution |
|---|---|
| 256KB buffer overflows with DQL | ExchangeStrategies with 100MB |
| DRAPI errors are JSON, not text | ErrorResponse mapped with Jackson |
| Generic exception makes handling difficult | CustomWebClientException with statusCode and ErrorResponse |
| Documents with corrupted MIME | Automatic fallback with richTextAs=none |
| Token needs to be in every request | getUserToken() from VaadinSession |
| Hardcoded password in properties | ${DOMINO_API_PASSWORD:changeme} with environment variable |
| Commas in DQL fields get encoded | String.format() instead of URI builder |

---

With this configuration, WebClient works robustly with DRAPI — large buffers for DQL, typed error handling, MIME fallback, and transparent JWT token. The service never needs to worry about HTTP details.
