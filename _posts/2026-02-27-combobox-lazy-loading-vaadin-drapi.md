---
layout: post
title: "ComboBox no Vaadin com lazy loading do DRAPI: estatico, dinamico e campos dependentes"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, combobox]
---

Formularios no Vaadin frequentemente precisam de ComboBoxes que carregam dados de outras entidades. No nosso caso, os dados vem do Domino via DRAPI. Este post mostra tres padroes de ComboBox que usamos: estatico (itens fixos), dinamico (lazy loading do DRAPI) e dependente (um combo filtra o proximo).

## Padrao 1: ComboBox estatico

Para listas pequenas e fixas, definimos os itens diretamente:

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

### ComboBox reutilizavel: PaisComboBox

Para listas estaticas que aparecem em varios formularios, criamos um componente reutilizavel:

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

Uso:
```java
PaisComboBox paisComboBox = new PaisComboBox("Pais");
form.add(paisComboBox);
```

## Padrao 2: ComboBox com lazy loading do DRAPI

Para entidades com muitos registros, usamos o callback `setItems()` que busca sob demanda conforme o usuario digita:

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

### Como funciona por baixo

1. Usuario abre o ComboBox → callback dispara com `offset=0, limit=50, filter=""`
2. Service chama `GET /lists/_intraCodigos?dataSource=empresas&count=50&start=0`
3. Retorna 50 primeiros codigos
4. Usuario digita "ACM" → callback dispara com `filter="ACM"`
5. Service chama `GET /lists/_intraCodigos?...&startsWith=ACM`
6. Retorna apenas codigos que comecam com "ACM"

## Padrao 2b: ComboBox carregado de uma vez (dataset medio)

Para datasets medios (~100-500 itens), carregamos tudo e deixamos o Vaadin filtrar client-side:

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

## Padrao 3: ComboBoxes dependentes (cascata)

O caso mais interessante: selecionar um Grupo Economico filtra as Empresas disponiveis, e selecionar uma Empresa preenche os campos de endereco.

### Cascata de 3 niveis: Grupo → Empresa → Endereco

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

### Nivel 2: Empresas filtradas pelo Grupo

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

### Nivel 3: Empresa preenche endereco

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

## Visibilidade condicional de campos

Um ComboBox pode controlar a visibilidade de outros campos:

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

## Resumo dos padroes

| Padrao | Quando usar | Exemplo |
|---|---|---|
| Estatico | Itens fixos, < 30 opcoes | UFs, regimes tributarios |
| Reutilizavel | Mesma lista em varios forms | PaisComboBox |
| Lazy loading | > 500 registros, DRAPI | Grupos economicos |
| Carregado | 30-500 registros, rapido | Verticais, cargos |
| Dependente | Selecao filtra proximo combo | Grupo → Empresa → Endereco |
| Condicional | Selecao mostra/esconde campos | Pais → CNPJ visivel/oculto |

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| ComboBox com 5000+ itens trava a UI | Lazy loading com callback setItems(query -> ...) |
| Selecionar grupo precisa filtrar empresas | ValueChangeListener limpa e reconfigura o combo dependente |
| Campos dependentes ficam sujos ao trocar selecao | clear() nos campos dependentes antes de reconfigurar |
| Mesmo combo reutilizado em 5 formularios | Classe PaisComboBox estende ComboBox<String> |
| CNPJ nao faz sentido para empresa estrangeira | setVisible(false) baseado no valor do combo Pais |

---

Com esses tres padroes, qualquer formulario pode ter ComboBoxes eficientes — desde listas fixas de 5 itens ate lookups lazy de milhares de registros com cascata de 3 niveis.
