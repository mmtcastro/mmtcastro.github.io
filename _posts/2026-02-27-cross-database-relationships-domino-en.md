---
layout: post
title: "Parent-child relationships across Domino databases: cross-database lookups and deletion validation via DRAPI"
date: 2026-02-27
lang: en
categories: [domino, drapi, java, vaadin, relationships]
---

In Domino, documents can have relationships across different databases — a Company in the `empresas` database can have Contacts in the same database and Deals in the `sales` database. DRAPI does not have native foreign keys, so we created a mechanism with indexed views and automatic deletion validation. This post shows how we implemented these relationships.

## The mechanism: IdOrigem + _intraIdsOrigem view

Each child document stores the parent document's ID in the `IdOrigem` field. A Domino view called `_intraIdsOrigem` indexes these relationships:

```
View _intraIdsOrigem (indexada por IdOrigem):
┌─────────────────────────────────────┬──────────────────────────────────────┬─────────────┐
│ IdOrigem                            │ Id                                   │ CampoOrigem │
├─────────────────────────────────────┼──────────────────────────────────────┼─────────────┤
│ empresas_Empresa_abc123             │ empresas_Contato_def456              │ contatos    │
│ empresas_Empresa_abc123             │ empresas_Contato_ghi789              │ contatos    │
│ empresas_GrupoEconomico_xyz999     │ empresas_Empresa_abc123              │ empresas    │
└─────────────────────────────────────┴──────────────────────────────────────┴─────────────┘
```

- **IdOrigem**: parent document ID
- **Id**: child document ID
- **CampoOrigem**: relationship type (to filter when a parent has children of different types)

## Finding children: findChildrenIdsByIdOrigem()

The method in AbstractService searches for children via the view, with cross-database support:

```java
public List<String> findChildrenIdsByIdOrigem(
        String idOrigem, String campoOrigem,
        String targetScope) {

    List<String> childrenIds = new ArrayList<>();

    if (idOrigem == null || idOrigem.isBlank()) {
        return childrenIds;
    }

    if (targetScope == null || targetScope.isBlank()) {
        targetScope = this.scope;  // mesmo banco
    }

    // Consulta a view _intraIdsOrigem
    String uri = String.format(
        "/lists/_intraIdsOrigem?dataSource=%s"
        + "&key=%s&documents=false",
        targetScope, idOrigem);

    String rawResponse = webClient.get()
        .uri(uri)
        .header("Authorization",
            "Bearer " + getUserToken())
        .retrieve()
        .bodyToMono(String.class)
        .block();

    if (rawResponse == null || rawResponse.isBlank()) {
        return childrenIds;
    }

    JsonNode root = objectMapper.readTree(rawResponse);
    if (root.isArray()) {
        for (JsonNode node : root) {
            // Filtra por CampoOrigem se especificado
            String nodeCampo = getFieldValue(
                node, "CampoOrigem");
            if (campoOrigem != null
                    && !campoOrigem.equalsIgnoreCase(
                        nodeCampo)) {
                continue;
            }

            String childId = getFieldValue(node, "Id");
            if (childId != null && !childId.isBlank()) {
                childrenIds.add(childId);
            }
        }
    }

    return childrenIds;
}
```

### Helper methods

```java
// Verifica existencia de filhos
public boolean hasChildren(String idOrigem,
        String campoOrigem) {
    return !findChildrenIdsByIdOrigem(
        idOrigem, campoOrigem).isEmpty();
}

// Conta filhos
public int countChildren(String idOrigem,
        String campoOrigem) {
    return findChildrenIdsByIdOrigem(
        idOrigem, campoOrigem).size();
}

// Cross-database: busca filhos em outro banco
public boolean hasChildren(String idOrigem,
        String campoOrigem, String targetScope) {
    return !findChildrenIdsByIdOrigem(
        idOrigem, campoOrigem, targetScope).isEmpty();
}
```

## Deletion validation: getPodeDeletar()

The AbstractService defines a hook that subclasses override to define deletion rules:

```java
// AbstractService (implementacao padrao: permite exclusao)
public List<String> getPodeDeletar(T model) {
    return null;  // null = pode deletar
}

// Metodo delete() verifica automaticamente
public DeleteResponse delete(T model) {
    // Verifica se pode deletar
    List<String> motivos = getPodeDeletar(model);
    if (motivos != null && !motivos.isEmpty()) {
        DeleteResponse response = new DeleteResponse();
        response.setStatus("403");
        response.setMessage(
            String.join(" | ", motivos));
        return response;
    }

    // Exclusao real
    // DELETE /document/{unid}?dataSource={scope}
    ...
}
```

### Concrete example: EmpresaService

```java
@Override
public List<String> getPodeDeletar(Empresa empresa) {
    List<String> motivos = new ArrayList<>();

    // Regra 1: verifica Contatos (mesmo banco)
    if (hasContatos(empresa.getId())) {
        int count = countContatosByEmpresa(
            empresa.getId());
        motivos.add(String.format(
            "Possui %d Contato(s). "
            + "Apague os contatos primeiro.", count));
    }

    // Regra 2: verifica Negocios (banco sales)
    if (!possoDeletarEmpresaPorNegocios(
            empresa.getCodigo())) {
        motivos.add(
            "Possui Negocios registrados. "
            + "Desative a empresa em vez de apagar.");
    }

    return motivos.isEmpty() ? null : motivos;
}
```

### Example: GrupoEconomicoService (multiple relationships)

```java
@Override
public List<String> getPodeDeletar(
        GrupoEconomico grupo) {
    List<String> motivos = new ArrayList<>();

    // Regra 1: verifica Empresas filhas
    if (hasChildren(grupo.getId(), null)) {
        int count = countChildren(grupo.getId(), null);
        motivos.add(String.format(
            "Possui %d Empresa(s) associada(s). "
            + "Apague as empresas primeiro.", count));
    }

    // Regra 2: verifica Negocios (cross-database)
    if (grupo.getCodigo() != null) {
        var negocios = negocioService
            .findByCodigoGrupoEconomico(
                grupo.getCodigo());
        if (negocios != null && !negocios.isEmpty()) {
            motivos.add(
                "Possui Negocios registrados. "
                + "Desative o grupo em vez de apagar.");
        }
    }

    return motivos.isEmpty() ? null : motivos;
}
```

## Cross-database lookup: searching in another database

The `targetScope` parameter allows searching for relationships in different databases:

```java
// Busca contatos da empresa (mesmo banco: empresas)
List<String> contatoIds = findChildrenIdsByIdOrigem(
    empresa.getId(), "contatos");  // scope default

// Busca negocios da empresa (banco: sales)
List<String> negocioIds = findChildrenIdsByIdOrigem(
    empresa.getId(), null, "sales");

// Busca documento por ID em outro banco
Response<Deal> deal = dealService.findById(
    dealId, "sales");
```

## The deletion flow in the UI

```java
// Na AbstractViewDoc
protected void delete() {
    ConfirmDialog dialog = new ConfirmDialog(
        "Confirmar exclusao",
        "Deseja excluir este registro?",
        "Excluir",
        e -> {
            DeleteResponse response =
                getService().delete(model);

            if ("403".equals(response.getStatus())) {
                // Mostra motivos da recusa
                Notification.show(
                    response.getMessage(),
                    5000,
                    Notification.Position.MIDDLE);
            } else {
                // Sucesso: volta para a lista
                Notification.show("Excluido com sucesso");
                getUI().ifPresent(ui ->
                    ui.navigate(getBackView()));
            }
        },
        "Cancelar", e -> {});

    dialog.open();
}
```

## The complete flow

```
Hierarquia:
  GrupoEconomico "ACME"
    └─ Empresa "ACME Matriz"
        ├─ Contato "Joao Silva"
        └─ Contato "Maria Santos"
    └─ Empresa "ACME Filial SP"

Cenario: tentar excluir "ACME Matriz"
  │
  ▼  View chama getService().delete(empresa)
  │
  ▼  delete() chama getPodeDeletar(empresa)
  │
  ├─ hasContatos(empresa.getId())?
  │   └─ GET /lists/_intraIdsOrigem?dataSource=empresas
  │         &key=empresas_Empresa_abc123
  │   └─ Retorna 2 contatos → SIM
  │   └─ motivos.add("Possui 2 Contato(s)...")
  │
  ├─ possoDeletarEmpresaPorNegocios("ACME-001")?
  │   └─ Consulta banco sales → SIM, tem negocios
  │   └─ motivos.add("Possui Negocios registrados...")
  │
  ▼  Retorna DeleteResponse(403, motivos)
  │
  ▼  UI mostra Notification:
     "Possui 2 Contato(s). Apague os contatos primeiro.
      | Possui Negocios registrados. Desative a empresa."
```

## Lessons learned

| Challenge | Solution |
|---|---|
| Domino has no foreign keys | _intraIdsOrigem view indexed by IdOrigem |
| Children can be in different databases | targetScope parameter for cross-database lookup |
| Deleting without checking children causes orphans | getPodeDeletar() automatically validates before deletion |
| A parent can have children of different types | CampoOrigem filters by relationship type |
| Deletion rules vary by entity | Hook overridden in each concrete service |
| Error message needs to be informative | List of reasons concatenated with separator |

---

This mechanism transforms Domino — which natively has no foreign keys — into a system with validated referential integrity. The developer only needs to override `getPodeDeletar()` to define rules, and the framework takes care of the rest.
