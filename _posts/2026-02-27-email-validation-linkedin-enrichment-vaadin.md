---
layout: post
title: "Validacao de e-mail com ZeroBounce e enriquecimento LinkedIn com Apollo.io no Vaadin"
date: 2026-02-27
categories: [java, vaadin, zerobounce, apollo, linkedin, email]
---

Formularios de contato precisam de e-mails validos — e se pudermos enriquecer o cadastro automaticamente com dados do LinkedIn? Este post mostra como integramos o ZeroBounce (validacao de e-mail via SMTP) e o Apollo.io (busca de perfil LinkedIn) em um formulario Vaadin, com preenchimento automatico de nome, cargo e URL do LinkedIn.

## O fluxo completo

```
Usuario digita e-mail e pressiona TAB
  |
  v  Vaadin EmailValidator (formato)
  |
  v  ZeroBounce API (validacao SMTP)
  |
  +-- INVALID / DO_NOT_MAIL --> Bloqueia formulario
  |
  +-- VALID / CATCH-ALL --> Libera campos extras
       |
       v  Apollo.io API (busca LinkedIn)
       |
       +-- Encontrou --> Preenche nome, cargo, LinkedIn URL
       |
       +-- Nao encontrou --> Usuario preenche manualmente
```

O formulario so libera os campos apos a validacao do e-mail. Se o Apollo encontrar dados, preenche automaticamente nome, sobrenome e cargo.

## ZeroBounce: validacao de e-mail via SMTP

### O servico

```java
@Service
public class ZerobounceService {

    private final WebClient webClient;

    @Value("${zerobounce.api.key:}")
    private String apiKey;

    public ZerobounceService() {
        this.webClient = WebClient.builder()
            .baseUrl("https://api.zerobounce.net/v2")
            .defaultHeader(HttpHeaders.CONTENT_TYPE,
                MediaType.APPLICATION_JSON_VALUE)
            .build();
    }

    public Zerobounce validarEmail(String email) {
        return webClient.get()
            .uri(uriBuilder -> uriBuilder
                .path("/validate")
                .queryParam("api_key", apiKey)
                .queryParam("email", email)
                .build())
            .retrieve()
            .bodyToMono(Zerobounce.class)
            .onErrorResume(e -> Mono.empty())
            .block();
    }

    public EmailValidationStatus validarStatusEmail(
            String email) {
        Zerobounce result = validarEmail(email);

        if (result == null
                || result.getStatus() == null) {
            return EmailValidationStatus.UNCERTAIN;
        }

        String status = result.getStatus().toLowerCase();

        return switch (status) {
            case "valid" ->
                EmailValidationStatus.VALID;
            case "invalid", "do_not_mail" ->
                EmailValidationStatus.INVALID;
            case "catch-all" ->
                EmailValidationStatus.UNCERTAIN;
            default ->
                EmailValidationStatus.UNCERTAIN;
        };
    }
}
```

### O modelo de resposta

O ZeroBounce retorna dados ricos sobre o e-mail:

```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public class Zerobounce {
    private String address;      // E-mail validado
    private String status;       // valid, invalid, catch-all, do_not_mail
    private String sub_status;   // Detalhe do status
    private boolean free_email;  // Gmail, Yahoo, etc.
    private String did_you_mean; // Sugestao de correcao
    private String domain;
    private String domain_age_days;
    private String smtp_provider;
    private String mx_found;
    private String mx_record;
    private String firstname;
    private String lastname;
    private String gender;
    private String country;
    private String region;
    private String city;
    private String zipcode;
    private String processed_at;
}
```

### O enum de status

```java
public enum EmailValidationStatus {
    VALID,
    INVALID,
    UNCERTAIN  // catch-all ou erro
}
```

O ZeroBounce verifica via SMTP se o e-mail existe de verdade — nao apenas se o formato esta correto. Dominios "catch-all" aceitam qualquer endereco, entao nao podemos confirmar se o e-mail especifico e valido.

## Apollo.io: busca de dados LinkedIn

### O servico

```java
@Service
public class ApolloLinkedinService {

    private static final String BASE_URL =
        "https://api.apollo.io";
    private static final String MATCH_ENDPOINT =
        "/api/v1/people/match";
    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$");

    private final WebClient webClient;
    private final String apiKey;

    public ApolloLinkedinService(
            @Value("${apollo.api.key:}") String apiKey) {
        this.apiKey = apiKey;
        this.webClient = WebClient.builder()
            .baseUrl(BASE_URL)
            .defaultHeader(HttpHeaders.CONTENT_TYPE,
                MediaType.APPLICATION_JSON_VALUE)
            .build();

        if (apiKey == null || apiKey.isBlank()) {
            log.warn("Apollo API key nao configurada!");
        }
    }

    public LinkedInResponse buscarContato(
            String email, String firstName,
            String lastName) {

        if (email == null
                || !EMAIL_PATTERN.matcher(email).matches()) {
            throw new IllegalArgumentException(
                "Email invalido");
        }

        Body body = Body.of(
            apiKey, email, firstName, lastName);

        return webClient.post()
            .uri(MATCH_ENDPOINT)
            .bodyValue(body)
            .retrieve()
            .onStatus(HttpStatusCode::isError,
                clientResponse -> {
                    int status = clientResponse
                        .statusCode().value();
                    return clientResponse
                        .bodyToMono(String.class)
                        .flatMap(msg -> Mono.error(
                            new RuntimeException(
                                "Apollo HTTP " + status
                                + ": " + msg)));
                })
            .bodyToMono(LinkedInResponse.class)
            .block();
    }
}
```

### Os modelos request/response

```java
public class ApolloLinkedin {

    // Request body
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class Body {
        private String api_key;
        private Boolean reveal_personal_emails = true;
        private String email;
        private String first_name;
        private String last_name;

        public static Body of(String apiKey,
                String email, String firstName,
                String lastName) {
            return new Body(apiKey, true,
                email, firstName, lastName);
        }
    }

    // Response
    @Data
    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class LinkedInResponse {
        private Person person;

        public List<Match> getMatches() {
            if (person == null) {
                return Collections.emptyList();
            }
            return Collections.singletonList(
                new Match(person));
        }
    }

    @Data
    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class Person {
        private String id;
        private String first_name;
        private String last_name;
        private String title;          // Cargo
        private String linkedin_url;   // Perfil LinkedIn
        private Organization organization;
    }

    @Data
    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class Organization {
        private String id;
        private String name;           // Nome da empresa
        private String linkedin_url;   // LinkedIn da empresa
    }
}
```

O Apollo recebe e-mail + nome e retorna o perfil LinkedIn completo: cargo, URL do LinkedIn do contato e da empresa.

## Integracao no formulario Vaadin

### Cadeia de validacao no Binder

```java
binder.forField(codigoField)
    .asRequired("Informe o e-mail")
    .withValidator(
        new EmailValidator("E-mail invalido"))
    .withValidator(
        (value, context) -> validateZerobounce(value))
    .withValidator(
        (value, context) -> validateEmailUnico(value))
    .bind(Contato::getCodigo, (contato, value) -> {
        contato.setCodigo(value);
        contato.setEmail(value);
    });
```

Tres niveis de validacao em cadeia:
1. **EmailValidator** — formato basico (regex)
2. **ZeroBounce** — validacao SMTP real
3. **Unicidade** — verifica se o e-mail ja existe no banco

### Validacao ZeroBounce com feedback visual

```java
private ValidationResult validateZerobounce(
        String value) {
    if (value == null || value.isBlank()) {
        return ValidationResult.ok();
    }

    // Se o email mudou, forcar nova validacao
    if (!value.equalsIgnoreCase(ultimoEmailValidado)) {
        emailValidado = false;
        apolloConsultado = false;
    }
    if (emailValidado) {
        return ValidationResult.ok();
    }

    // Feedback visual
    emailStatus.setVisible(true);
    emailStatus.setText("Validando e-mail...");
    emailStatus.getStyle()
        .set("color", "var(--lumo-primary-color)");

    Zerobounce result =
        zerobounceService.validarEmail(value);

    if (result == null
            || result.getStatus() == null) {
        emailStatus.setText("Erro na validacao");
        emailStatus.getStyle()
            .set("color", "var(--lumo-error-color)");
        return ValidationResult.error(
            "Erro ao validar e-mail.");
    }

    String status = result.getStatus()
        .trim().toLowerCase();

    return switch (status) {
        case "valid" -> {
            emailStatus.setText("E-mail valido");
            emailStatus.getStyle()
                .set("color",
                    "var(--lumo-success-color)");
            emailValidado = true;
            ultimoEmailValidado = value;
            setExtraFieldsVisible(true);

            // Dispara busca LinkedIn automaticamente
            if (isNovo && !apolloConsultado) {
                apolloConsultado = true;
                consultarApollo();
            }
            yield ValidationResult.ok();
        }

        case "catch-all" -> {
            emailStatus.setText(
                "Dominio catch-all (aceito)");
            emailStatus.getStyle()
                .set("color",
                    "var(--lumo-warning-color)");
            emailValidado = true;
            ultimoEmailValidado = value;
            setExtraFieldsVisible(true);

            if (isNovo && !apolloConsultado) {
                apolloConsultado = true;
                consultarApollo();
            }
            yield ValidationResult.ok();
        }

        default -> {
            emailStatus.setText(
                "E-mail invalido: "
                + result.getStatus());
            emailStatus.getStyle()
                .set("color",
                    "var(--lumo-error-color)");
            emailValidado = false;
            setExtraFieldsVisible(false);
            yield ValidationResult.error(
                "E-mail invalido (ZeroBounce: "
                + result.getStatus() + ").");
        }
    };
}
```

O `switch` com `yield` do Java 21 torna o codigo bem expressivo. Cada status do ZeroBounce tem um tratamento visual diferente no formulario.

### Consulta Apollo automatica

```java
private void consultarApollo() {
    String email = codigoField.getValue();
    if (email == null || email.isBlank()) return;

    try {
        String firstName = primeiroNome.getValue();
        String lastName = sobrenome.getValue();

        ApolloLinkedin.LinkedInResponse response =
            apolloLinkedinService.buscarContato(
                email, firstName, lastName);

        if (response == null
                || response.getMatches().isEmpty()) {
            Notification.show(
                "LinkedIn: nenhum perfil encontrado",
                3000, Notification.Position.BOTTOM_START);
            return;
        }

        ApolloLinkedin.Match match =
            response.getMatches().get(0);

        // Exibir dados no componente visual
        apolloLinkedinView.popular(match);
        apolloLinkedinDetails.setVisible(true);
        apolloLinkedinDetails.setOpened(true);

        // Persistir no modelo
        model.setLinkedinContatoUrl(
            match.getLinkedin_url());
        model.setLinkedinContatoCargo(match.getTitle());
        model.setLinkedinContatoId(match.getId());

        if (match.getOrganization() != null) {
            model.setLinkedinEmpresaUrl(
                match.getOrganization().getLinkedin_url());
            model.setLinkedinEmpresaNome(
                match.getOrganization().getName());
            model.setLinkedinEmpresaId(
                match.getOrganization().getId());
        }

        // Auto-preencher campos do formulario
        String apolloFirst = match.getFirst_name();
        String apolloLast = match.getLast_name();
        String apolloTitle = match.getTitle();

        getUI().ifPresent(ui -> ui.access(() -> {
            if (apolloFirst != null)
                primeiroNome.setValue(apolloFirst);
            if (apolloLast != null)
                sobrenome.setValue(apolloLast);
            if (apolloTitle != null)
                cargo.setValue(apolloTitle);
        }));

        Notification.show("LinkedIn: perfil encontrado!",
            3000, Notification.Position.BOTTOM_START);

    } catch (Exception e) {
        Notification.show(
            "Erro ao consultar LinkedIn: "
            + e.getMessage(),
            5000, Notification.Position.BOTTOM_START);
    }
}
```

O `getUI().ifPresent(ui -> ui.access(...))` e necessario porque estamos atualizando a UI fora do ciclo normal de validacao do Binder. Sem isso, o Vaadin nao propagaria as mudancas para o browser.

## ApolloLinkedinView: exibindo dados do LinkedIn

```java
public class ApolloLinkedinView extends VerticalLayout {

    private TextField primeiroNomeField =
        criarField("Primeiro Nome (LinkedIn)");
    private TextField sobrenomeField =
        criarField("Sobrenome (LinkedIn)");
    private TextField cargoField = criarField("Cargo");
    private Anchor linkedinContatoLink =
        new Anchor("", "LinkedIn do Contato");
    private TextField idContatoField =
        criarField("ID Apollo Contato");
    private TextField empresaNomeField =
        criarField("Empresa (LinkedIn)");
    private Anchor linkedinEmpresaLink =
        new Anchor("", "LinkedIn da Empresa");
    private TextField idEmpresaField =
        criarField("ID Apollo Empresa");

    public ApolloLinkedinView() {
        setSpacing(false);
        setPadding(false);
        setWidthFull();

        configurarLink(linkedinContatoLink);
        configurarLink(linkedinEmpresaLink);

        FormLayout form = new FormLayout();
        form.setWidthFull();
        form.setResponsiveSteps(
            new FormLayout.ResponsiveStep("0", 1),
            new FormLayout.ResponsiveStep("600px", 2));

        form.add(primeiroNomeField, sobrenomeField,
            cargoField, linkedinContatoLink,
            idContatoField, empresaNomeField,
            linkedinEmpresaLink, idEmpresaField);

        add(form);
    }

    // Preencher a partir da resposta da API
    public void popular(ApolloLinkedin.Match match) {
        if (match == null) return;

        primeiroNomeField.setValue(
            Optional.ofNullable(match.getFirst_name())
                .orElse(""));
        sobrenomeField.setValue(
            Optional.ofNullable(match.getLast_name())
                .orElse(""));
        cargoField.setValue(
            Optional.ofNullable(match.getTitle())
                .orElse(""));
        setLinkValue(linkedinContatoLink,
            match.getLinkedin_url(),
            "LinkedIn do Contato");
        idContatoField.setValue(
            Optional.ofNullable(match.getId())
                .orElse(""));

        if (match.getOrganization() != null) {
            empresaNomeField.setValue(
                Optional.ofNullable(
                    match.getOrganization().getName())
                    .orElse(""));
            setLinkValue(linkedinEmpresaLink,
                match.getOrganization().getLinkedin_url(),
                "LinkedIn da Empresa");
            idEmpresaField.setValue(
                Optional.ofNullable(
                    match.getOrganization().getId())
                    .orElse(""));
        }
    }

    // Preencher a partir de dados ja salvos
    public void popularFromContato(Contato contato) {
        if (contato == null) return;

        cargoField.setValue(
            Optional.ofNullable(
                contato.getLinkedinContatoCargo())
                .orElse(""));
        setLinkValue(linkedinContatoLink,
            contato.getLinkedinContatoUrl(),
            "LinkedIn do Contato");
        empresaNomeField.setValue(
            Optional.ofNullable(
                contato.getLinkedinEmpresaNome())
                .orElse(""));
        setLinkValue(linkedinEmpresaLink,
            contato.getLinkedinEmpresaUrl(),
            "LinkedIn da Empresa");
    }

    private static void configurarLink(Anchor link) {
        link.setTarget("_blank");
        link.getStyle()
            .set("font-size",
                "var(--lumo-font-size-m)")
            .set("padding", "var(--lumo-space-xs) 0");
    }

    private static TextField criarField(String label) {
        TextField field = new TextField(label);
        field.setReadOnly(true);
        field.setWidthFull();
        return field;
    }
}
```

O componente tem dois modos: `popular()` para preencher a partir da API (novo contato) e `popularFromContato()` para exibir dados ja salvos (contato existente). Isso evita consultas desnecessarias ao Apollo quando o contato ja foi enriquecido.

## Armazenando os dados no modelo

```java
// Campos do modelo Contato para LinkedIn
@JsonProperty("Linkedin")
private String linkedin;

@JsonProperty("LinkedinContatoUrl")
private String linkedinContatoUrl;

@JsonProperty("LinkedinContatoCargo")
private String linkedinContatoCargo;

@JsonProperty("LinkedinContatoId")
private String linkedinContatoId;

@JsonProperty("LinkedinEmpresaUrl")
private String linkedinEmpresaUrl;

@JsonProperty("LinkedinEmpresaNome")
private String linkedinEmpresaNome;

@JsonProperty("LinkedinEmpresaId")
private String linkedinEmpresaId;

// Campos para controle de validacao
@JsonProperty("CheckEmail")
private List<String> checkEmail;

@JsonProperty("DataCheckEmail")
private ZonedDateTime dataCheckEmail;
```

Os dados do Apollo sao persistidos junto com o contato. Na proxima vez que o contato for aberto, `popularFromContato()` exibe os dados sem precisar consultar a API novamente.

## Configuracao e seguranca

```properties
# application.properties
zerobounce.api.key=${ZEROBOUNCE_API_KEY:}
apollo.api.key=${APOLLO_API_KEY:}
```

```bash
# .env (nao comitar!)
ZEROBOUNCE_API_KEY=sua-chave-zerobounce
APOLLO_API_KEY=sua-chave-apollo
```

```java
# .env.example (comitar como referencia)
# API Key do ZeroBounce (validacao de e-mail)
# Obtenha em: https://www.zerobounce.net/
ZEROBOUNCE_API_KEY=

# API Key do Apollo.io (enriquecimento LinkedIn)
# Obtenha em: https://app.apollo.io/#/settings/integrations/api_keys
APOLLO_API_KEY=
```

**Nunca** coloque chaves de API diretamente no codigo ou no `application.properties`. Use variaveis de ambiente via `.env` (com dotenv-java) e mantenha o `.env` no `.gitignore`.

## View de teste standalone

Para testar a validacao independente do formulario de contato:

```java
@Route("zerobounce")
@PageTitle("Validador de E-mails")
public class ZerobounceView extends VerticalLayout {

    private final ZerobounceService service;
    private final TextField emailField =
        new TextField("E-mail para validar");
    private final Button validarButton =
        new Button("Validar");
    private final TextArea resultadoArea =
        new TextArea("Resultado");

    public ZerobounceView(ZerobounceService service) {
        this.service = service;
        resultadoArea.setWidthFull();
        resultadoArea.setHeight("200px");
        resultadoArea.setReadOnly(true);

        validarButton.addClickListener(
            e -> validarEmail());
        add(emailField, validarButton, resultadoArea);
    }

    private void validarEmail() {
        String email = emailField.getValue();
        if (email == null || email.trim().isEmpty()) {
            resultadoArea.setValue(
                "Insira um e-mail valido.");
            return;
        }

        Zerobounce result =
            service.validarEmail(email);
        if (result == null) {
            resultadoArea.setValue(
                "Erro ao chamar a API.");
            return;
        }

        StringBuilder sb = new StringBuilder();
        sb.append("E-mail: ")
            .append(result.getAddress()).append("\n");
        sb.append("Status: ")
            .append(result.getStatus()).append("\n");
        sb.append("Sub-status: ")
            .append(result.getSub_status()).append("\n");
        sb.append("Dominio: ")
            .append(result.getDomain()).append("\n");
        sb.append("MX encontrado: ")
            .append(result.getMx_found()).append("\n");
        sb.append("MX Record: ")
            .append(result.getMx_record()).append("\n");
        sb.append("SMTP Provider: ")
            .append(result.getSmtp_provider())
            .append("\n");
        resultadoArea.setValue(sb.toString());
    }
}
```

Essa view e util para validar e-mails avulsos e verificar que a integracao com o ZeroBounce esta funcionando.

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Formato valido nao garante e-mail real | ZeroBounce valida via SMTP, nao so regex |
| Dominios catch-all aceitam qualquer e-mail | Status UNCERTAIN — aceita mas avisa o usuario |
| Cadastro de contato sem dados enriquecidos | Apollo.io auto-preenche nome, cargo e LinkedIn |
| Chaves de API expostas no codigo | Variaveis de ambiente via .env + dotenv-java |
| Campos LinkedIn precisam persistir | 7 campos dedicados no modelo Contato |
| Consulta Apollo desnecessaria em contato existente | `popularFromContato()` carrega dados ja salvos |
| Atualizar UI fora do ciclo do Binder | `getUI().ifPresent(ui -> ui.access(...))` |
| Validacao precisa ser em cadeia | Binder: EmailValidator -> ZeroBounce -> Unicidade |

---

Com ZeroBounce + Apollo.io integrados ao Vaadin Binder, o formulario de contato valida o e-mail em tempo real, enriquece automaticamente com dados do LinkedIn e persiste tudo no documento. O usuario so precisa digitar o e-mail — o resto e automatico.
