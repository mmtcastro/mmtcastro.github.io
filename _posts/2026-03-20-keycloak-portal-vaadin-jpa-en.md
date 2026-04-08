---
layout: post
title: "User Portal: Integrating Keycloak PostgreSQL with Vaadin via Read-Only JPA"
date: 2026-03-20
categories: [domino, drapi, java, spring, vaadin, keycloak, postgresql, jpa]
lang: en
---

In our HCL Domino migration, authentication moved to **Keycloak** hosted on a dedicated PostgreSQL. We needed a portal to view and manage realm users — but without depending on Keycloak's Admin API, which requires privileged credentials and is more complex to configure. The solution: connect directly to Keycloak's PostgreSQL in **read-only** mode via Spring Data JPA.

## Why access the database directly?

Keycloak's Admin REST API is powerful, but:
- Requires a service account with realm-admin permissions
- Adds an extra HTTP layer (Keycloak → API → our app)
- For simple queries (list users, view attributes), it's overkill

Since our use case is purely **read-only** (viewing who's in the realm), connecting directly to PostgreSQL is simpler, faster, and doesn't require Keycloak admin credentials.

## Datasource configuration

Spring Boot 4 configures the main datasource via `application.properties`:

```properties
# PostgreSQL - Keycloak Database (READ-ONLY)
spring.datasource.url=${KEYCLOAK_DB_URL:jdbc:postgresql://192.168.168.5:5432/keycloak?options=-c%20TimeZone%3DAmerica/Sao_Paulo}
spring.datasource.username=keycloak
spring.datasource.password=${KEYCLOAK_DB_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.hibernate.ddl-auto=none
```

The URL is configurable via the `KEYCLOAK_DB_URL` environment variable, with a fallback to the production server. This allows using a local PostgreSQL in dev without code changes.

The `TimeZone=America/Sao_Paulo` parameter in the URL is critical — without it, PostgreSQL rejects the connection if the JVM timezone isn't recognized (we discovered this when migrating to Java 25 on a machine with Buenos Aires timezone).

## Resolving the Realm UUID

Keycloak stores everything by `realm_id` (UUID), not by the friendly name. `PortalService` resolves this at startup:

```java
@Service
public class PortalService {

    @Value("${keycloak.properties.realm-name:tdec-portal}")
    private String realmName;

    private String realmId;

    @PostConstruct
    public void init() {
        try {
            realmId = jdbcTemplate.queryForObject(
                "SELECT id FROM realm WHERE name = ?",
                String.class, realmName);
            logger.info("Keycloak realm '{}' resolved to id={}", realmName, realmId);
        } catch (Exception e) {
            logger.error("Failed to resolve realm '{}': {}", realmName, e.getMessage());
        }
    }
}
```

If PostgreSQL isn't reachable (common in dev), the service degrades gracefully — the portal shows an error message instead of crashing the application.

## Listing realm users

With `realmId` resolved, the query is straightforward:

```java
public List<Map<String, Object>> findUsers() {
    return jdbcTemplate.queryForList(
        "SELECT id, username, email, first_name, last_name, enabled " +
        "FROM user_entity WHERE realm_id = ? ORDER BY username",
        realmId);
}
```

## PortalView: Vaadin grid with users

The view displays users in a standard Grid:

```java
@Route(value = "portal", layout = MainLayout.class)
@RolesAllowed("ROLE_EVERYONE")
public class PortalView extends VerticalLayout {
    // Grid with columns: Username, Email, First Name, Last Name, Active
    // Loaded via portalService.findUsers()
}
```

## Dev vs Production

`KEYCLOAK_DB_URL` allows pointing to different PostgreSQL instances:

- **Dev**: local PostgreSQL or none (app starts without portal)
- **Production**: `jdbc:postgresql://192.168.168.5:5432/keycloak` (real Keycloak server)

`KEYCLOAK_DB_PASSWORD` lives in `.env` (dev) or container secrets (production via podman).

## Result

With ~50 lines of code, we have a functional portal showing Keycloak users without depending on the Admin API. Read-only PostgreSQL access is secure (limited credentials), fast (direct query), and degrades gracefully when the database is unavailable.
