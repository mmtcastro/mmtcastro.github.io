---
layout: post
title: "Address lookup with ViaCEP and CNPJ validation with ReceitaWS in Vaadin"
date: 2026-02-27
lang: en
categories: [java, vaadin, viacep, receitaws, cnpj, cep]
---

Registering a company in Brazil means filling in a complete address and validating the CNPJ (Brazilian company tax ID) with the federal tax authority. Both processes can be automated with public APIs: ViaCEP resolves the address from the postal code (CEP), and ReceitaWS returns the company name, size, partners, registration status and much more from the CNPJ. This post shows how we integrated both APIs into Vaadin forms.

## The complete flow

```
Company Registration:

1. User types the CNPJ and presses ENTER
   |
   v  Validates format (14 digits)
   |
   v  Checks if CNPJ already exists in the system
   |
   v  ReceitaWS API (GET /v1/cnpj/{cnpj})
   |
   +-- BAIXADA (closed) --> Blocks registration
   |
   +-- OK --> Auto-populates:
        - Company name, trade name
        - Full address (street, district, city, state, zip)
        - Phone, email, company size, opening date
        - Shows details panel (partners, CNAE, tax regime)

2. User types the CEP (postal code)
   |
   v  ViaCEP API (GET /ws/{cep}/json/)
   |
   +-- Found --> Populates street, district, city, state
   |
   +-- Not found --> User fills in manually
```

## ReceitaWS: CNPJ lookup with the Brazilian tax authority

### The service

ReceitaWS uses Java's native `HttpClient` (not Spring WebClient) since it is a simple public API that does not require a token:

```java
@Service
public class ReceitaWsService {

    private static final String API_URL =
        "https://receitaws.com.br/v1/cnpj/";

    private final HttpClient httpClient =
        HttpClient.newBuilder()
            .version(HttpClient.Version.HTTP_1_1)
            .connectTimeout(Duration.ofSeconds(15))
            .followRedirects(
                HttpClient.Redirect.NORMAL)
            .build();

    private final ObjectMapper objectMapper =
        new ObjectMapper()
            .registerModule(new JavaTimeModule());

    public ReceitaWs findCnpj(String cnpj) {
        String url = API_URL + cnpj;

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .GET()
            .header("Accept", "application/json")
            .header("User-Agent", "MyApp/1.0")
            .timeout(Duration.ofSeconds(15))
            .build();

        HttpResponse<String> response =
            httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        if (response.statusCode()
                != HttpURLConnection.HTTP_OK) {
            throw new RuntimeException(
                "Could not query the CNPJ. "
                + "Try again later.");
        }

        JsonNode root = objectMapper
            .readTree(response.body());

        // Functional error from the API
        if (root.has("status")
                && root.get("status").isTextual()
                && !"OK".equalsIgnoreCase(
                    root.get("status").asText())) {
            throw new RuntimeException(
                "CNPJ not found or "
                + "unavailable for query.");
        }

        return objectMapper.treeToValue(
            root, ReceitaWs.class);
    }
}
```

**Why HttpClient instead of WebClient?** ReceitaWS is a public API with no authentication. Java 11+'s native `HttpClient` is simpler for this case — no Spring WebFlux dependency needed.

### The response model

ReceitaWS returns very rich data. The model maps everything with inner classes:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
@Getter @Setter
public class ReceitaWs {

    private String status;

    @JsonProperty("ultima_atualizacao")
    @JsonFormat(
        pattern = "yyyy-MM-dd'T'HH:mm:ss.SSSX")
    private ZonedDateTime ultimaAtualizacao;

    private String cnpj;
    private String tipo;       // MATRIZ or FILIAL
    private String porte;      // ME, EPP, DEMAIS
    private String nome;       // Legal name
    private String fantasia;   // Trade name
    private String abertura;   // Opening date

    @JsonProperty("atividadePrincipal")
    private List<Atividade> atividadePrincipal;

    @JsonProperty("atividadesSecundarias")
    private List<Atividade> atividadesSecundarias;

    @JsonProperty("natureza_juridica")
    private String naturezaJuridica;

    // Address
    private String logradouro;
    private String numero;
    private String complemento;
    private String cep;
    private String bairro;
    private String municipio;
    private String uf;

    // Contact
    private String email;
    private String telefone;

    // Registration status
    private String situacao;      // ATIVA, BAIXADA, etc.
    @JsonProperty("data_situacao")
    private String dataSituacao;
    @JsonProperty("motivo_situacao")
    private String motivoSituacao;

    @JsonProperty("capital_social")
    private Double capitalSocial;

    private List<Socio> qsa;     // Partners
    private Simples simples;      // Tax regime

    // Inner classes
    public static class Atividade {
        private String code;      // CNAE code
        private String text;      // Description
    }

    public static class Socio {
        private String nome;
        private String qual;      // Qualification
        private String paisOrigem;
        private String nomeRepLegal;
        private String qualRepLegal;
    }

    @Getter @Setter
    public static class Simples {
        private Boolean optante;
        private String dataOpcao;
        private String dataExclusao;
    }
}
```

A single CNPJ returns the legal name, trade name, full address, CNAE (business activity code), partners, tax regime and registration status. This saves a lot of manual data entry.

## Integration in the Vaadin form

### Query button inside the CNPJ field

```java
// Inline button inside the CNPJ field
Button btnConsultaCnpj = new Button("Search CNPJ");
btnConsultaCnpj.setIcon(VaadinIcon.SEARCH.create());
btnConsultaCnpj.addThemeVariants(
    ButtonVariant.LUMO_TERTIARY_INLINE,
    ButtonVariant.LUMO_SMALL);
btnConsultaCnpj.getStyle()
    .set("color", "var(--lumo-primary-color)")
    .set("font-weight", "600");

// Two ways to trigger the query
btnConsultaCnpj.addClickListener(
    event -> consultarCnpj(true));
cgcField.addKeyPressListener(
    Key.ENTER, e -> consultarCnpj(true));

// Button as field suffix
cgcField.setSuffixComponent(btnConsultaCnpj);
```

`setSuffixComponent()` places the button inside the field, creating a clean UX where the user types and clicks without leaving the field.

### The consultarCnpj() method

```java
private void consultarCnpj(boolean forcar) {
    // 1. Validate format
    String cnpj = Optional.ofNullable(
            cgcField.getValue())
        .orElse("")
        .replaceAll("[^0-9]", "");

    if (cnpj.length() != 14
            || !Utils.isCnpjValido(cnpj)) {
        Notification.show(
            "Enter a valid CNPJ.",
            3000, Position.MIDDLE);
        return;
    }

    // Cache: don't re-query the same CNPJ
    if (!forcar
            && cnpj.equals(ultimoCnpjConsultado)) {
        return;
    }

    // 2. Check for duplicates BEFORE the API call
    //    (saves API credits)
    String cnpjFormatado = formatarCnpj(cnpj);
    Response<Empresa> existente =
        service.findByCnpj(cnpj);

    if (!existente.isSuccess()) {
        existente = service.findByCnpj(cnpjFormatado);
    }

    if (existente.isSuccess()
            && existente.getModel() != null) {
        Empresa empresaExistente =
            existente.getModel();
        if (!isMesmoDocumento(empresaExistente)) {
            String codigo =
                empresaExistente.getCodigo();
            Notification.show(
                "CNPJ already registered for "
                + codigo, 5000, Position.MIDDLE);
            cgcField.setInvalid(true);
            return;
        }
    }

    // 3. Query ReceitaWS
    try {
        ReceitaWs receita =
            receitaWsService.findCnpj(cnpj);

        // 4. Reject closed companies
        if ("BAIXADA".equalsIgnoreCase(
                receita.getSituacao())) {
            Notification.show(
                "CNPJ has BAIXADA (closed) status. "
                + "Cannot be registered.",
                5000, Position.MIDDLE);
            cgcField.setInvalid(true);
            return;
        }

        // 5. Auto-populate the form
        receitaWsView.popular(receita);
        receitaWsDetails.setVisible(true);

        model.setNome(receita.getNome());
        model.setNomeFantasia(receita.getFantasia());
        model.setCgc(receita.getCnpj());
        model.setTelefones(receita.getTelefone());
        model.setEmailNfeServicos(receita.getEmail());
        model.setEndereco(receita.getLogradouro());
        model.setNumero(receita.getNumero());
        model.setComplemento(receita.getComplemento());
        model.setBairro(receita.getBairro());
        model.setCidade(receita.getMunicipio());
        model.setEstado(receita.getUf());
        model.setCep(receita.getCep());
        model.setStatusCnpj(receita.getSituacao());
        model.setPorte(receita.getPorte());
        model.setPais("Brasil");

        // Opening date: convert dd/MM/yyyy
        if (receita.getAbertura() != null) {
            model.setDataAbertura(ZonedDateTime.of(
                LocalDate.parse(
                    receita.getAbertura(),
                    DateTimeFormatter.ofPattern(
                        "dd/MM/yyyy")),
                LocalTime.MIDNIGHT,
                ZoneId.systemDefault()));
        }

        binder.readBean(model);
        updateExtraFieldsVisibility(true);
        ultimoCnpjConsultado = cnpj;

        Notification.show(
            "CNPJ queried successfully.",
            3000, Position.MIDDLE);

    } catch (RuntimeException e) {
        Notification.show(
            e.getMessage(), 6000, Position.MIDDLE);
        clearCamposCnpj(true);
    }
}
```

**Key points:**
- Checks for duplicates **before** calling the API (saves ReceitaWS credits)
- Rejects companies with BAIXADA (closed) status
- Converts the opening date from `dd/MM/yyyy` to `ZonedDateTime`
- `binder.readBean(model)` updates all form fields at once

### ReceitaWsView: details panel

```java
public class ReceitaWsView extends VerticalLayout {

    private TextField statusField =
        criarField("Status");
    private TextField cnpjField =
        criarField("CNPJ");
    private TextField tipoField =
        criarField("Type");
    private TextField porteField =
        criarField("Size");
    private TextField nomeField =
        criarField("Legal Name");
    private TextField fantasiaField =
        criarField("Trade Name");
    private TextField naturezaField =
        criarField("Legal Nature");
    private TextField situacaoField =
        criarField("Registration Status");
    private TextField capitalField =
        criarField("Share Capital");
    private TextArea sociosField =
        criarTextArea("Partners");
    private TextArea atividadesPrincipaisField =
        criarTextArea("Main Activity");
    private TextField simplesField =
        criarField("Simples Nacional");
    // ... more fields

    public void popular(ReceitaWs dados) {
        nomeField.setValue(
            Optional.ofNullable(dados.getNome())
                .orElse(""));
        fantasiaField.setValue(
            Optional.ofNullable(dados.getFantasia())
                .orElse(""));
        situacaoField.setValue(
            Optional.ofNullable(dados.getSituacao())
                .orElse(""));

        // Formatted share capital
        capitalField.setValue(
            dados.getCapitalSocial() != null
                ? "R$ " + String.format(
                    Locale.getDefault(), "%,.2f",
                    dados.getCapitalSocial())
                : "");

        // Partners list
        sociosField.setValue(
            Optional.ofNullable(dados.getQsa())
                .filter(lista -> !lista.isEmpty())
                .map(lista -> lista.stream()
                    .map(s -> s.getNome()
                        + " (" + s.getQual() + ")")
                    .collect(
                        Collectors.joining("\n")))
                .orElse(""));

        // CNAE (business activity code)
        atividadesPrincipaisField.setValue(
            Optional.ofNullable(
                    dados.getAtividadePrincipal())
                .filter(lista -> !lista.isEmpty())
                .map(lista -> lista.stream()
                    .map(a -> a.getCode()
                        + " - " + a.getText())
                    .collect(
                        Collectors.joining("\n")))
                .orElse(""));

        // Simples Nacional tax regime
        if (dados.getSimples() != null) {
            simplesField.setValue(
                Boolean.TRUE.equals(
                    dados.getSimples().getOptante())
                    ? "Yes" : "No");
        }
    }
}
```

The panel is displayed inside a `Details` (accordion) — the user can expand to see all the tax authority data without cluttering the main form.

### Clearing fields on error

```java
private void clearCamposCnpj(boolean manterCnpj) {
    String cnpjAtual = manterCnpj
        ? cgcField.getValue() : null;

    ultimoCnpjConsultado = null;
    model.setNome(null);
    model.setNomeFantasia(null);
    model.setTelefones(null);
    model.setEmailNfeServicos(null);
    model.setEndereco(null);
    model.setNumero(null);
    model.setBairro(null);
    model.setCidade(null);
    model.setEstado(null);
    model.setCep(null);
    model.setStatusCnpj(null);
    model.setPorte(null);
    model.setDataAbertura(null);

    binder.readBean(model);

    if (cnpjAtual != null) {
        cgcField.setValue(cnpjAtual);
    }
}
```

The `manterCnpj` flag preserves the value typed in the CNPJ field while clearing all other fields. This avoids the user having to retype the CNPJ.

## ViaCEP: address lookup by postal code

### The service

```java
@Service
public class ViacepService {

    private static final String API_URL =
        "https://viacep.com.br/ws/";

    private final WebClient webClient;
    private final ObjectMapper objectMapper;

    public ViacepService(
            WebClientService webClientService,
            ObjectMapper objectMapper) {
        this.webClient =
            webClientService.getWebClient();
        this.objectMapper = objectMapper;
    }

    public Viacep buscarCep(String cep) {
        String url = API_URL + cep + "/json/";

        String rawResponse = webClient.get()
            .uri(url)
            .header("Accept", "application/json")
            .retrieve()
            .onStatus(HttpStatusCode::isError,
                clientResponse -> {
                    int status = clientResponse
                        .statusCode().value();
                    return Mono.error(
                        new RuntimeException(
                            "HTTP Error: " + status));
                })
            .bodyToMono(String.class)
            .block();

        JsonNode root = objectMapper
            .readTree(rawResponse);

        // The API returns {"erro": true} for invalid CEP
        if (root.has("erro")
                && root.get("erro").asBoolean()) {
            throw new RuntimeException(
                "CEP not found.");
        }

        return objectMapper.readValue(
            root.toString(), Viacep.class);
    }
}
```

### The response model

```java
@JsonIgnoreProperties(ignoreUnknown = true)
@Getter @Setter
public class Viacep {
    private String cep;
    private String logradouro;  // Street
    private String complemento;
    private String bairro;      // District

    @JsonProperty("localidade")
    private String cidade;       // API calls it "localidade"

    private String uf;           // State
    private String ibge;         // IBGE municipality code
    private String ddd;          // Phone area code
    private String siafi;        // SIAFI code
    private String estado;
    private String regiao;       // Region
}
```

Note the `@JsonProperty("localidade")`: the ViaCEP API returns the field as `localidade`, but in the model we map it to `cidade` (city) for consistency with the rest of the system.

### ViacepView: displaying CEP data

```java
public class ViacepView extends VerticalLayout {

    private final TextField cepField =
        criarField("CEP");
    private final TextField logradouroField =
        criarField("Street");
    private final TextField bairroField =
        criarField("District");
    private final TextField cidadeField =
        criarField("City");
    private final TextField ufField =
        criarField("State");
    private final TextField dddField =
        criarField("Area Code");
    private final TextField ibgeField =
        criarField("IBGE Code");

    public ViacepView() {
        setSpacing(false);
        setPadding(false);
        setWidthFull();

        FormLayout form = new FormLayout();
        form.setResponsiveSteps(
            new FormLayout.ResponsiveStep("0", 1),
            new FormLayout.ResponsiveStep("600px", 2));

        form.add(cepField, logradouroField,
            bairroField, cidadeField,
            ufField, dddField, ibgeField);
        add(form);
    }

    public void popular(Viacep viaCep) {
        cepField.setValue(
            Optional.ofNullable(viaCep.getCep())
                .orElse(""));
        logradouroField.setValue(
            Optional.ofNullable(
                viaCep.getLogradouro()).orElse(""));
        bairroField.setValue(
            Optional.ofNullable(viaCep.getBairro())
                .orElse(""));
        cidadeField.setValue(
            Optional.ofNullable(viaCep.getCidade())
                .orElse(""));
        ufField.setValue(
            Optional.ofNullable(viaCep.getUf())
                .orElse(""));
        dddField.setValue(
            Optional.ofNullable(viaCep.getDdd())
                .orElse(""));
        ibgeField.setValue(
            Optional.ofNullable(viaCep.getIbge())
                .orElse(""));
    }

    private static TextField criarField(String label) {
        TextField field = new TextField(label);
        field.setReadOnly(true);
        field.setWidthFull();
        return field;
    }
}
```

## Batch CNPJ verification (Grid)

Besides the individual form, the lookup also works in batch from the company list:

```java
// In EmpresasComponent (company Grid)
private void checkCnpj(Empresa empresa,
        Button button) {

    String cgc = empresa.getCgc();
    if (cgc == null || cgc.isBlank()) {
        Notification.show("CNPJ not provided",
            3000, Position.MIDDLE);
        return;
    }

    // Visual feedback on the button
    button.setEnabled(false);
    button.setText("Querying...");

    try {
        String cnpjLimpo =
            cgc.replaceAll("[^0-9]", "");

        ReceitaWs resultado =
            receitaWsService.findCnpj(cnpjLimpo);

        if (resultado != null
                && resultado.getSituacao() != null) {
            empresa.setStatusCnpj(
                resultado.getSituacao());
            empresa.setDataSituacao(
                resultado.getDataSituacao());
            // Save to database...
        }
    } catch (Exception e) {
        Notification.show(e.getMessage(),
            5000, Position.MIDDLE);
    } finally {
        button.setEnabled(true);
        button.setText("Query");
    }
}
```

This allows verifying the registration status of already registered companies — useful for discovering CNPJs that were closed after initial registration.

## Error handling pattern

Both APIs can fail for different reasons. The handling covers each case:

```java
// ReceitaWS - possible errors:
try {
    ReceitaWs receita =
        receitaWsService.findCnpj(cnpj);
    // ...
} catch (RuntimeException e) {
    // Messages come ready from the service:
    // - "CNPJ not found or unavailable"
    // - "Service unavailable" (SSL)
    // - "Error querying" (timeout/IO)
    Notification.show(e.getMessage(),
        6000, Position.MIDDLE);
}

// ViaCEP - possible errors:
try {
    Viacep endereco =
        viacepService.buscarCep(cep);
    // ...
} catch (RuntimeException e) {
    // - "CEP not found"
    // - "HTTP Error: 400"
    Notification.show(e.getMessage(),
        3000, Position.MIDDLE);
}
```

The service encapsulates error messages — the view never needs to know HTTP details.

## Lessons learned

| Challenge | Solution |
|---|---|
| Filling complete address manually | ViaCEP resolves everything from the postal code |
| CNPJ needs tax authority validation | ReceitaWS returns registration status and full data |
| Company with closed CNPJ should not be registered | BAIXADA status check before accepting |
| Duplicate queries waste API credits | Check CNPJ in database BEFORE calling ReceitaWS |
| Same CNPJ in two registrations | Uniqueness validation before external API call |
| API field "localidade" vs model "cidade" | `@JsonProperty("localidade")` in the DTO |
| Full data clutters the form | `Details` (accordion) to display ReceitaWS panel |
| CNPJ format varies (with/without dots) | `replaceAll("[^0-9]", "")` normalizes to 14 digits |
| Public API doesn't need reactive WebClient | Java 11+ native `HttpClient` is sufficient |
| Opening date in dd/MM/yyyy format | `DateTimeFormatter.ofPattern("dd/MM/yyyy")` to convert |

---

With ViaCEP + ReceitaWS integrated into the Vaadin form, company registration becomes almost automatic: the user types the CNPJ, presses ENTER, and the form fills itself with the legal name, address, size and registration status. Brazilian public APIs that save hours of manual data entry.
