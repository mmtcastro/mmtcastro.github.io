---
layout: post
title: "ACL do Domino no Vaadin: controlando acesso a paginas e botoes via DRAPI"
date: 2026-02-27
categories: [domino, drapi, vaadin, spring-security, acl]
---

Um dos maiores diferenciais do HCL Domino e seu modelo de seguranca granular: cada banco de dados tem sua propria ACL (Access Control List) com niveis de acesso, roles e flags. Quando migramos para Spring Boot + Vaadin, precisamos reproduzir esse modelo para que a aplicacao respeite as mesmas regras de seguranca que ja existem no Domino.

Este post mostra como buscamos a ACL via Domino REST API, resolvemos o acesso efetivo do usuario, e aplicamos essas permissoes tanto no controle de rotas (quem pode acessar uma pagina) quanto na visibilidade de elementos da interface (quem ve o botao de editar ou excluir).

## Os niveis de acesso do Domino

O Domino define 7 niveis de acesso, do mais restrito ao mais permissivo:

```java
public enum DominoAccessLevel {
    NOACCESS,      // Sem acesso
    DEPOSITOR,     // Pode depositar documentos (sem ler)
    READER,        // Pode ler
    AUTHOR,        // Pode ler e editar documentos proprios
    EDITOR,        // Pode ler e editar qualquer documento
    DESIGNER,      // Pode modificar design (views, forms)
    MANAGER;       // Acesso total

    public boolean atLeast(DominoAccessLevel other) {
        return this.ordinal() >= other.ordinal();
    }
}
```

Alem dos niveis, cada entrada na ACL pode ter **flags** especiais:

```java
public enum DominoAclFlag {
    NODELETE,           // Nao pode excluir documentos
    AUTHOR_NOCREATE,    // Author sem permissao de criar
    PUBLICREADER,       // Acesso publico de leitura
    PUBLICWRITER        // Acesso publico de escrita
}
```

E cada entrada pode ter **roles** — os famosos `@ROLES` do Domino que controlam acesso a views, forms e secoes.

## Buscando a ACL via DRAPI

O DRAPI expoe um endpoint administrativo para consultar as entradas da ACL de qualquer banco de dados:

```
GET /api/admin-v1/acl/entries?dataSource={scope}
Authorization: Bearer {admin-token}
```

A resposta e um array de entradas:

```json
[
  {
    "name": "John Doe",
    "type": "PERSON",
    "level": "MANAGER",
    "roles": ["Admin", "Financeiro"],
    "flags": []
  },
  {
    "name": "Management",
    "type": "GROUP",
    "level": "EDITOR",
    "roles": ["Financeiro"],
    "flags": ["NODELETE"]
  },
  {
    "name": "Sales",
    "type": "GROUP",
    "level": "AUTHOR",
    "roles": ["Vendas"],
    "flags": ["AUTHOR_NOCREATE"]
  },
  {
    "name": "Everyone",
    "type": "",
    "level": "READER",
    "roles": [],
    "flags": []
  }
]
```

**Importante**: este endpoint requer um token de administrador. Nao e o token do usuario logado — e um token de servico com permissao de leitura da ACL.

## O modelo de dados da ACL

Cada entrada da ACL e mapeada para um POJO:

```java
@Getter @Setter
@JsonIgnoreProperties(ignoreUnknown = true)
public class DominoAclEntry {
    private String name;                 // "John Doe", "Management", "Everyone"
    private DominoAccessLevel level;     // EDITOR, MANAGER, etc.
    private String type;                 // "PERSON", "GROUP", "SERVER" ou vazio
    private List<String> roles = new ArrayList<>();  // Roles Domino
    private Set<String> flags = new HashSet<>();     // NODELETE, AUTHOR_NOCREATE, etc.

    public boolean isPerson() { return "PERSON".equalsIgnoreCase(type); }
    public boolean isGroup()  { return "GROUP".equalsIgnoreCase(type); }
}
```

## Token administrativo com cache

Para acessar o endpoint de ACL, mantemos um token de servico com cache e renovacao automatica:

```java
@Service
public class AclAdminTokenService {
    private final DominoAuthService dominoAuthService;
    private TokenData cachedToken;

    public synchronized String getAdminBearerToken() {
        if (cachedToken == null || isExpiringSoon(cachedToken)) {
            var adminUser = dominoAuthService.authenticate(
                    adminUsername, adminPassword);
            cachedToken = adminUser != null ? adminUser.getTokenData() : null;
        }
        return cachedToken != null ? cachedToken.getBearer() : null;
    }

    private boolean isExpiringSoon(TokenData token) {
        if (token == null || token.getClaims() == null) return true;
        long expEpoch = token.getClaims().getExp();
        Instant expInstant = Instant.ofEpochSecond(expEpoch);
        return expInstant.minusSeconds(60).isBefore(Instant.now());
    }
}
```

O token e renovado 60 segundos antes de expirar, evitando falhas durante chamadas.

## O servico de ACL

O servico busca as entradas da ACL e resolve o acesso efetivo do usuario:

```java
@Service
public class DominoAclService {

    private final WebClient webClient;
    private final AclAdminTokenService aclAdminTokenService;

    public List<DominoAclEntry> loadAcl(String dataSource) {
        String bearer = aclAdminTokenService.getAdminBearerToken();
        String uri = "/api/admin-v1/acl/entries?dataSource=" + dataSource;

        try {
            String body = webClient.get()
                    .uri(uri)
                    .header("Authorization", "Bearer " + bearer)
                    .header("Accept", "application/json")
                    .retrieve()
                    .bodyToMono(String.class)
                    .block();

            DominoAclEntry[] entries = objectMapper.readValue(body, DominoAclEntry[].class);
            return Arrays.asList(entries);
        } catch (Exception e) {
            log.error("Erro ao carregar ACL de {}: {}", dataSource, e.getMessage());
            return List.of();
        }
    }
}
```

## Resolvendo o acesso efetivo

A parte mais importante: dado um usuario e a lista de entradas da ACL, qual e o acesso efetivo dele? O Domino segue regras especificas:

1. **Entradas PERSON tem prioridade absoluta** sobre grupos
2. Se nao houver entrada PERSON, usa o **grupo com nivel mais alto**
3. Se nao casar com nenhum grupo, usa a entrada **Everyone**
4. **Roles sao cumulativas** — o usuario herda roles de todas as entradas que casam
5. **Flags vem da melhor entrada** (nao sao cumulativas)

```java
public DominoEffectiveAccess resolveEffectiveAccess(
        List<DominoAclEntry> aclEntries, User user) {

    // 1. Construir chaves do usuario (nome, username, grupos)
    List<String> userKeys = buildUserKeys(user);

    // 2. Filtrar entradas que casam com o usuario
    List<DominoAclEntry> matchingEntries = aclEntries.stream()
        .filter(entry -> userKeys.contains(normalize(entry.getName())))
        .toList();

    // 3. Fallback para Everyone
    if (matchingEntries.isEmpty()) {
        matchingEntries = aclEntries.stream()
            .filter(e -> "EVERYONE".equalsIgnoreCase(normalize(e.getName())))
            .toList();
    }

    // 4. Buscar melhor nivel (PERSON primeiro)
    DominoAclEntry bestEntry = null;
    DominoAccessLevel bestLevel = DominoAccessLevel.NOACCESS;

    // Prioridade 1: entradas PERSON
    for (DominoAclEntry entry : matchingEntries) {
        if (entry.isPerson() && entry.getLevel().compareTo(bestLevel) > 0) {
            bestLevel = entry.getLevel();
            bestEntry = entry;
        }
    }

    // Prioridade 2: grupos (se nao achou PERSON)
    if (bestEntry == null) {
        for (DominoAclEntry entry : matchingEntries) {
            if (entry.getLevel().compareTo(bestLevel) > 0) {
                bestLevel = entry.getLevel();
                bestEntry = entry;
            }
        }
    }

    // 5. Roles: acumular de TODAS as entradas que casam
    Set<String> effectiveRoles = matchingEntries.stream()
        .filter(e -> e.getRoles() != null)
        .flatMap(e -> e.getRoles().stream())
        .collect(Collectors.toSet());

    // 6. Flags: apenas da melhor entrada
    Set<DominoAclFlag> effectiveFlags = bestEntry != null
        ? parseFlags(bestEntry.getFlags())
        : EnumSet.noneOf(DominoAclFlag.class);

    return new DominoEffectiveAccess(bestLevel, effectiveFlags, effectiveRoles);
}
```

### Construindo as chaves do usuario

O usuario pode casar com a ACL por diferentes identificadores:

```java
private List<String> buildUserKeys(User user) {
    List<String> keys = new ArrayList<>();

    // Nome curto (login)
    if (user.getUsername() != null) keys.add(normalize(user.getUsername()));

    // Username preferido (DRAPI)
    if (user.getPreferredUsername() != null)
        keys.add(normalize(user.getPreferredUsername()));

    // Nome completo
    if (user.getFullName() != null) keys.add(normalize(user.getFullName()));

    // Grupos do Domino (retornados pelo /userinfo)
    if (user.getNames() != null) {
        user.getNames().forEach(name -> keys.add(normalize(name)));
    }

    // Fallback universal
    keys.add("EVERYONE");

    return keys;
}
```

## O acesso efetivo: metodos para a UI

A classe `DominoEffectiveAccess` encapsula o resultado e oferece metodos prontos para a interface:

```java
public class DominoEffectiveAccess {
    private final DominoAccessLevel level;
    private final Set<DominoAclFlag> flags;
    private final Set<String> roles;

    public boolean canRead() {
        return level.compareTo(DominoAccessLevel.READER) >= 0
                || flags.contains(DominoAclFlag.PUBLICREADER);
    }

    public boolean canCreate() {
        if (flags.contains(DominoAclFlag.AUTHOR_NOCREATE)) return false;
        if (flags.contains(DominoAclFlag.PUBLICWRITER)) return true;
        return level.compareTo(DominoAccessLevel.AUTHOR) >= 0;
    }

    public boolean canEditOwn() {
        return level.compareTo(DominoAccessLevel.AUTHOR) >= 0;
    }

    public boolean canEditOthers() {
        return level.compareTo(DominoAccessLevel.EDITOR) >= 0;
    }

    public boolean canDelete() {
        return level.compareTo(DominoAccessLevel.EDITOR) >= 0
                && !flags.contains(DominoAclFlag.NODELETE);
    }

    public boolean isReadOnly() {
        if (!canRead()) return true;
        if (level.compareTo(DominoAccessLevel.READER) <= 0) return true;
        return flags.contains(DominoAclFlag.AUTHOR_NOCREATE);
    }

    public boolean hasRole(String role) {
        return roles != null && roles.contains(role);
    }
}
```

## Aplicando na rota: @RolesAllowed

No Vaadin, usamos `@RolesAllowed` para controlar quem pode acessar cada pagina. Os grupos do Domino sao convertidos em roles do Spring Security:

```java
// No modelo User, grupos Domino viram ROLE_*
public List<String> getRoles() {
    if (names == null || names.isEmpty()) {
        return List.of("ROLE_USER");
    }
    return names.stream()
            .map(group -> "ROLE_" + group.trim().toUpperCase())
            .toList();
}
```

Exemplo: um usuario que pertence aos grupos `["Management", "Finance"]` recebe as roles `["ROLE_MANAGEMENT", "ROLE_FINANCE"]`.

Na view Vaadin:

```java
// Qualquer usuario autenticado pode acessar
@Route(value = "companies", layout = MainLayout.class)
@RolesAllowed("ROLE_EVERYONE")
public class CompaniesView extends VerticalLayout { ... }

// Apenas gestao e financeiro
@Route(value = "receipt/:unid?", layout = MainLayout.class)
@RolesAllowed({"ROLE_MANAGEMENT", "ROLE_FINANCE"})
public class ReceiptView extends AbstractViewDoc<Receipt> { ... }
```

### O BeforeEnterListener que faz funcionar

Um listener intercepta toda navegacao e valida as roles:

```java
@Component
public class VaadinSecurityConfig implements VaadinServiceInitListener {

    @Override
    public void serviceInit(ServiceInitEvent event) {
        event.getSource().addUIInitListener(uiEvent ->
            uiEvent.getUI().addBeforeEnterListener(new RolesAccessControl()));
    }

    private static class RolesAccessControl implements BeforeEnterListener {

        @Override
        public void beforeEnter(BeforeEnterEvent event) {
            Class<?> target = event.getNavigationTarget();

            // @AnonymousAllowed → libera
            if (target.isAnnotationPresent(AnonymousAllowed.class)) return;

            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            boolean authenticated = auth != null && auth.isAuthenticated()
                    && !(auth instanceof AnonymousAuthenticationToken);

            // @PermitAll → basta estar logado
            if (target.isAnnotationPresent(PermitAll.class)) {
                if (!authenticated) event.forwardTo(LoginView.class);
                return;
            }

            // @RolesAllowed → verificar roles
            RolesAllowed rolesAllowed = target.getAnnotation(RolesAllowed.class);
            if (rolesAllowed != null) {
                if (!authenticated) {
                    event.forwardTo(LoginView.class);
                    return;
                }

                List<String> required = Arrays.asList(rolesAllowed.value());
                List<String> userRoles = auth.getAuthorities().stream()
                        .map(GrantedAuthority::getAuthority)
                        .toList();

                boolean hasRole = required.stream().anyMatch(userRoles::contains);
                if (!hasRole) {
                    event.forwardTo("access-denied");
                }
            }
        }
    }
}
```

## Aplicando na UI: visibilidade de botoes

Para controlar botoes de edicao, exclusao e criacao com base no nivel de acesso do Domino, criamos um componente `CrudActionBar`:

```java
public class CrudActionBar extends HorizontalLayout {
    private final Button editButton;
    private final Button saveButton;
    private final Button cancelButton;
    private final Button deleteButton;

    private boolean canEdit = true;
    private boolean canDelete = true;
    private boolean canCreate = true;

    public void setPermissions(DominoEffectiveAccess access) {
        if (access == null) return;
        this.canEdit = access.canEditOthers();
        this.canDelete = access.canDelete();
        this.canCreate = access.canCreate();
    }

    public void updateButtonVisibility(boolean isNew, boolean isReadOnly) {
        // Editar: visivel se leitura, nao e novo, e TEM permissao
        editButton.setVisible(isReadOnly && !isNew && canEdit);

        // Salvar: visivel em modo edicao, com verificacao de permissao
        boolean canSave = isNew ? canCreate : canEdit;
        saveButton.setVisible(!isReadOnly && canSave);

        // Cancelar: sempre visivel em modo edicao
        cancelButton.setVisible(!isReadOnly);

        // Excluir: visivel se nao e novo, leitura, e TEM permissao
        deleteButton.setVisible(!isNew && isReadOnly && canDelete);
    }
}
```

### Exemplo concreto: view de recibos

```java
@Route(value = "receipt/:unid?", layout = MainLayout.class)
@RolesAllowed({"ROLE_MANAGEMENT", "ROLE_FINANCE"})
public class ReceiptView extends AbstractViewDoc<Receipt> {

    private final ReceiptService service;

    @Override
    protected void configurePermissions() {
        User user = getCurrentUser();
        if (user == null) return;

        // Carregar ACL do banco financeiro
        List<DominoAclEntry> aclEntries = service.loadAcl("financial_db");
        DominoEffectiveAccess access = service.resolveEffectiveAccess(aclEntries, user);

        // Aplicar permissoes na barra de acoes
        actionBar.setPermissions(access);

        // Campos somente leitura para readers
        if (access.isReadOnly()) {
            numberField.setReadOnly(true);
            dateField.setReadOnly(true);
            valueField.setReadOnly(true);
        }

        // Logica baseada em role Domino
        if (access.hasRole("Financeiro")) {
            approvalSection.setVisible(true);
        }
    }
}
```

## O fluxo completo

```
Login do usuario
    │
    ├── LDAP valida credenciais
    ├── POST /auth → token JWT
    └── GET /userinfo → grupos Domino ["Management", "Finance", ...]
            │
            └── Convertidos para Spring Security: ROLE_MANAGEMENT, ROLE_FINANCE
                    │
                    ├── CONTROLE DE ROTA (@RolesAllowed)
                    │   └── BeforeEnterListener verifica roles
                    │       ├── Tem role? → acessa a pagina
                    │       └── Nao tem? → redireciona para access-denied
                    │
                    └── CONTROLE DE UI (DominoEffectiveAccess)
                        │
                        ├── GET /api/admin-v1/acl/entries?dataSource={scope}
                        │   └── Retorna entradas da ACL do banco
                        │
                        ├── resolveEffectiveAccess(aclEntries, user)
                        │   ├── PERSON > GROUP > Everyone
                        │   ├── Roles acumulativas
                        │   └── Flags da melhor entrada
                        │
                        └── DominoEffectiveAccess
                            ├── canRead()       → pode ver a pagina
                            ├── canCreate()     → ve botao Novo
                            ├── canEditOthers() → ve botao Editar
                            ├── canDelete()     → ve botao Excluir
                            ├── isReadOnly()    → campos desabilitados
                            └── hasRole("X")    → secoes condicionais
```

## Dois niveis de seguranca, um modelo

| Nivel | Mecanismo | Origem | Controla |
|---|---|---|---|
| **Rota** | `@RolesAllowed` | Grupos Domino → Spring Security | Quem pode acessar a pagina |
| **UI** | `DominoEffectiveAccess` | ACL do banco via DRAPI | O que o usuario ve e pode fazer |

O primeiro nivel (rota) usa os **grupos do Domino** convertidos em roles do Spring Security. E rapido e automatico — basta anotar a view.

O segundo nivel (UI) usa a **ACL real do banco de dados** via DRAPI. E mais granular — diferencia Editor de Reader, respeita flags como NODELETE, e permite logica baseada em roles Domino especificas.

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Endpoint ACL requer token admin | AclAdminTokenService com cache e renovacao |
| Usuario pode casar por nome, grupo ou CN | buildUserKeys() testa todas as variantes |
| PERSON deve ter prioridade sobre GROUP | Busca em duas passadas (PERSON primeiro) |
| Roles precisam ser cumulativas | Acumula de todas as entradas que casam |
| Flags nao sao cumulativas | Usa flags apenas da melhor entrada |
| AUTHOR pode nao ter direito de criar | Flag AUTHOR_NOCREATE impede canCreate() |
| EDITOR pode nao ter direito de excluir | Flag NODELETE impede canDelete() |

---

Com essa arquitetura, a aplicacao Vaadin respeita o mesmo modelo de seguranca que ja funciona no Domino ha anos. Administradores continuam gerenciando permissoes na ACL do Domino — a aplicacao Java simplesmente consulta e obedece.
