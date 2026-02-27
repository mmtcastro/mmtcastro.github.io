---
layout: post
title: "Relacionamentos pai-filho entre bancos Domino: cross-database lookups e validacao de exclusao via DRAPI"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, relationships]
---

No Domino, documentos podem ter relacionamentos entre bancos de dados diferentes — uma Empresa no banco `empresas` pode ter Contatos no mesmo banco e Negocios no banco `sales`. O DRAPI nao tem foreign keys nativas, entao criamos um mecanismo com views indexadas e validacao automatica de exclusao. Este post mostra como implementamos esses relacionamentos.

## O mecanismo: IdOrigem + view _intraIdsOrigem

Cada documento filho armazena o ID do documento pai no campo `IdOrigem`. Uma view Domino chamada `_intraIdsOrigem` indexa esses relacionamentos:

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

- **IdOrigem**: ID do documento pai
- **Id**: ID do documento filho
- **CampoOrigem**: tipo do relacionamento (para filtrar quando um pai tem filhos de tipos diferentes)

## Busca de filhos: findChildrenIdsByIdOrigem()

O metodo no AbstractService busca filhos via a view, com suporte a cross-database:

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

### Metodos auxiliares

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

## Validacao de exclusao: getPodeDeletar()

O AbstractService define um hook que subclasses sobrescrevem para definir regras de exclusao:

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

### Exemplo concreto: EmpresaService

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

### Exemplo: GrupoEconomicoService (multiplos relacionamentos)

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

## Cross-database lookup: buscando em outro banco

O parametro `targetScope` permite buscar relacionamentos em bancos diferentes:

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

## O fluxo de exclusao na UI

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

## O fluxo completo

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

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Domino nao tem foreign keys | View _intraIdsOrigem indexada por IdOrigem |
| Filhos podem estar em bancos diferentes | Parametro targetScope para cross-database lookup |
| Exclusao sem verificar filhos causa orfaos | getPodeDeletar() verifica automaticamente antes de excluir |
| Um pai pode ter filhos de tipos diferentes | CampoOrigem filtra por tipo de relacionamento |
| Regras de exclusao variam por entidade | Hook sobrescrito em cada service concreto |
| Mensagem de erro precisa ser informativa | Lista de motivos concatenados com separador |

---

Esse mecanismo transforma o Domino — que nativamente nao tem foreign keys — em um sistema com integridade referencial validada. O desenvolvedor so precisa sobrescrever `getPodeDeletar()` para definir regras, e o framework cuida do resto.
