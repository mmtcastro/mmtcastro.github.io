---
layout: post
title: "AbstractModelDoc, AbstractService e AbstractViewDoc: a decisao arquitetural que acelera a migracao Domino para Vaadin"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, architecture]
---

Quando comecamos a migrar um ERP inteiro do HCL Domino para Java/Vaadin, a primeira pergunta foi: "Como evitar reescrever a mesma infraestrutura para cada uma das dezenas de entidades?" A resposta foi criar tres classes abstratas que encapsulam tudo que e comum — modelo, servico e view. Este post explica essa decisao, o que cada classe oferece, e quanto codigo voce economiza ao criar uma nova entidade.

## O problema: repetir infraestrutura para cada entidade

No Domino, cada banco de dados (NSF) tem suas proprias forms, views e agents. Ao migrar para Java/Vaadin com DRAPI, cada entidade precisaria de:

- **Modelo**: campos do Domino (@unid, @meta, revision, Form), geracao de ID, comparacao, attachment control
- **Service**: chamadas HTTP para CRUD (GET, POST, PUT, DELETE), tratamento de erro, token JWT, paginacao, multivalue, anexos
- **View**: layout de formulario responsivo, botoes Edit/Save/Cancel/Delete, binder, modo leitura/edicao, permissoes ACL

Sem abstraccao, cada entidade repetiria **2.400+ linhas** de infraestrutura. Com 30+ entidades, isso significaria 70.000+ linhas de codigo duplicado.

## A solucao: tres camadas abstratas

```
AbstractModelDoc (700+ linhas)
  ├─ Metadados Domino (@unid, @meta, revision, Form)
  ├─ Geracao de ID ({scope}_{form}_{uuid})
  ├─ Campos padrao (Codigo, Status, Autor, Data, Criacao...)
  ├─ Controle de anexos (uploads, downloads, exclusoes)
  ├─ Inner classes (Meta, RichText, UploadedFile)
  └─ equals/hashCode/compareTo por Id

AbstractService<T extends AbstractModelDoc> (1.200+ linhas)
  ├─ CRUD completo via DRAPI (findByUnid, findByCodigo, save, delete)
  ├─ Paginacao com ordenacao e full-text search
  ├─ Relacionamentos pai-filho (findChildren, hasChildren)
  ├─ Processamento de anexos (upload, download, delete)
  ├─ Serializacao multivalue (populaMultivalues, flattenForDomino)
  ├─ Resolucao automatica de scope e form
  └─ Token JWT e WebClient configurado

AbstractViewDoc<T extends AbstractModelDoc> (515+ linhas)
  ├─ FormLayout responsivo (1→2→3 colunas)
  ├─ Binder com validacao
  ├─ Botoes CRUD com estados automaticos
  ├─ Modo leitura/edicao com toggle
  ├─ Integracao com Anexos
  ├─ Dialogo de confirmacao para delete
  ├─ Navegacao (breadcrumb, back, rota com :unid)
  └─ Hooks para permissoes ACL
```

## AbstractModelDoc: o modelo base

Toda entidade que vem do Domino via DRAPI herda de `AbstractModelDoc`. Isso garante que campos comuns existam em todos os modelos:

```java
public abstract class AbstractModelDoc implements Comparable<AbstractModelDoc> {

    // Metadados do Domino
    @JsonProperty("@meta")
    private Meta meta;

    @JsonProperty("@unid")
    private String unid;

    @JsonProperty("Form")
    private String form;

    // Identificacao de negocio
    private String Id;
    private String Codigo;
    private String IdOrigem;        // Relacionamento pai-filho

    // Campos padrao
    private String Status;
    private String Autor;
    private String Responsavel;
    private String Nome;
    private String Descricao;
    private ZonedDateTime Criacao;
    private LocalDate Data;

    // Controle de acesso
    private TreeSet<String> Autores;
    private TreeSet<String> Leitores;

    // Controle de anexos (@JsonIgnore = nao vai para o JSON)
    @JsonProperty("$FILES")
    protected List<String> fileNames;

    @JsonIgnore
    protected List<UploadedFile> uploads = new ArrayList<>();

    @JsonIgnore
    protected List<String> anexosParaExcluir = new ArrayList<>();
}
```

### O que o modelo base fornece de graca

**Geracao de ID unico:**
```java
public String generateNewModelId() {
    // Gera ID no formato: {scope}_{form}_{uuid}
    // Ex: "empresas_Empresa_a1b2c3d4-e5f6-..."
    String scope = Utils.getScopeFromClass(this.getClass());
    String form = this.getClass().getSimpleName();
    return scope + "_" + form + "_" + UUID.randomUUID().toString();
}
```

**Inicializacao automatica:**
```java
public void init() {
    this.form = this.getClass().getSimpleName();
    this.Id = generateNewModelId();
    this.Criacao = ZonedDateTime.now();
    this.Autor = UtilsSession.getCurrentUser().getName();
}
```

**Inner classes reutilizaveis:**
```java
public static class Meta {
    private String unid;
    private String revision;
    private ZonedDateTime created;
    private ZonedDateTime lastmodified;
    private boolean editable;
}

public static class RichText {
    private String type;       // "text/html"
    private String encoding;   // "BASE64"
    private String content;    // HTML codificado
}

public static class UploadedFile {
    private String fileName;
    private byte[] fileData;
}
```

### Exemplo: modelo concreto com 29 linhas

```java
@Getter @Setter
@JsonIgnoreProperties(ignoreUnknown = true)
public class Colaborador extends AbstractModelDoc {

    @JsonAlias({ "Funcionario", "funcionario" })
    private String funcionario;

    public Colaborador() {
        super();
    }
}
```

Essas 29 linhas herdam **700+ linhas** de infraestrutura: metadados, ID, anexos, comparacao, inicializacao.

### Exemplo: modelo com multivalue (73 linhas)

```java
@Getter @Setter
public class Vertical extends AbstractModelDoc {

    @JsonIgnore
    private Unidades unidades;

    @JsonDeserialize(using = BodyDeserializer.class)
    protected RichText obs;

    // Inner class para cada item da lista
    @Getter @Setter
    public static class Unidade extends AbstractModelDocMultivalue {
        private String responsavel;
        private String estado;
        private LocalDate criacao;
        private Double valor;
        private String status;
    }

    // Wrapper que herda operacoes de lista
    @Getter @Setter
    public static class Unidades
            extends AbstractModelListaMultivalue<Unidade> {
    }

    public Unidades getUnidades() {
        if (unidades == null) unidades = new Unidades();
        return unidades;
    }
}
```

## AbstractService: o CRUD completo via DRAPI

O service base encapsula **todas** as chamadas HTTP para o DRAPI. A classe concreta so precisa de um construtor:

```java
@Service
public class ColaboradorService extends AbstractService<Colaborador> {

    public ColaboradorService(DominoAclService dominoAclService) {
        super(dominoAclService);
    }
}
```

Essas 7 linhas herdam **1.200+ linhas** de funcionalidade.

### Resolucao automatica de scope e form

O construtor do AbstractService usa generics para resolver automaticamente o banco de dados (scope) e o form:

```java
public AbstractService(DominoAclService dominoAclService) {
    this.dominoAclService = dominoAclService;

    // Descobre a classe do modelo via generics
    this.modelClass = (Class<T>) ((ParameterizedType)
        getClass().getGenericSuperclass())
        .getActualTypeArguments()[0];

    // Extrai o scope do pacote: com.example.app.empresas.model.Empresa
    //                                               ^^^^^^^^
    this.scope = Utils.getScopeFromClass(modelClass);  // "empresas"
    this.form = modelClass.getSimpleName();             // "Empresa"
}
```

Isso significa que `EmpresaService`, `ColaboradorService`, `CargoService` — todos sabem automaticamente qual banco de dados acessar no DRAPI.

### O que o service base fornece de graca

**Busca por UNID:**
```java
public Response<T> findByUnid(String unid) {
    // GET /document/{unid}?dataSource={scope}&richTextAs=mime
}
```

**Busca por codigo:**
```java
public Response<T> findByCodigo(String codigo) {
    // GET /lists/_intraCodigos?dataSource={scope}&key={codigo}&key={form}
}
```

**Listagem paginada com ordenacao e busca:**
```java
public List<T> findAllByCodigo(int offset, int count,
        List<QuerySortOrder> sortOrders, String search) {
    // GET /lists/{view}?dataSource={scope}&count={count}&start={offset}
    //     &column=Codigo&direction={asc|desc}&ftSearchQuery={search}
    // Retorna lista + this.totalCount (do header x-totalcount)
}
```

**Save (cria ou atualiza):**
```java
public SaveResponse save(T model) {
    if (model.getMeta() == null) {
        // POST /document?dataSource={scope}&richTextAs=mime
    } else {
        model.newRevision();
        // PUT /document/{unid}?dataSource={scope}&revision={rev}
    }
}
```

**Delete com validacao:**
```java
public DeleteResponse delete(T model) {
    List<String> reasons = getPodeDeletar(model);
    if (reasons != null && !reasons.isEmpty()) {
        return new DeleteResponse(403, reasons);
    }
    // DELETE /document/{unid}?dataSource={scope}
}
```

**Relacionamentos pai-filho:**
```java
public List<String> findChildrenIdsByIdOrigem(String idOrigem) {
    // GET /lists/_intraIdsOrigem?dataSource={scope}&key={idOrigem}
}

public boolean hasChildren(String idOrigem) { ... }
public int countChildren(String idOrigem) { ... }
```

**Anexos e multivalue:**
```java
public boolean processarAnexos(T model, String unid) { ... }
public void populaMultivalues(Object model, JsonNode root) { ... }
public ObjectNode flattenForDomino(T model) { ... }
```

### Hooks para customizacao

O service base oferece hooks que as subclasses podem sobrescrever:

```java
// Regra de exclusao customizada
@Override
protected List<String> getPodeDeletar(Empresa model) {
    if (countChildren(model.getId(), "empresa") > 0) {
        return List.of("Nao e possivel excluir: empresa tem contatos vinculados");
    }
    return null; // permite exclusao
}
```

### Exemplo: service com queries especializadas

```java
@Service
public class EspelhoService extends AbstractService<Espelho> {

    public EspelhoService(DominoAclService dominoAclService) {
        super(dominoAclService);
    }

    // CRUD herdado: findByUnid, findByCodigo, save, delete

    // Queries especializadas adicionadas:
    public List<Espelho> findByDateRange(LocalDate start, LocalDate end) {
        String dql = String.format(
            "Form = 'Espelho' and DataFaturamento >= @dt('%s') "
            + "and DataFaturamento <= @dt('%s')",
            start, end);

        Map<String, Object> payload = Map.of("query", dql);
        // POST /query?dataSource={scope}
        // ... retorna lista filtrada
    }
}
```

## AbstractViewDoc: o formulario pronto

A view base fornece o formulario completo com CRUD, navegacao e permissoes. A classe concreta so precisa implementar a configuracao de campos:

```java
@PageTitle("Cargo")
@Route(value = "cargo/:unid?", layout = MainLayout.class)
@RolesAllowed("ROLE_EVERYONE")
public class CargoView extends AbstractViewDoc<Cargo> {

    private final CargoService cargoService;
    private final DominoAclService dominoAclService;

    // Campos do formulario
    private final TextField codigoField = new TextField("Codigo");
    private final TextField descricaoField = new TextField("Descricao");
    private final TextField statusField = new TextField("Status");
    private final RichTextView obsField = new RichTextView();

    public CargoView(CargoService cargoService,
            DominoAclService dominoAclService) {
        this.cargoService = cargoService;
        this.dominoAclService = dominoAclService;
        buildView();
    }

    @Override
    protected void configureForm(FormLayout form) {
        form.add(codigoField, descricaoField, statusField);
        form.setColspan(obsField, 2);
        form.add(obsField);
    }

    @Override
    protected void configureBinder(Binder<Cargo> binder) {
        binder.forField(codigoField)
            .bind(Cargo::getCodigo, Cargo::setCodigo);
        binder.forField(descricaoField)
            .bind(Cargo::getDescricao, Cargo::setDescricao);
        binder.forField(statusField)
            .bind(Cargo::getStatus, Cargo::setStatus);
        binder.forField(obsField)
            .withNullRepresentation(new AbstractModelDoc.RichText())
            .bind(Cargo::getObs, Cargo::setObs);
    }

    @Override
    protected void updateReadOnlyFields() {
        codigoField.setReadOnly(isReadOnly || !isNovo);
        descricaoField.setReadOnly(isReadOnly);
        statusField.setReadOnly(isReadOnly);
        obsField.setReadOnly(isReadOnly);
    }

    @Override
    protected void configurePermissions() {
        User user = UtilsSession.getCurrentUser();
        if (user != null && getActionBar() != null) {
            var aclEntries = dominoAclService.loadAcl(
                cargoService.getScope());
            DominoEffectiveAccess access =
                dominoAclService.resolveEffectiveAccess(
                    aclEntries, user);
            getActionBar().setPermissions(access);
        }
    }

    // Metodos obrigatorios (template)
    @Override
    protected AbstractService<Cargo> getService() {
        return cargoService;
    }

    @Override
    protected Class<Cargo> getModelClass() {
        return Cargo.class;
    }

    @Override
    protected String getRouteBase() { return "cargo"; }

    @Override
    protected String getViewTitle() { return "Cargo"; }

    @Override
    protected Class<? extends Component> getBackView() {
        return CargosView.class;
    }

    @Override
    protected Cargo createNewModel() {
        Cargo novo = new Cargo();
        novo.setData(LocalDate.now());
        return novo;
    }
}
```

**128 linhas** que herdam **515+ linhas** de infraestrutura de UI.

### O que a view base fornece de graca

**Ciclo de vida completo:**
```java
// beforeEnter() automatico:
// - Se unid == "novo": cria modelo via createNewModel()
// - Se unid informado: carrega via getService().findByUnid()
// - Configura binder, form, permissoes, anexos
```

**Operacoes CRUD automaticas:**
```java
protected void save() {
    binder.writeBean(model);                        // UI → Modelo
    SaveResponse response = getService().save(model); // Modelo → DRAPI
    if (response.isSuccess()) {
        // Processa anexos, atualiza URL, notifica sucesso
    }
}

protected void cancel() {
    // Se novo: volta para lista
    // Se existente: recarrega do DRAPI
}

protected void delete() {
    // Dialogo de confirmacao → getService().delete(model) → volta para lista
}
```

**Layout responsivo automatico:**
```java
// FormLayout com breakpoints:
// < 500px: 1 coluna
// 500-900px: 2 colunas
// > 900px: 3 colunas
```

**Modo leitura/edicao:**
```java
protected void edit() {
    isReadOnly = false;
    updateReadOnlyState();   // Chama updateReadOnlyFields() da subclasse
    updateButtonsState();    // Mostra Save/Cancel, esconde Edit
}
```

**Footer com metadados:**
```
Criado em: 01/03/2026 14:30 • Autor: joao.silva
```

### Hooks disponiveis para customizacao

```java
// Inicializacao de modelo novo
protected void onNewModel(T model) { }

// Pos-carregamento de modelo existente
protected void afterLoad(T model) { }

// Configuracao de permissoes ACL
protected void configurePermissions() { }

// Controle de campos readonly
protected void updateReadOnlyFields() { }

// Usar CrudActionBar em vez de botoes simples?
protected boolean useCrudActionBar() { return false; }

// Habilitar anexos?
protected boolean useAnexos() { return true; }

// Duplo clique para editar?
protected boolean allowDoubleClickEdit() { return true; }
```

## Quando NAO usar a classe abstrata

Nem toda view precisa de `AbstractViewDoc`. Views somente leitura ou com layout muito diferente podem ser criadas diretamente:

```java
@Route(value = "espelho", layout = MainLayout.class)
public class EspelhoView extends VerticalLayout
        implements HasUrlParameter<String> {

    private final EspelhoService service;

    @Override
    public void setParameter(BeforeEvent event, String parameter) {
        Espelho espelho = service.findByUnid(parameter).getModel();
        buildReadOnlyView(espelho);
    }

    private void buildReadOnlyView(Espelho espelho) {
        // Layout customizado com cards, secoes, scroll
        // Sem botoes de edicao
        // 60+ campos somente leitura
    }
}
```

O `EspelhoService` ainda herda de `AbstractService` (para usar findByUnid), mas a view nao precisa do framework de formulario.

## Os numeros: quanto codigo voce economiza

| Componente | Linhas herdadas | Linhas escritas | Percentual escrito |
|---|---|---|---|
| Modelo simples (Colaborador) | 700+ | 29 | 4% |
| Modelo com multivalue (Vertical) | 700+ | 73 | 9% |
| Service minimo (ColaboradorService) | 1.200+ | 7 | 0.6% |
| Service com queries (EspelhoService) | 1.200+ | 385 | 24% |
| View CRUD completa (CargoView) | 515+ | 128 | 20% |

**Para uma entidade tipica (modelo + service + view)**, o desenvolvedor escreve ~160 linhas e herda ~2.400 linhas. Isso e **6% de codigo customizado** para 100% de funcionalidade CRUD.

## A decisao: vale a pena?

### Vantagens

| Beneficio | Impacto |
|---|---|
| Elimina duplicacao | 30+ entidades × 2.400 linhas = 72.000 linhas que nao existem |
| Consistencia de UX | Todos os formularios seguem o mesmo padrao visual |
| Velocidade de desenvolvimento | Nova entidade CRUD completa em ~30 minutos |
| Manutencao centralizada | Correcao em AbstractService beneficia todos os services |
| Compliance tributario | Campos padrao (impostos, NFe) ja estao no modelo base |
| Seguranca uniforme | ACL e permissoes tratados de forma consistente |

### Riscos e mitigacoes

| Risco | Mitigacao |
|---|---|
| Classe base muito grande | Hooks permitem customizacao sem modificar o base |
| Heranca rigida | Sempre possivel nao usar o abstract (como EspelhoView) |
| Curva de aprendizado | Uma vez entendido, produtividade aumenta exponencialmente |
| Mudanca no base quebra tudo | Testes unitarios nos metodos base |

### Quando usar cada abordagem

```
Nova entidade CRUD?
  ├─ Sim → AbstractModelDoc + AbstractService + AbstractViewDoc
  │        (128 linhas para CRUD completo)
  │
  └─ View somente leitura ou layout customizado?
      ├─ Sim → AbstractModelDoc + AbstractService + VerticalLayout
      │        (usa service herdado, view customizada)
      │
      └─ Entidade sem DRAPI (ex: config local)?
          └─ Classe Java normal (sem heranca)
```

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Domino tem quirks (UNID, revision, multivalue) | Encapsulados em AbstractModelDoc e AbstractService |
| Cada banco tem seu scope | Resolucao automatica via pacote Java do modelo |
| Forms tem campos muito diferentes | Hooks de customizacao (configureForm, configureBinder) |
| Algumas views nao precisam de CRUD | Nao forcar heranca — usar AbstractService sem AbstractViewDoc |
| Regras de exclusao variam por entidade | Hook getPodeDeletar() retorna lista de motivos |
| ACL varia por banco | configurePermissions() e CrudActionBar por view |

---

A decisao de investir nas tres classes abstratas custou algumas semanas no inicio da migracao, mas acelerou enormemente o restante do projeto. Cada nova entidade migrada do Domino segue o mesmo padrao: criar modelo, criar service, criar view — e 95% da infraestrutura ja esta pronta.
