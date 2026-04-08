---
layout: post
title: "Portal de Usuarios: Integrando Keycloak PostgreSQL com Vaadin via JPA Read-Only"
date: 2026-03-20
categories: [domino, drapi, java, spring, vaadin, keycloak, postgresql, jpa]
---

Na nossa migracao do HCL Domino, a autenticacao migrou para o **Keycloak** hospedado em um PostgreSQL dedicado. Precisavamos de um portal para visualizar e gerenciar usuarios do realm — mas sem depender da Admin API do Keycloak, que exige credenciais privilegiadas e e mais complexa de configurar. A solucao: conectar diretamente no PostgreSQL do Keycloak em modo **read-only** via Spring Data JPA.

## Por que acessar o banco diretamente?

A Admin REST API do Keycloak e poderosa, mas:
- Exige um service account com permissoes de realm-admin
- Adiciona uma camada HTTP extra (Keycloak → API → nossa app)
- Para consultas simples (listar usuarios, ver atributos), e overkill

Como nosso caso e apenas **leitura** (visualizar quem esta no realm), conectar direto no PostgreSQL e mais simples, rapido e nao exige credenciais admin do Keycloak.

## Configuracao do datasource

O Spring Boot 4 configura o datasource principal via `application.properties`:

```properties
# PostgreSQL - Keycloak Database (READ-ONLY)
spring.datasource.url=${KEYCLOAK_DB_URL:jdbc:postgresql://192.168.168.5:5432/keycloak?options=-c%20TimeZone%3DAmerica/Sao_Paulo}
spring.datasource.username=keycloak
spring.datasource.password=${KEYCLOAK_DB_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.hibernate.ddl-auto=none
```

A URL e configuravel via variavel de ambiente `KEYCLOAK_DB_URL`, com fallback para o servidor de producao. Isso permite usar um PostgreSQL local em dev sem alterar o codigo.

O parametro `TimeZone=America/Sao_Paulo` na URL e critico — sem ele, o PostgreSQL rejeita a conexao se o timezone da JVM nao for reconhecido (descobrimos isso ao migrar para Java 25 em uma maquina com timezone de Buenos Aires).

## Resolvendo o Realm UUID

O Keycloak armazena tudo por `realm_id` (UUID), nao pelo nome amigavel. O `PortalService` resolve isso na inicializacao:

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
            logger.info("Keycloak realm '{}' resolvido para id={}", realmName, realmId);
        } catch (Exception e) {
            logger.error("Falha ao resolver realm '{}': {}", realmName, e.getMessage());
        }
    }
}
```

Se o PostgreSQL nao estiver acessivel (comum em dev), o servico degrada gracefully — o portal mostra uma mensagem de erro em vez de derrubar a aplicacao.

## Listando usuarios do realm

Com o `realmId` resolvido, a consulta e direta:

```java
public List<Map<String, Object>> findUsers() {
    return jdbcTemplate.queryForList(
        "SELECT id, username, email, first_name, last_name, enabled " +
        "FROM user_entity WHERE realm_id = ? ORDER BY username",
        realmId);
}
```

## PortalView: grid Vaadin com usuarios

A view exibe os usuarios em um Grid padrao:

```java
@Route(value = "portal", layout = MainLayout.class)
@RolesAllowed("ROLE_EVERYONE")
public class PortalView extends VerticalLayout {
    // Grid com colunas: Username, Email, Nome, Sobrenome, Ativo
    // Carrega via portalService.findUsers()
}
```

## Dev vs Producao

O `KEYCLOAK_DB_URL` permite apontar para diferentes PostgreSQL:

- **Dev**: PostgreSQL local ou nenhum (app inicia sem portal)
- **Producao**: `jdbc:postgresql://192.168.168.5:5432/keycloak` (servidor Keycloak real)

A variavel `KEYCLOAK_DB_PASSWORD` fica no `.env` (dev) ou nos secrets do container (producao via podman).

## Resultado

Com ~50 linhas de codigo, temos um portal funcional que mostra usuarios do Keycloak sem depender da Admin API. O acesso read-only ao PostgreSQL e seguro (credenciais limitadas), rapido (query direta), e degrada bem quando o banco nao esta disponivel.
