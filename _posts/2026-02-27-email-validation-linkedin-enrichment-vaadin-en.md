---
layout: post
title: "Email validation with ZeroBounce and LinkedIn enrichment with Apollo.io in Vaadin"
date: 2026-02-27
lang: en
categories: [java, vaadin, zerobounce, apollo, linkedin, email]
---

Contact forms need valid emails — and what if we could automatically enrich the record with LinkedIn data? This post shows how we integrated ZeroBounce (SMTP email validation) and Apollo.io (LinkedIn profile lookup) into a Vaadin form, with automatic population of name, job title and LinkedIn URL.

## The complete flow

```
User types email and presses TAB
  |
  v  Vaadin EmailValidator (format)
  |
  v  ZeroBounce API (SMTP validation)
  |
  +-- INVALID / DO_NOT_MAIL --> Blocks the form
  |
  +-- VALID / CATCH-ALL --> Shows extra fields
       |
       v  Apollo.io API (LinkedIn lookup)
       |
       +-- Found --> Populates name, title, LinkedIn URL
       |
       +-- Not found --> User fills in manually
```

The form only reveals extra fields after the email is validated. If Apollo finds data, it automatically populates first name, last name and job title.

## ZeroBounce: SMTP email validation

### The service

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

### The response model

ZeroBounce returns rich data about the email:

```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public class Zerobounce {
    private String address;      // Validated email
    private String status;       // valid, invalid, catch-all, do_not_mail
    private String sub_status;   // Status detail
    private boolean free_email;  // Gmail, Yahoo, etc.
    private String did_you_mean; // Correction suggestion
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

### The status enum

```java
public enum EmailValidationStatus {
    VALID,
    INVALID,
    UNCERTAIN  // catch-all or error
}
```

ZeroBounce validates via SMTP whether the email actually exists — not just whether the format is correct. "Catch-all" domains accept any address, so we cannot confirm whether the specific email is valid.

## Apollo.io: LinkedIn data lookup

### The service

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
            log.warn("Apollo API key not configured!");
        }
    }

    public LinkedInResponse buscarContato(
            String email, String firstName,
            String lastName) {

        if (email == null
                || !EMAIL_PATTERN.matcher(email).matches()) {
            throw new IllegalArgumentException(
                "Invalid email");
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

### The request/response models

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
        private String title;          // Job title
        private String linkedin_url;   // LinkedIn profile
        private Organization organization;
    }

    @Data
    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class Organization {
        private String id;
        private String name;           // Company name
        private String linkedin_url;   // Company LinkedIn
    }
}
```

Apollo receives email + name and returns the full LinkedIn profile: job title, LinkedIn URL for both the contact and the company.

## Integration in the Vaadin form

### Binder validation chain

```java
binder.forField(codigoField)
    .asRequired("Email is required")
    .withValidator(
        new EmailValidator("Invalid email"))
    .withValidator(
        (value, context) -> validateZerobounce(value))
    .withValidator(
        (value, context) -> validateEmailUnico(value))
    .bind(Contato::getCodigo, (contato, value) -> {
        contato.setCodigo(value);
        contato.setEmail(value);
    });
```

Three levels of chained validation:
1. **EmailValidator** — basic format (regex)
2. **ZeroBounce** — real SMTP validation
3. **Uniqueness** — checks if the email already exists in the database

### ZeroBounce validation with visual feedback

```java
private ValidationResult validateZerobounce(
        String value) {
    if (value == null || value.isBlank()) {
        return ValidationResult.ok();
    }

    // If the email changed, force new validation
    if (!value.equalsIgnoreCase(ultimoEmailValidado)) {
        emailValidado = false;
        apolloConsultado = false;
    }
    if (emailValidado) {
        return ValidationResult.ok();
    }

    // Visual feedback
    emailStatus.setVisible(true);
    emailStatus.setText("Validating email...");
    emailStatus.getStyle()
        .set("color", "var(--lumo-primary-color)");

    Zerobounce result =
        zerobounceService.validarEmail(value);

    if (result == null
            || result.getStatus() == null) {
        emailStatus.setText("Validation error");
        emailStatus.getStyle()
            .set("color", "var(--lumo-error-color)");
        return ValidationResult.error(
            "Error validating email.");
    }

    String status = result.getStatus()
        .trim().toLowerCase();

    return switch (status) {
        case "valid" -> {
            emailStatus.setText("Valid email");
            emailStatus.getStyle()
                .set("color",
                    "var(--lumo-success-color)");
            emailValidado = true;
            ultimoEmailValidado = value;
            setExtraFieldsVisible(true);

            // Trigger LinkedIn lookup automatically
            if (isNovo && !apolloConsultado) {
                apolloConsultado = true;
                consultarApollo();
            }
            yield ValidationResult.ok();
        }

        case "catch-all" -> {
            emailStatus.setText(
                "Catch-all domain (accepted)");
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
                "Invalid email: "
                + result.getStatus());
            emailStatus.getStyle()
                .set("color",
                    "var(--lumo-error-color)");
            emailValidado = false;
            setExtraFieldsVisible(false);
            yield ValidationResult.error(
                "Invalid email (ZeroBounce: "
                + result.getStatus() + ").");
        }
    };
}
```

The Java 21 `switch` with `yield` makes the code expressive. Each ZeroBounce status has different visual treatment in the form.

### Automatic Apollo lookup

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
                "LinkedIn: no profile found",
                3000, Notification.Position.BOTTOM_START);
            return;
        }

        ApolloLinkedin.Match match =
            response.getMatches().get(0);

        // Display data in visual component
        apolloLinkedinView.popular(match);
        apolloLinkedinDetails.setVisible(true);
        apolloLinkedinDetails.setOpened(true);

        // Persist in the model
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

        // Auto-populate form fields
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

        Notification.show("LinkedIn: profile found!",
            3000, Notification.Position.BOTTOM_START);

    } catch (Exception e) {
        Notification.show(
            "Error querying LinkedIn: "
            + e.getMessage(),
            5000, Notification.Position.BOTTOM_START);
    }
}
```

The `getUI().ifPresent(ui -> ui.access(...))` is necessary because we are updating the UI outside the Binder's normal validation cycle. Without this, Vaadin would not propagate the changes to the browser.

## ApolloLinkedinView: displaying LinkedIn data

```java
public class ApolloLinkedinView extends VerticalLayout {

    private TextField primeiroNomeField =
        criarField("First Name (LinkedIn)");
    private TextField sobrenomeField =
        criarField("Last Name (LinkedIn)");
    private TextField cargoField = criarField("Job Title");
    private Anchor linkedinContatoLink =
        new Anchor("", "Contact LinkedIn");
    private TextField idContatoField =
        criarField("Apollo Contact ID");
    private TextField empresaNomeField =
        criarField("Company (LinkedIn)");
    private Anchor linkedinEmpresaLink =
        new Anchor("", "Company LinkedIn");
    private TextField idEmpresaField =
        criarField("Apollo Company ID");

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

    // Populate from API response
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
            "Contact LinkedIn");
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
                "Company LinkedIn");
            idEmpresaField.setValue(
                Optional.ofNullable(
                    match.getOrganization().getId())
                    .orElse(""));
        }
    }

    // Populate from already saved data
    public void popularFromContato(Contato contato) {
        if (contato == null) return;

        cargoField.setValue(
            Optional.ofNullable(
                contato.getLinkedinContatoCargo())
                .orElse(""));
        setLinkValue(linkedinContatoLink,
            contato.getLinkedinContatoUrl(),
            "Contact LinkedIn");
        empresaNomeField.setValue(
            Optional.ofNullable(
                contato.getLinkedinEmpresaNome())
                .orElse(""));
        setLinkValue(linkedinEmpresaLink,
            contato.getLinkedinEmpresaUrl(),
            "Company LinkedIn");
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

The component has two modes: `popular()` to populate from the API (new contact) and `popularFromContato()` to display already saved data (existing contact). This avoids unnecessary Apollo calls when the contact has already been enriched.

## Storing data in the model

```java
// Contato model fields for LinkedIn
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

// Email validation tracking
@JsonProperty("CheckEmail")
private List<String> checkEmail;

@JsonProperty("DataCheckEmail")
private ZonedDateTime dataCheckEmail;
```

Apollo data is persisted alongside the contact. Next time the contact is opened, `popularFromContato()` displays the data without querying the API again.

## Configuration and security

```properties
# application.properties
zerobounce.api.key=${ZEROBOUNCE_API_KEY:}
apollo.api.key=${APOLLO_API_KEY:}
```

```bash
# .env (do not commit!)
ZEROBOUNCE_API_KEY=your-zerobounce-key
APOLLO_API_KEY=your-apollo-key
```

```java
# .env.example (commit as reference)
# ZeroBounce API Key (email validation)
# Get yours at: https://www.zerobounce.net/
ZEROBOUNCE_API_KEY=

# Apollo.io API Key (LinkedIn enrichment)
# Get yours at: https://app.apollo.io/#/settings/integrations/api_keys
APOLLO_API_KEY=
```

**Never** put API keys directly in the code or in `application.properties`. Use environment variables via `.env` (with dotenv-java) and keep `.env` in `.gitignore`.

## Standalone test view

To test validation independently from the contact form:

```java
@Route("zerobounce")
@PageTitle("Email Validator")
public class ZerobounceView extends VerticalLayout {

    private final ZerobounceService service;
    private final TextField emailField =
        new TextField("Email to validate");
    private final Button validarButton =
        new Button("Validate");
    private final TextArea resultadoArea =
        new TextArea("Result");

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
                "Please enter a valid email.");
            return;
        }

        Zerobounce result =
            service.validarEmail(email);
        if (result == null) {
            resultadoArea.setValue(
                "Error calling the API.");
            return;
        }

        StringBuilder sb = new StringBuilder();
        sb.append("Email: ")
            .append(result.getAddress()).append("\n");
        sb.append("Status: ")
            .append(result.getStatus()).append("\n");
        sb.append("Sub-status: ")
            .append(result.getSub_status()).append("\n");
        sb.append("Domain: ")
            .append(result.getDomain()).append("\n");
        sb.append("MX found: ")
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

This view is useful for validating standalone emails and verifying that the ZeroBounce integration is working.

## Lessons learned

| Challenge | Solution |
|---|---|
| Valid format does not guarantee a real email | ZeroBounce validates via SMTP, not just regex |
| Catch-all domains accept any email | UNCERTAIN status — accepts but warns the user |
| Contact registration without enriched data | Apollo.io auto-populates name, title and LinkedIn |
| API keys exposed in code | Environment variables via .env + dotenv-java |
| LinkedIn fields need to persist | 7 dedicated fields in the Contato model |
| Unnecessary Apollo queries for existing contacts | `popularFromContato()` loads already saved data |
| Updating UI outside the Binder cycle | `getUI().ifPresent(ui -> ui.access(...))` |
| Validation needs to be chained | Binder: EmailValidator -> ZeroBounce -> Uniqueness |

---

With ZeroBounce + Apollo.io integrated into the Vaadin Binder, the contact form validates the email in real time, automatically enriches it with LinkedIn data and persists everything to the document. The user only needs to type the email — the rest is automatic.
