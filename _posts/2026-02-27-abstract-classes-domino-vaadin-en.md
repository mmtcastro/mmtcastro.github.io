---
layout: post
title: "AbstractModelDoc, AbstractService and AbstractViewDoc: the architectural decision that accelerates Domino to Vaadin migration"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, architecture]
lang: en
---

When we started migrating an entire ERP from HCL Domino to Java/Vaadin, the first question was: "How do we avoid rewriting the same infrastructure for each of the dozens of entities?" The answer was to create three abstract classes that encapsulate everything that is common — model, service and view. This post explains that decision, what each class provides, and how much code you save when creating a new entity.

## The problem: repeating infrastructure for each entity

In Domino, each database (NSF) has its own forms, views and agents. When migrating to Java/Vaadin with DRAPI, each entity would need:

- **Model**: Domino fields (@unid, @meta, revision, Form), ID generation, comparison, attachment control
- **Service**: HTTP calls for CRUD (GET, POST, PUT, DELETE), error handling, JWT token, pagination, multivalue, attachments
- **View**: responsive form layout, Edit/Save/Cancel/Delete buttons, binder, read/edit mode, ACL permissions

Without abstraction, each entity would repeat **2,400+ lines** of infrastructure. With 30+ entities, that would mean 70,000+ lines of duplicated code.

## The solution: three abstract layers

```
AbstractModelDoc (700+ lines)
  ├─ Domino metadata (@unid, @meta, revision, Form)
  ├─ ID generation ({scope}_{form}_{uuid})
  ├─ Standard fields (Codigo, Status, Autor, Data, Criacao...)
  ├─ Attachment control (uploads, downloads, deletions)
  ├─ Inner classes (Meta, RichText, UploadedFile)
  └─ equals/hashCode/compareTo by Id

AbstractService<T extends AbstractModelDoc> (1,200+ lines)
  ├─ Full CRUD via DRAPI (findByUnid, findByCodigo, save, delete)
  ├─ Pagination with sorting and full-text search
  ├─ Parent-child relationships (findChildren, hasChildren)
  ├─ Attachment processing (upload, download, delete)
  ├─ Multivalue serialization (populaMultivalues, flattenForDomino)
  ├─ Automatic scope and form resolution
  └─ JWT token and configured WebClient

AbstractViewDoc<T extends AbstractModelDoc> (515+ lines)
  ├─ Responsive FormLayout (1→2→3 columns)
  ├─ Binder with validation
  ├─ CRUD buttons with automatic states
  ├─ Read/edit mode with toggle
  ├─ Attachment integration
  ├─ Confirmation dialog for delete
  ├─ Navigation (breadcrumb, back, route with :unid)
  └─ Hooks for ACL permissions
```

## AbstractModelDoc: the base model

Every entity that comes from Domino via DRAPI extends `AbstractModelDoc`. This ensures common fields exist in all models:

```java
public abstract class AbstractModelDoc implements Comparable<AbstractModelDoc> {

    // Domino metadata
    @JsonProperty("@meta")
    private Meta meta;

    @JsonProperty("@unid")
    private String unid;

    @JsonProperty("Form")
    private String form;

    // Business identification
    private String Id;
    private String Codigo;
    private String IdOrigem;        // Parent-child relationship

    // Standard fields
    private String Status;
    private String Autor;
    private String Responsavel;
    private String Nome;
    private String Descricao;
    private ZonedDateTime Criacao;
    private LocalDate Data;

    // Access control
    private TreeSet<String> Autores;
    private TreeSet<String> Leitores;

    // Attachment control (@JsonIgnore = not included in JSON)
    @JsonProperty("$FILES")
    protected List<String> fileNames;

    @JsonIgnore
    protected List<UploadedFile> uploads = new ArrayList<>();

    @JsonIgnore
    protected List<String> anexosParaExcluir = new ArrayList<>();
}
```

### What the base model provides for free

**Unique ID generation:**
```java
public String generateNewModelId() {
    // Generates ID in format: {scope}_{form}_{uuid}
    // Ex: "empresas_Empresa_a1b2c3d4-e5f6-..."
    String scope = Utils.getScopeFromClass(this.getClass());
    String form = this.getClass().getSimpleName();
    return scope + "_" + form + "_" + UUID.randomUUID().toString();
}
```

**Automatic initialization:**
```java
public void init() {
    this.form = this.getClass().getSimpleName();
    this.Id = generateNewModelId();
    this.Criacao = ZonedDateTime.now();
    this.Autor = UtilsSession.getCurrentUser().getName();
}
```

**Reusable inner classes:**
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
    private String content;    // Encoded HTML
}

public static class UploadedFile {
    private String fileName;
    private byte[] fileData;
}
```

### Example: concrete model with 29 lines

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

These 29 lines inherit **700+ lines** of infrastructure: metadata, ID, attachments, comparison, initialization.

### Example: model with multivalue (73 lines)

```java
@Getter @Setter
public class Vertical extends AbstractModelDoc {

    @JsonIgnore
    private Unidades unidades;

    @JsonDeserialize(using = BodyDeserializer.class)
    protected RichText obs;

    // Inner class for each list item
    @Getter @Setter
    public static class Unidade extends AbstractModelDocMultivalue {
        private String responsavel;
        private String estado;
        private LocalDate criacao;
        private Double valor;
        private String status;
    }

    // Wrapper that inherits list operations
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

## AbstractService: full CRUD via DRAPI

The base service encapsulates **all** HTTP calls to DRAPI. The concrete class only needs a constructor:

```java
@Service
public class ColaboradorService extends AbstractService<Colaborador> {

    public ColaboradorService(DominoAclService dominoAclService) {
        super(dominoAclService);
    }
}
```

These 7 lines inherit **1,200+ lines** of functionality.

### Automatic scope and form resolution

The AbstractService constructor uses generics to automatically resolve the database (scope) and form:

```java
public AbstractService(DominoAclService dominoAclService) {
    this.dominoAclService = dominoAclService;

    // Discovers the model class via generics
    this.modelClass = (Class<T>) ((ParameterizedType)
        getClass().getGenericSuperclass())
        .getActualTypeArguments()[0];

    // Extracts scope from package: com.example.app.empresas.model.Empresa
    //                                              ^^^^^^^^
    this.scope = Utils.getScopeFromClass(modelClass);  // "empresas"
    this.form = modelClass.getSimpleName();             // "Empresa"
}
```

This means `EmpresaService`, `ColaboradorService`, `CargoService` — all automatically know which DRAPI database to access.

### What the base service provides for free

**Find by UNID:**
```java
public Response<T> findByUnid(String unid) {
    // GET /document/{unid}?dataSource={scope}&richTextAs=mime
}
```

**Find by business code:**
```java
public Response<T> findByCodigo(String codigo) {
    // GET /lists/_intraCodigos?dataSource={scope}&key={codigo}&key={form}
}
```

**Paginated listing with sorting and search:**
```java
public List<T> findAllByCodigo(int offset, int count,
        List<QuerySortOrder> sortOrders, String search) {
    // GET /lists/{view}?dataSource={scope}&count={count}&start={offset}
    //     &column=Codigo&direction={asc|desc}&ftSearchQuery={search}
    // Returns list + this.totalCount (from x-totalcount header)
}
```

**Save (create or update):**
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

**Delete with validation:**
```java
public DeleteResponse delete(T model) {
    List<String> reasons = getPodeDeletar(model);
    if (reasons != null && !reasons.isEmpty()) {
        return new DeleteResponse(403, reasons);
    }
    // DELETE /document/{unid}?dataSource={scope}
}
```

**Parent-child relationships:**
```java
public List<String> findChildrenIdsByIdOrigem(String idOrigem) {
    // GET /lists/_intraIdsOrigem?dataSource={scope}&key={idOrigem}
}

public boolean hasChildren(String idOrigem) { ... }
public int countChildren(String idOrigem) { ... }
```

**Attachments and multivalue:**
```java
public boolean processarAnexos(T model, String unid) { ... }
public void populaMultivalues(Object model, JsonNode root) { ... }
public ObjectNode flattenForDomino(T model) { ... }
```

### Hooks for customization

The base service provides hooks that subclasses can override:

```java
// Custom deletion rule
@Override
protected List<String> getPodeDeletar(Empresa model) {
    if (countChildren(model.getId(), "empresa") > 0) {
        return List.of("Cannot delete: company has linked contacts");
    }
    return null; // allow deletion
}
```

### Example: service with specialized queries

```java
@Service
public class EspelhoService extends AbstractService<Espelho> {

    public EspelhoService(DominoAclService dominoAclService) {
        super(dominoAclService);
    }

    // Inherited CRUD: findByUnid, findByCodigo, save, delete

    // Specialized queries added:
    public List<Espelho> findByDateRange(LocalDate start, LocalDate end) {
        String dql = String.format(
            "Form = 'Espelho' and DataFaturamento >= @dt('%s') "
            + "and DataFaturamento <= @dt('%s')",
            start, end);

        Map<String, Object> payload = Map.of("query", dql);
        // POST /query?dataSource={scope}
        // ... returns filtered list
    }
}
```

## AbstractViewDoc: the ready-made form

The base view provides the complete form with CRUD, navigation and permissions. The concrete class only needs to implement field configuration:

```java
@PageTitle("Cargo")
@Route(value = "cargo/:unid?", layout = MainLayout.class)
@RolesAllowed("ROLE_EVERYONE")
public class CargoView extends AbstractViewDoc<Cargo> {

    private final CargoService cargoService;
    private final DominoAclService dominoAclService;

    // Form fields
    private final TextField codigoField = new TextField("Code");
    private final TextField descricaoField = new TextField("Description");
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

    // Required methods (template)
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

**128 lines** that inherit **515+ lines** of UI infrastructure.

### What the base view provides for free

**Complete lifecycle:**
```java
// Automatic beforeEnter():
// - If unid == "novo": creates model via createNewModel()
// - If unid provided: loads via getService().findByUnid()
// - Sets up binder, form, permissions, attachments
```

**Automatic CRUD operations:**
```java
protected void save() {
    binder.writeBean(model);                        // UI → Model
    SaveResponse response = getService().save(model); // Model → DRAPI
    if (response.isSuccess()) {
        // Process attachments, update URL, notify success
    }
}

protected void cancel() {
    // If new: navigate back to list
    // If existing: reload from DRAPI
}

protected void delete() {
    // Confirmation dialog → getService().delete(model) → navigate back
}
```

**Automatic responsive layout:**
```java
// FormLayout with breakpoints:
// < 500px: 1 column
// 500-900px: 2 columns
// > 900px: 3 columns
```

**Read/edit mode:**
```java
protected void edit() {
    isReadOnly = false;
    updateReadOnlyState();   // Calls subclass updateReadOnlyFields()
    updateButtonsState();    // Shows Save/Cancel, hides Edit
}
```

**Metadata footer:**
```
Created: 03/01/2026 14:30 • Author: john.doe
```

### Available hooks for customization

```java
// New model initialization
protected void onNewModel(T model) { }

// Post-load processing for existing model
protected void afterLoad(T model) { }

// ACL permission configuration
protected void configurePermissions() { }

// Read-only field control
protected void updateReadOnlyFields() { }

// Use CrudActionBar instead of simple buttons?
protected boolean useCrudActionBar() { return false; }

// Enable attachments?
protected boolean useAnexos() { return true; }

// Double-click to edit?
protected boolean allowDoubleClickEdit() { return true; }
```

## When NOT to use the abstract class

Not every view needs `AbstractViewDoc`. Read-only views or views with very different layouts can be created directly:

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
        // Custom layout with cards, sections, scroll
        // No edit buttons
        // 60+ read-only fields
    }
}
```

`EspelhoService` still extends `AbstractService` (to use findByUnid), but the view does not need the form framework.

## The numbers: how much code you save

| Component | Inherited lines | Written lines | Percentage written |
|---|---|---|---|
| Simple model (Colaborador) | 700+ | 29 | 4% |
| Model with multivalue (Vertical) | 700+ | 73 | 9% |
| Minimal service (ColaboradorService) | 1,200+ | 7 | 0.6% |
| Service with queries (EspelhoService) | 1,200+ | 385 | 24% |
| Full CRUD view (CargoView) | 515+ | 128 | 20% |

**For a typical entity (model + service + view)**, the developer writes ~160 lines and inherits ~2,400 lines. That is **6% custom code** for 100% CRUD functionality.

## The decision: is it worth it?

### Advantages

| Benefit | Impact |
|---|---|
| Eliminates duplication | 30+ entities x 2,400 lines = 72,000 lines that don't exist |
| UX consistency | All forms follow the same visual pattern |
| Development speed | New full CRUD entity in ~30 minutes |
| Centralized maintenance | Fix in AbstractService benefits all services |
| Tax compliance | Standard fields (taxes, invoices) already in base model |
| Uniform security | ACL and permissions handled consistently |

### Risks and mitigations

| Risk | Mitigation |
|---|---|
| Base class too large | Hooks allow customization without modifying the base |
| Rigid inheritance | Always possible to skip the abstract (like EspelhoView) |
| Learning curve | Once understood, productivity increases exponentially |
| Change in base breaks everything | Unit tests on base methods |

### When to use each approach

```
New CRUD entity?
  ├─ Yes → AbstractModelDoc + AbstractService + AbstractViewDoc
  │        (128 lines for full CRUD)
  │
  └─ Read-only view or custom layout?
      ├─ Yes → AbstractModelDoc + AbstractService + VerticalLayout
      │        (uses inherited service, custom view)
      │
      └─ Entity without DRAPI (e.g., local config)?
          └─ Plain Java class (no inheritance)
```

## Lessons learned

| Challenge | Solution |
|---|---|
| Domino has quirks (UNID, revision, multivalue) | Encapsulated in AbstractModelDoc and AbstractService |
| Each database has its own scope | Automatic resolution via Java model package |
| Forms have very different fields | Customization hooks (configureForm, configureBinder) |
| Some views don't need CRUD | Don't force inheritance — use AbstractService without AbstractViewDoc |
| Deletion rules vary by entity | getPodeDeletar() hook returns list of reasons |
| ACL varies by database | configurePermissions() and CrudActionBar per view |

---

The decision to invest in the three abstract classes cost a few weeks at the beginning of the migration, but enormously accelerated the rest of the project. Each new entity migrated from Domino follows the same pattern: create model, create service, create view — and 95% of the infrastructure is already in place.
