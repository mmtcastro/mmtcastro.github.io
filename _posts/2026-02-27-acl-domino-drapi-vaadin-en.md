---
layout: post
title: "Domino ACL in Vaadin: controlling page access and button visibility via DRAPI"
date: 2026-02-27
categories: [domino, drapi, vaadin, spring-security, acl]
lang: en
---

One of HCL Domino's greatest strengths is its granular security model: each database has its own ACL (Access Control List) with access levels, roles, and flags. When migrating to Spring Boot + Vaadin, we needed to replicate this model so the application respects the same security rules that already exist in Domino.

This post shows how we fetch the ACL via Domino REST API, resolve the user's effective access, and apply these permissions both for route control (who can access a page) and for UI element visibility (who sees the edit or delete button).

## Domino access levels

Domino defines 7 access levels, from most restrictive to most permissive:

```java
public enum DominoAccessLevel {
    NOACCESS,      // No access
    DEPOSITOR,     // Can deposit documents (cannot read)
    READER,        // Can read
    AUTHOR,        // Can read and edit own documents
    EDITOR,        // Can read and edit any document
    DESIGNER,      // Can modify design (views, forms)
    MANAGER;       // Full access

    public boolean atLeast(DominoAccessLevel other) {
        return this.ordinal() >= other.ordinal();
    }
}
```

Beyond levels, each ACL entry can have special **flags**:

```java
public enum DominoAclFlag {
    NODELETE,           // Cannot delete documents
    AUTHOR_NOCREATE,    // Author without create permission
    PUBLICREADER,       // Public reader access
    PUBLICWRITER        // Public writer access
}
```

And each entry can have **roles** — the well-known Domino `@ROLES` that control access to views, forms, and sections.

## Fetching the ACL via DRAPI

DRAPI exposes an administrative endpoint to query ACL entries from any database:

```
GET /api/admin-v1/acl/entries?dataSource={scope}
Authorization: Bearer {admin-token}
```

The response is an array of entries:

```json
[
  {
    "name": "John Doe",
    "type": "PERSON",
    "level": "MANAGER",
    "roles": ["Admin", "Finance"],
    "flags": []
  },
  {
    "name": "Management",
    "type": "GROUP",
    "level": "EDITOR",
    "roles": ["Finance"],
    "flags": ["NODELETE"]
  },
  {
    "name": "Sales",
    "type": "GROUP",
    "level": "AUTHOR",
    "roles": ["Sales"],
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

**Important**: this endpoint requires an admin token. It is not the logged-in user's token — it is a service token with ACL read permission.

## The ACL data model

Each ACL entry maps to a POJO:

```java
@Getter @Setter
@JsonIgnoreProperties(ignoreUnknown = true)
public class DominoAclEntry {
    private String name;                 // "John Doe", "Management", "Everyone"
    private DominoAccessLevel level;     // EDITOR, MANAGER, etc.
    private String type;                 // "PERSON", "GROUP", "SERVER" or empty
    private List<String> roles = new ArrayList<>();  // Domino roles
    private Set<String> flags = new HashSet<>();     // NODELETE, AUTHOR_NOCREATE, etc.

    public boolean isPerson() { return "PERSON".equalsIgnoreCase(type); }
    public boolean isGroup()  { return "GROUP".equalsIgnoreCase(type); }
}
```

## Admin token with caching

To access the ACL endpoint, we maintain a service token with caching and automatic renewal:

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

The token is renewed 60 seconds before expiration, preventing failures during calls.

## The ACL service

The service fetches ACL entries and resolves the user's effective access:

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
            log.error("Error loading ACL from {}: {}", dataSource, e.getMessage());
            return List.of();
        }
    }
}
```

## Resolving effective access

The most important part: given a user and the list of ACL entries, what is their effective access? Domino follows specific rules:

1. **PERSON entries have absolute priority** over groups
2. If no PERSON entry matches, uses the **group with the highest level**
3. If no group matches, uses the **Everyone** entry
4. **Roles are cumulative** — the user inherits roles from all matching entries
5. **Flags come from the best entry** (not cumulative)

```java
public DominoEffectiveAccess resolveEffectiveAccess(
        List<DominoAclEntry> aclEntries, User user) {

    // 1. Build user keys (name, username, groups)
    List<String> userKeys = buildUserKeys(user);

    // 2. Filter matching entries
    List<DominoAclEntry> matchingEntries = aclEntries.stream()
        .filter(entry -> userKeys.contains(normalize(entry.getName())))
        .toList();

    // 3. Fallback to Everyone
    if (matchingEntries.isEmpty()) {
        matchingEntries = aclEntries.stream()
            .filter(e -> "EVERYONE".equalsIgnoreCase(normalize(e.getName())))
            .toList();
    }

    // 4. Find best level (PERSON first)
    DominoAclEntry bestEntry = null;
    DominoAccessLevel bestLevel = DominoAccessLevel.NOACCESS;

    // Priority 1: PERSON entries
    for (DominoAclEntry entry : matchingEntries) {
        if (entry.isPerson() && entry.getLevel().compareTo(bestLevel) > 0) {
            bestLevel = entry.getLevel();
            bestEntry = entry;
        }
    }

    // Priority 2: groups (if no PERSON found)
    if (bestEntry == null) {
        for (DominoAclEntry entry : matchingEntries) {
            if (entry.getLevel().compareTo(bestLevel) > 0) {
                bestLevel = entry.getLevel();
                bestEntry = entry;
            }
        }
    }

    // 5. Roles: accumulate from ALL matching entries
    Set<String> effectiveRoles = matchingEntries.stream()
        .filter(e -> e.getRoles() != null)
        .flatMap(e -> e.getRoles().stream())
        .collect(Collectors.toSet());

    // 6. Flags: from the best entry only
    Set<DominoAclFlag> effectiveFlags = bestEntry != null
        ? parseFlags(bestEntry.getFlags())
        : EnumSet.noneOf(DominoAclFlag.class);

    return new DominoEffectiveAccess(bestLevel, effectiveFlags, effectiveRoles);
}
```

### Building user keys

A user can match ACL entries through different identifiers:

```java
private List<String> buildUserKeys(User user) {
    List<String> keys = new ArrayList<>();

    // Short name (login)
    if (user.getUsername() != null) keys.add(normalize(user.getUsername()));

    // Preferred username (DRAPI)
    if (user.getPreferredUsername() != null)
        keys.add(normalize(user.getPreferredUsername()));

    // Full name
    if (user.getFullName() != null) keys.add(normalize(user.getFullName()));

    // Domino groups (returned by /userinfo)
    if (user.getNames() != null) {
        user.getNames().forEach(name -> keys.add(normalize(name)));
    }

    // Universal fallback
    keys.add("EVERYONE");

    return keys;
}
```

## Effective access: UI helper methods

The `DominoEffectiveAccess` class encapsulates the result and provides ready-to-use methods for the interface:

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

## Applying to routes: @RolesAllowed

In Vaadin, we use `@RolesAllowed` to control who can access each page. Domino groups are converted to Spring Security roles:

```java
// In the User model, Domino groups become ROLE_*
public List<String> getRoles() {
    if (names == null || names.isEmpty()) {
        return List.of("ROLE_USER");
    }
    return names.stream()
            .map(group -> "ROLE_" + group.trim().toUpperCase())
            .toList();
}
```

Example: a user belonging to groups `["Management", "Finance"]` gets roles `["ROLE_MANAGEMENT", "ROLE_FINANCE"]`.

In the Vaadin view:

```java
// Any authenticated user can access
@Route(value = "companies", layout = MainLayout.class)
@RolesAllowed("ROLE_EVERYONE")
public class CompaniesView extends VerticalLayout { ... }

// Only management and finance
@Route(value = "receipt/:unid?", layout = MainLayout.class)
@RolesAllowed({"ROLE_MANAGEMENT", "ROLE_FINANCE"})
public class ReceiptView extends AbstractViewDoc<Receipt> { ... }
```

### The BeforeEnterListener that makes it work

A listener intercepts all navigation and validates roles:

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

            // @AnonymousAllowed → permit
            if (target.isAnnotationPresent(AnonymousAllowed.class)) return;

            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            boolean authenticated = auth != null && auth.isAuthenticated()
                    && !(auth instanceof AnonymousAuthenticationToken);

            // @PermitAll → just need to be logged in
            if (target.isAnnotationPresent(PermitAll.class)) {
                if (!authenticated) event.forwardTo(LoginView.class);
                return;
            }

            // @RolesAllowed → check roles
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

## Applying to the UI: button visibility

To control edit, delete, and create buttons based on the Domino access level, we created a `CrudActionBar` component:

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
        // Edit: visible if read-only, not new, AND has permission
        editButton.setVisible(isReadOnly && !isNew && canEdit);

        // Save: visible in edit mode, with permission check
        boolean canSave = isNew ? canCreate : canEdit;
        saveButton.setVisible(!isReadOnly && canSave);

        // Cancel: always visible in edit mode
        cancelButton.setVisible(!isReadOnly);

        // Delete: visible if not new, read-only, AND has permission
        deleteButton.setVisible(!isNew && isReadOnly && canDelete);
    }
}
```

### Concrete example: receipt view

```java
@Route(value = "receipt/:unid?", layout = MainLayout.class)
@RolesAllowed({"ROLE_MANAGEMENT", "ROLE_FINANCE"})
public class ReceiptView extends AbstractViewDoc<Receipt> {

    private final ReceiptService service;

    @Override
    protected void configurePermissions() {
        User user = getCurrentUser();
        if (user == null) return;

        // Load ACL from the financial database
        List<DominoAclEntry> aclEntries = service.loadAcl("financial_db");
        DominoEffectiveAccess access = service.resolveEffectiveAccess(aclEntries, user);

        // Apply permissions to action bar
        actionBar.setPermissions(access);

        // Read-only fields for readers
        if (access.isReadOnly()) {
            numberField.setReadOnly(true);
            dateField.setReadOnly(true);
            valueField.setReadOnly(true);
        }

        // Logic based on Domino role
        if (access.hasRole("Finance")) {
            approvalSection.setVisible(true);
        }
    }
}
```

## The complete flow

```
User login
    │
    ├── LDAP validates credentials
    ├── POST /auth → JWT token
    └── GET /userinfo → Domino groups ["Management", "Finance", ...]
            │
            └── Converted to Spring Security: ROLE_MANAGEMENT, ROLE_FINANCE
                    │
                    ├── ROUTE CONTROL (@RolesAllowed)
                    │   └── BeforeEnterListener checks roles
                    │       ├── Has role? → access the page
                    │       └── No role? → redirect to access-denied
                    │
                    └── UI CONTROL (DominoEffectiveAccess)
                        │
                        ├── GET /api/admin-v1/acl/entries?dataSource={scope}
                        │   └── Returns ACL entries for the database
                        │
                        ├── resolveEffectiveAccess(aclEntries, user)
                        │   ├── PERSON > GROUP > Everyone
                        │   ├── Cumulative roles
                        │   └── Flags from best entry
                        │
                        └── DominoEffectiveAccess
                            ├── canRead()       → can see the page
                            ├── canCreate()     → sees New button
                            ├── canEditOthers() → sees Edit button
                            ├── canDelete()     → sees Delete button
                            ├── isReadOnly()    → fields disabled
                            └── hasRole("X")    → conditional sections
```

## Two security levels, one model

| Level | Mechanism | Source | Controls |
|---|---|---|---|
| **Route** | `@RolesAllowed` | Domino groups → Spring Security | Who can access the page |
| **UI** | `DominoEffectiveAccess` | Database ACL via DRAPI | What the user sees and can do |

The first level (route) uses **Domino groups** converted to Spring Security roles. It is fast and automatic — just annotate the view.

The second level (UI) uses the **actual database ACL** via DRAPI. It is more granular — it differentiates Editor from Reader, respects flags like NODELETE, and enables logic based on specific Domino roles.

## Lessons learned

| Challenge | Solution |
|---|---|
| ACL endpoint requires admin token | AclAdminTokenService with caching and renewal |
| User can match by name, group, or CN | buildUserKeys() tests all variants |
| PERSON must take priority over GROUP | Two-pass search (PERSON first) |
| Roles must be cumulative | Accumulate from all matching entries |
| Flags are not cumulative | Use flags from the best entry only |
| AUTHOR may not have create rights | AUTHOR_NOCREATE flag prevents canCreate() |
| EDITOR may not have delete rights | NODELETE flag prevents canDelete() |

---

With this architecture, the Vaadin application respects the same security model that has been working in Domino for years. Administrators continue managing permissions in the Domino ACL — the Java application simply queries and obeys.
