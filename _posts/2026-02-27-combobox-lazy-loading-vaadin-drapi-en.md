---
layout: post
title: "ComboBox in Vaadin with DRAPI lazy loading: static, dynamic and dependent fields"
date: 2026-02-27
lang: en
categories: [domino, drapi, java, vaadin, combobox]
---

Forms in Vaadin frequently need ComboBoxes that load data from other entities. In our case, the data comes from Domino via DRAPI. This post shows three ComboBox patterns we use: static (fixed items), dynamic (lazy loading from DRAPI) and dependent (one combo filters the next).

## Pattern 1: Static ComboBox

For small, fixed lists, we define the items directly:

```java
// Lista fixa de opcoes
indicadorIeComboBox.setItems(
    "Contribuinte", "Nao Contribuinte", "Isento");
indicadorIeComboBox.setPlaceholder(
    "Selecione o Indicador");

// Estados brasileiros
ufComboBox.setItems("AC", "AL", "AP", "AM", "BA", "CE",
    "DF", "ES", "GO", "MA", "MT", "MS", "MG", "PA",
    "PB", "PR", "PE", "PI", "RJ", "RN", "RS", "RO",
    "RR", "SC", "SP", "SE", "TO");

// Regime tributario
regimeTributarioComboBox.setItems(
    "Normal", "Simples Nacional",
    "Microempreendedor Individual");
```

### Reusable ComboBox: PaisComboBox

For static lists that appear in multiple forms, we created a reusable component:

```java
public class PaisComboBox extends ComboBox<String> {

    public PaisComboBox(String label) {
        super(label);
        setItems(Utils.getPaises()); // Lista de paises
        setValue("Brasil");          // Default
        setPlaceholder("Selecione um Pais");
        setWidthFull();
    }
}
```

Usage:
```java
PaisComboBox paisComboBox = new PaisComboBox("Pais");
form.add(paisComboBox);
```

## Pattern 2: ComboBox with DRAPI lazy loading

For entities with many records, we use the `setItems()` callback that fetches on demand as the user types:

```java
private void configureComboBoxes() {
    List<QuerySortOrder> sortOrders = Collections
        .singletonList(new QuerySortOrder(
            "Codigo", SortDirection.ASCENDING));

    // Lazy loading: busca no DRAPI conforme o usuario digita
    grupoEconomicoComboBox.setItems(
        query -> grupoEconomicoService
            .findAllByCodigo(
                query.getOffset(),
                query.getLimit(),
                sortOrders,
                query.getFilter().orElse(""),
                GrupoEconomico.class, false)
            .stream()
            .map(GrupoEconomico::getCodigo));

    grupoEconomicoComboBox.setPlaceholder(
        "Selecione um Grupo Economico");
}
```

### How it works under the hood

1. User opens the ComboBox -> callback fires with `offset=0, limit=50, filter=""`
2. Service calls `GET /lists/_intraCodigos?dataSource=empresas&count=50&start=0`
3. Returns the first 50 codes
4. User types "ACM" -> callback fires with `filter="ACM"`
5. Service calls `GET /lists/_intraCodigos?...&startsWith=ACM`
6. Returns only codes that start with "ACM"

## Pattern 2b: Fully loaded ComboBox (medium dataset)

For medium datasets (~100-500 items), we load everything and let Vaadin filter client-side:

```java
// Carrega todas as verticais do servico
List<Vertical> verticais = verticalService.getVerticais();
if (verticais != null) {
    verticalComboBox.setItems(
        verticais.stream()
            .map(Vertical::getCodigo)
            .toList());
}
verticalComboBox.setPlaceholder("Selecione uma Vertical");
```

## Pattern 3: Dependent ComboBoxes (cascade)

The most interesting case: selecting an Economic Group filters the available Companies, and selecting a Company fills in the address fields.

### 3-level cascade: Group -> Company -> Address

```java
// Nivel 1: Grupo Economico (lazy loading)
grupoEconomicoComboBox.setItems(
    query -> grupoEconomicoService
        .findAllByCodigo(...)
        .stream()
        .map(GrupoEconomico::getCodigo));

grupoEconomicoComboBox.addValueChangeListener(event -> {
    String codigoGrupo = event.getValue();

    // Limpa selecoes dependentes
    empresaComboBox.clear();
    codigoField.clear();

    if (codigoGrupo == null || codigoGrupo.isBlank()) {
        empresaComboBox.setVisible(false);
        return;
    }

    // Busca dados do grupo selecionado
    Response<GrupoEconomico> response =
        grupoEconomicoService.findByCodigo(codigoGrupo);
    if (!response.isSuccess()) {
        Notification.show("Grupo nao encontrado");
        return;
    }

    grupoSelecionado = response.getModel();

    // Atualiza modelo
    model.setCodigoGrupoEconomico(
        grupoSelecionado.getCodigo());
    model.setIdGrupoEconomico(
        grupoSelecionado.getId());

    // Configura combo de Empresas (filtrado pelo grupo)
    configureEmpresaComboBoxForGrupo(codigoGrupo);
    empresaComboBox.setVisible(true);
});
```

### Level 2: Companies filtered by Group

```java
private void configureEmpresaComboBoxForGrupo(
        String codigoGrupo) {

    // Busca empresas do grupo selecionado
    List<Empresa> empresasDoGrupo =
        grupoEconomicoService
            .findEmpresasByCodigoGrupoEconomico(
                codigoGrupo);

    if (empresasDoGrupo == null
            || empresasDoGrupo.isEmpty()) {
        empresaComboBox.setItems(List.of());
        empresaComboBox.setPlaceholder(
            "Nenhuma empresa neste grupo");
    } else {
        List<String> codigos = empresasDoGrupo.stream()
            .map(Empresa::getCodigo)
            .filter(c -> c != null && !c.isBlank())
            .sorted()
            .toList();
        empresaComboBox.setItems(codigos);
        empresaComboBox.setPlaceholder(
            "Selecione (" + codigos.size()
            + " disponiveis)");
    }
}
```

### Level 3: Company fills in address

```java
empresaComboBox.addValueChangeListener(event -> {
    String codigoEmpresa = event.getValue();

    if (codigoEmpresa == null
            || codigoEmpresa.isBlank()) {
        return;
    }

    // Busca empresa selecionada
    Response<Empresa> response =
        empresaService.findByCodigo(codigoEmpresa);
    if (!response.isSuccess()) return;

    empresaSelecionada = response.getModel();

    // Atualiza modelo
    model.setCodigoOrigem(
        empresaSelecionada.getCodigo());
    model.setIdOrigem(
        empresaSelecionada.getId());

    // Preenche campos de endereco
    fillAddressFromEmpresa(empresaSelecionada);
});

private void fillAddressFromEmpresa(Empresa empresa) {
    logradouro.setValue(
        empresa.getEndereco() != null
            ? empresa.getEndereco() : "");
    numero.setValue(
        empresa.getNumero() != null
            ? empresa.getNumero() : "");
    cidade.setValue(
        empresa.getCidade() != null
            ? empresa.getCidade() : "");
    estado.setValue(
        empresa.getEstado() != null
            ? empresa.getEstado() : "");
    pais.setValue(
        empresa.getPais() != null
            ? empresa.getPais() : "Brasil");
    cep.setValue(
        empresa.getCep() != null
            ? empresa.getCep() : "");
}
```

## Conditional field visibility

A ComboBox can control the visibility of other fields:

```java
// Esconde CNPJ se pais nao e Brasil
paisComboBox.addValueChangeListener(event -> {
    String pais = event.getValue();
    boolean isBrasil = pais == null
        || "Brasil".equalsIgnoreCase(pais.trim());

    cgcField.setVisible(isBrasil);
    if (!isBrasil) {
        cgcField.clear();
    }
});

// Mostra campos especificos de "Cliente"
tipoComboBox.addValueChangeListener(event -> {
    String tipo = event.getValue();
    boolean isCliente = "Cliente".equalsIgnoreCase(tipo);
    origemClienteComboBox.setVisible(isCliente);
    gerenteContaComboBox.setVisible(isCliente);
});
```

## Pattern summary

| Pattern | When to use | Example |
|---|---|---|
| Static | Fixed items, < 30 options | Brazilian states, tax regimes |
| Reusable | Same list in multiple forms | PaisComboBox |
| Lazy loading | > 500 records, DRAPI | Economic groups |
| Fully loaded | 30-500 records, fast | Verticals, job titles |
| Dependent | Selection filters next combo | Group -> Company -> Address |
| Conditional | Selection shows/hides fields | Country -> CNPJ visible/hidden |

## Lessons learned

| Challenge | Solution |
|---|---|
| ComboBox with 5000+ items freezes the UI | Lazy loading with setItems(query -> ...) callback |
| Selecting a group needs to filter companies | ValueChangeListener clears and reconfigures the dependent combo |
| Dependent fields get dirty when changing selection | clear() on dependent fields before reconfiguring |
| Same combo reused in 5 forms | PaisComboBox class extends ComboBox<String> |
| CNPJ makes no sense for a foreign company | setVisible(false) based on the Country combo value |

---

With these three patterns, any form can have efficient ComboBoxes — from fixed lists of 5 items to lazy lookups of thousands of records with 3-level cascading.
