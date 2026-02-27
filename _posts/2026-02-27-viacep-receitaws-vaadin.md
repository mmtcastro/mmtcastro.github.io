---
layout: post
title: "Consulta de CEP com ViaCEP e validacao de CNPJ com ReceitaWS no Vaadin"
date: 2026-02-27
categories: [java, vaadin, viacep, receitaws, cnpj, cep]
---

Cadastrar uma empresa no Brasil significa preencher endereco completo e validar o CNPJ na Receita Federal. Esses dois processos podem ser automatizados com APIs publicas: o ViaCEP resolve o endereco a partir do CEP, e o ReceitaWS retorna razao social, porte, socios, situacao cadastral e muito mais a partir do CNPJ. Este post mostra como integramos as duas APIs em formularios Vaadin.

## O fluxo completo

```
Cadastro de Empresa:

1. Usuario digita o CNPJ e pressiona ENTER
   |
   v  Valida formato (14 digitos)
   |
   v  Verifica se CNPJ ja existe no sistema
   |
   v  ReceitaWS API (GET /v1/cnpj/{cnpj})
   |
   +-- BAIXADA --> Bloqueia cadastro
   |
   +-- OK --> Preenche automaticamente:
        - Razao social, nome fantasia
        - Endereco completo (logradouro, bairro, cidade, UF, CEP)
        - Telefone, e-mail, porte, data de abertura
        - Exibe painel com dados completos (socios, CNAE, Simples)

2. Usuario digita o CEP
   |
   v  ViaCEP API (GET /ws/{cep}/json/)
   |
   +-- Encontrou --> Preenche logradouro, bairro, cidade, UF
   |
   +-- Nao encontrou --> Usuario preenche manualmente
```

## ReceitaWS: consulta de CNPJ na Receita Federal

### O servico

O ReceitaWS usa o `HttpClient` nativo do Java (nao o WebClient do Spring) por ser uma API publica simples que nao precisa de token:

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
                "Nao foi possivel consultar o CNPJ. "
                + "Tente novamente mais tarde.");
        }

        JsonNode root = objectMapper
            .readTree(response.body());

        // Erro funcional da API
        if (root.has("status")
                && root.get("status").isTextual()
                && !"OK".equalsIgnoreCase(
                    root.get("status").asText())) {
            throw new RuntimeException(
                "CNPJ nao encontrado ou "
                + "indisponivel para consulta.");
        }

        return objectMapper.treeToValue(
            root, ReceitaWs.class);
    }
}
```

**Por que HttpClient e nao WebClient?** A ReceitaWS e uma API publica sem autenticacao. O `HttpClient` nativo do Java 11+ e mais simples para esse caso — sem dependencia do Spring WebFlux.

### O modelo de resposta

A ReceitaWS retorna dados muito ricos. O modelo mapeia tudo com classes internas:

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
    private String tipo;       // MATRIZ ou FILIAL
    private String porte;      // ME, EPP, DEMAIS
    private String nome;       // Razao social
    private String fantasia;   // Nome fantasia
    private String abertura;   // Data de abertura

    @JsonProperty("atividadePrincipal")
    private List<Atividade> atividadePrincipal;

    @JsonProperty("atividadesSecundarias")
    private List<Atividade> atividadesSecundarias;

    @JsonProperty("natureza_juridica")
    private String naturezaJuridica;

    // Endereco
    private String logradouro;
    private String numero;
    private String complemento;
    private String cep;
    private String bairro;
    private String municipio;
    private String uf;

    // Contato
    private String email;
    private String telefone;

    // Situacao cadastral
    private String situacao;      // ATIVA, BAIXADA, etc.
    @JsonProperty("data_situacao")
    private String dataSituacao;
    @JsonProperty("motivo_situacao")
    private String motivoSituacao;

    @JsonProperty("capital_social")
    private Double capitalSocial;

    private List<Socio> qsa;     // Quadro societario
    private Simples simples;      // Optante Simples Nacional

    // Classes internas
    public static class Atividade {
        private String code;      // CNAE
        private String text;      // Descricao
    }

    public static class Socio {
        private String nome;
        private String qual;      // Qualificacao
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

Um unico CNPJ retorna razao social, nome fantasia, endereco completo, CNAE, quadro societario, regime tributario e situacao cadastral. Isso economiza muito preenchimento manual.

## Integracao no formulario Vaadin

### Botao de consulta no campo CNPJ

```java
// Botao inline dentro do campo CNPJ
Button btnConsultaCnpj = new Button("Pesquisar CNPJ");
btnConsultaCnpj.setIcon(VaadinIcon.SEARCH.create());
btnConsultaCnpj.addThemeVariants(
    ButtonVariant.LUMO_TERTIARY_INLINE,
    ButtonVariant.LUMO_SMALL);
btnConsultaCnpj.getStyle()
    .set("color", "var(--lumo-primary-color)")
    .set("font-weight", "600");

// Duas formas de disparar a consulta
btnConsultaCnpj.addClickListener(
    event -> consultarCnpj(true));
cgcField.addKeyPressListener(
    Key.ENTER, e -> consultarCnpj(true));

// Botao como sufixo do campo
cgcField.setSuffixComponent(btnConsultaCnpj);
```

O `setSuffixComponent()` coloca o botao dentro do campo, criando uma UX limpa onde o usuario digita e clica sem sair do campo.

### O metodo consultarCnpj()

```java
private void consultarCnpj(boolean forcar) {
    // 1. Valida formato
    String cnpj = Optional.ofNullable(
            cgcField.getValue())
        .orElse("")
        .replaceAll("[^0-9]", "");

    if (cnpj.length() != 14
            || !Utils.isCnpjValido(cnpj)) {
        Notification.show(
            "Informe um CNPJ valido.",
            3000, Position.MIDDLE);
        return;
    }

    // Cache: nao reconsulta o mesmo CNPJ
    if (!forcar
            && cnpj.equals(ultimoCnpjConsultado)) {
        return;
    }

    // 2. Verifica duplicidade ANTES da API
    //    (economiza creditos)
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
                "CNPJ ja cadastrado para "
                + codigo, 5000, Position.MIDDLE);
            cgcField.setInvalid(true);
            return;
        }
    }

    // 3. Consulta ReceitaWS
    try {
        ReceitaWs receita =
            receitaWsService.findCnpj(cnpj);

        // 4. Rejeita empresas BAIXADAS
        if ("BAIXADA".equalsIgnoreCase(
                receita.getSituacao())) {
            Notification.show(
                "CNPJ com situacao BAIXADA. "
                + "Nao pode ser cadastrado.",
                5000, Position.MIDDLE);
            cgcField.setInvalid(true);
            return;
        }

        // 5. Preenche formulario automaticamente
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

        // Data de abertura: converte dd/MM/yyyy
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
            "CNPJ consultado com sucesso.",
            3000, Position.MIDDLE);

    } catch (RuntimeException e) {
        Notification.show(
            e.getMessage(), 6000, Position.MIDDLE);
        clearCamposCnpj(true);
    }
}
```

**Pontos importantes:**
- Verifica duplicidade **antes** de chamar a API (economiza creditos da ReceitaWS)
- Rejeita empresas com situacao BAIXADA na Receita Federal
- Converte a data de abertura de `dd/MM/yyyy` para `ZonedDateTime`
- `binder.readBean(model)` atualiza todos os campos do formulario de uma vez

### ReceitaWsView: painel de detalhes

```java
public class ReceitaWsView extends VerticalLayout {

    private TextField statusField =
        criarField("Status");
    private TextField cnpjField =
        criarField("CNPJ");
    private TextField tipoField =
        criarField("Tipo");
    private TextField porteField =
        criarField("Porte");
    private TextField nomeField =
        criarField("Razao Social");
    private TextField fantasiaField =
        criarField("Nome Fantasia");
    private TextField naturezaField =
        criarField("Natureza Juridica");
    private TextField situacaoField =
        criarField("Situacao Cadastral");
    private TextField capitalField =
        criarField("Capital Social");
    private TextArea sociosField =
        criarTextArea("Quadro Societario");
    private TextArea atividadesPrincipaisField =
        criarTextArea("Atividade Principal");
    private TextField simplesField =
        criarField("Optante Simples");
    // ... mais campos

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

        // Capital social formatado
        capitalField.setValue(
            dados.getCapitalSocial() != null
                ? "R$ " + String.format(
                    Locale.getDefault(), "%,.2f",
                    dados.getCapitalSocial())
                : "");

        // Quadro societario
        sociosField.setValue(
            Optional.ofNullable(dados.getQsa())
                .filter(lista -> !lista.isEmpty())
                .map(lista -> lista.stream()
                    .map(s -> s.getNome()
                        + " (" + s.getQual() + ")")
                    .collect(
                        Collectors.joining("\n")))
                .orElse(""));

        // CNAE
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

        // Simples Nacional
        if (dados.getSimples() != null) {
            simplesField.setValue(
                Boolean.TRUE.equals(
                    dados.getSimples().getOptante())
                    ? "Sim" : "Nao");
        }
    }
}
```

O painel e exibido dentro de um `Details` (acordeao) — o usuario pode expandir para ver todos os dados da Receita sem poluir o formulario principal.

### Limpeza dos campos em caso de erro

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

O `manterCnpj` preserva o valor digitado no campo CNPJ enquanto limpa todos os outros campos. Isso evita que o usuario precise redigitar o CNPJ.

## ViaCEP: consulta de endereco por CEP

### O servico

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
                            "Erro HTTP: " + status));
                })
            .bodyToMono(String.class)
            .block();

        JsonNode root = objectMapper
            .readTree(rawResponse);

        // A API retorna {"erro": true} para CEP invalido
        if (root.has("erro")
                && root.get("erro").asBoolean()) {
            throw new RuntimeException(
                "CEP nao encontrado.");
        }

        return objectMapper.readValue(
            root.toString(), Viacep.class);
    }
}
```

### O modelo de resposta

```java
@JsonIgnoreProperties(ignoreUnknown = true)
@Getter @Setter
public class Viacep {
    private String cep;
    private String logradouro;
    private String complemento;
    private String bairro;

    @JsonProperty("localidade")
    private String cidade;       // API chama "localidade"

    private String uf;
    private String ibge;         // Codigo IBGE do municipio
    private String ddd;          // DDD telefone
    private String siafi;        // Codigo SIAFI
    private String estado;
    private String regiao;
}
```

Note o `@JsonProperty("localidade")`: a API ViaCEP retorna o campo como `localidade`, mas no modelo mapeamos como `cidade` para consistencia com o restante do sistema.

### ViacepView: exibindo dados do CEP

```java
public class ViacepView extends VerticalLayout {

    private final TextField cepField =
        criarField("CEP");
    private final TextField logradouroField =
        criarField("Logradouro");
    private final TextField bairroField =
        criarField("Bairro");
    private final TextField cidadeField =
        criarField("Cidade");
    private final TextField ufField =
        criarField("UF");
    private final TextField dddField =
        criarField("DDD");
    private final TextField ibgeField =
        criarField("IBGE");

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

## Verificacao de CNPJ em lote (Grid)

Alem do formulario individual, a consulta tambem funciona em lote na lista de empresas:

```java
// No EmpresasComponent (Grid de empresas)
private void checkCnpj(Empresa empresa,
        Button button) {

    String cgc = empresa.getCgc();
    if (cgc == null || cgc.isBlank()) {
        Notification.show("CNPJ nao informado",
            3000, Position.MIDDLE);
        return;
    }

    // Feedback visual no botao
    button.setEnabled(false);
    button.setText("Consultando...");

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
            // Salva no banco...
        }
    } catch (Exception e) {
        Notification.show(e.getMessage(),
            5000, Position.MIDDLE);
    } finally {
        button.setEnabled(true);
        button.setText("Consultar");
    }
}
```

Isso permite verificar a situacao cadastral de empresas ja cadastradas — util para descobrir CNPJs que foram baixados depois do cadastro.

## Padrao de tratamento de erros

Ambas as APIs podem falhar por motivos diferentes. O tratamento cobre cada caso:

```java
// ReceitaWS - erros possiveis:
try {
    ReceitaWs receita =
        receitaWsService.findCnpj(cnpj);
    // ...
} catch (RuntimeException e) {
    // Mensagens ja vem prontas do service:
    // - "CNPJ nao encontrado ou indisponivel"
    // - "Servico indisponivel" (SSL)
    // - "Erro ao consultar" (timeout/IO)
    Notification.show(e.getMessage(),
        6000, Position.MIDDLE);
}

// ViaCEP - erros possiveis:
try {
    Viacep endereco =
        viacepService.buscarCep(cep);
    // ...
} catch (RuntimeException e) {
    // - "CEP nao encontrado"
    // - "Erro HTTP: 400"
    Notification.show(e.getMessage(),
        3000, Position.MIDDLE);
}
```

O servico encapsula as mensagens de erro — a view nunca precisa saber detalhes do HTTP.

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Preencher endereco completo manualmente | ViaCEP resolve tudo a partir do CEP |
| CNPJ precisa de validacao na Receita | ReceitaWS retorna situacao cadastral e dados completos |
| Empresa com CNPJ baixado nao pode ser cadastrada | Verificacao de status BAIXADA antes de aceitar |
| Consulta duplicada gasta creditos da API | Verifica CNPJ no banco ANTES de chamar a ReceitaWS |
| Mesmo CNPJ em dois cadastros | Validacao de unicidade antes da consulta externa |
| API campo "localidade" vs modelo "cidade" | `@JsonProperty("localidade")` no DTO |
| Dados completos poluem o formulario | `Details` (acordeao) para exibir painel da ReceitaWS |
| Formato CNPJ pode variar (com/sem pontos) | `replaceAll("[^0-9]", "")` normaliza para 14 digitos |
| API publica nao precisa de WebClient reativo | `HttpClient` nativo do Java 11+ e suficiente |
| Data de abertura em dd/MM/yyyy | `DateTimeFormatter.ofPattern("dd/MM/yyyy")` para converter |

---

Com ViaCEP + ReceitaWS integrados ao formulario Vaadin, o cadastro de empresas fica quase automatico: o usuario digita o CNPJ, pressiona ENTER, e o formulario se preenche sozinho com razao social, endereco, porte e situacao cadastral. APIs publicas brasileiras que economizam horas de preenchimento manual.
