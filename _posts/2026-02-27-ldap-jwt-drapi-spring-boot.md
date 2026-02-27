---
layout: post
title: "Autenticacao com LDAP e JWT do Domino REST API no Spring Boot"
date: 2026-02-27
categories: [domino, drapi, spring-boot, security, ldap, jwt]
---

Uma das maiores vantagens do HCL Domino e seu modelo de seguranca integrado: diretorio LDAP, controle de acesso por ACL e autenticacao centralizada. Quando decidimos construir nossa aplicacao Spring Boot + Vaadin integrada ao Domino via REST API, a primeira questao foi: **como aproveitar essa infraestrutura de seguranca existente em vez de criar uma nova?**

A resposta: autenticacao dupla com **LDAP** (validacao de credenciais) e **JWT** (token de acesso ao DRAPI). Este post descreve como implementamos isso na pratica.

## A arquitetura de autenticacao

O fluxo que construimos usa tres camadas:

```
Usuario digita login/senha
        |
        v
  [1] LDAP Bind ──── Valida credenciais no Domino Directory
        |
        v
  [2] POST /auth ──── Obtem JWT token do Domino REST API
        |
        v
  [3] GET /userinfo ── Carrega perfil e grupos do usuario
        |
        v
  Token armazenado na sessao
        |
        v
  Todas as chamadas ao DRAPI usam: Authorization: Bearer {token}
```

Por que duas etapas? O LDAP valida que o usuario existe e a senha esta correta no diretorio Domino. O DRAPI gera um token JWT com escopo e permissoes especificas para acesso via REST API. Sao complementares.

## Passo 1: Configuracao do LDAP

O Domino Directory funciona como servidor LDAP nativo. Configuramos o Spring Security para autenticar contra ele.

### Propriedades de configuracao

```properties
# application.properties
ldap.urls=ldaps://domino-server-1:636,ldaps://domino-server-2:636
ldap.user-search-base=O=MyOrg
ldap.user-search-filter=(|(cn={0})(uid={0}))
ldap.manager-dn=CN=Admin User,O=MyOrg
ldap.manager-password=${DOMINO_PASSWORD}
```

Pontos importantes:
- Use `ldaps://` (porta 636) para conexao segura. O Domino suporta LDAP sobre SSL nativamente
- Configure **dois servidores** para failover — o `LdapContextSource` aceita multiplas URLs
- O filtro `(|(cn={0})(uid={0}))` permite login tanto pelo nome completo quanto pelo username curto
- A senha do manager vem de variavel de ambiente — nunca no codigo

### Classe de configuracao

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final LdapProperties ldapProperties;
    private final JwtValidationFilter jwtValidationFilter;

    // ... constructor injection

    @Bean
    LdapContextSource contextSource() {
        LdapContextSource cs = new LdapContextSource();
        cs.setUrls(ldapProperties.getUrls().toArray(new String[0]));
        cs.setUserDn(ldapProperties.getManagerDn());
        cs.setPassword(ldapProperties.getManagerPassword());
        cs.setPooled(true);
        cs.afterPropertiesSet();
        return cs;
    }

    @Bean
    LdapTemplate ldapTemplate(LdapContextSource contextSource) {
        return new LdapTemplate(contextSource);
    }

    @Bean
    AuthenticationManager authenticationManager(LdapContextSource contextSource) {
        LdapBindAuthenticationManagerFactory factory =
            new LdapBindAuthenticationManagerFactory(contextSource);
        factory.setUserSearchBase(ldapProperties.getUserSearchBase());
        factory.setUserSearchFilter(ldapProperties.getUserSearchFilter());
        return factory.createAuthenticationManager();
    }

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/images/**", "/VAADIN/**", "/", "/login", "/do-logout").permitAll()
            .anyRequest().authenticated());

        http.sessionManagement(session -> {
            session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED);
            session.invalidSessionUrl("/?expired=true");
        });

        http.csrf(csrf -> csrf.disable()); // Vaadin gerencia CSRF

        // Filtro JWT antes do filtro padrao do Spring Security
        http.addFilterBefore(jwtValidationFilter,
            UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

### Autenticacao via LDAP Bind

O metodo de autenticacao busca o DN do usuario e tenta um LDAP bind com a senha fornecida:

```java
private boolean authenticateLdap(String username, String password) {
    try {
        // 1. Buscar o DN do usuario no diretorio
        String userDn = findUserDn(username);
        if (userDn == null) {
            return false; // Usuario nao encontrado
        }

        // 2. Tentar bind com as credenciais do usuario
        ldapTemplate.getContextSource().getContext(userDn, password);
        return true; // Bind bem-sucedido = senha correta

    } catch (Exception e) {
        log.error("LDAP authentication failed: {}", e.getMessage());
        return false;
    }
}

private String findUserDn(String username) {
    String filter = ldapProperties.getUserSearchFilter().replace("{0}", username);
    String base = ldapProperties.getUserSearchBase();

    List<String> results = ldapTemplate.search(base, filter,
        (AttributesMapper<String>) attrs ->
            attrs.get("cn") != null ? attrs.get("cn").get().toString() : null);

    if (!results.isEmpty() && results.get(0) != null) {
        return "CN=" + results.get(0) + "," + base;
    }
    return null;
}
```

## Passo 2: Obtencao do JWT via DRAPI

Apos o LDAP confirmar as credenciais, chamamos o endpoint `/auth` do Domino REST API para obter um token JWT.

### O servico de autenticacao DRAPI

```java
@Service
public class DominoAuthService {

    private final WebClient webClient;
    private final ObjectMapper objectMapper;

    public User authenticate(String username, String password) {
        User user = new User(username);

        // 1. POST /auth para obter o token
        Map<String, String> credentials = new HashMap<>();
        credentials.put("username", username);
        credentials.put("password", password);

        String tokenResponse = webClient.post()
            .uri("/auth")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(credentials)
            .retrieve()
            .bodyToMono(String.class)
            .block();

        TokenData tokenData = objectMapper.readValue(tokenResponse, TokenData.class);
        user.setTokenData(tokenData);
        user.setToken(tokenData.getBearer());

        // 2. GET /userinfo para obter perfil e grupos
        User userInfo = webClient.get()
            .uri("/userinfo")
            .header("Authorization", "Bearer " + user.getToken())
            .retrieve()
            .bodyToMono(User.class)
            .block();

        // Preencher dados do usuario
        if (userInfo != null) {
            user.setName(userInfo.getName());
            user.setEmail(userInfo.getEmail());
            user.setNames(userInfo.getNames()); // grupos do Domino
        }

        return user;
    }
}
```

### Estrutura do TokenData

A resposta do `POST /auth` retorna um JSON com esta estrutura:

```json
{
  "bearer": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "claims": {
    "iss": "DominoServer",
    "sub": "CN=John Doe,O=MyOrg",
    "iat": 1740668400,
    "exp": 1740669600,
    "aud": ["Domino"],
    "CN": "John Doe",
    "scope": "MAIL $DATA $SETUP",
    "email": "john.doe@company.com"
  },
  "leeway": 0,
  "expSeconds": 1200,
  "issueDate": "2026-02-27T14:00:00Z"
}
```

A classe Java que mapeia essa resposta:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public class TokenData {
    private String bearer;
    private Claims claims;
    private int leeway;
    private int expSeconds;
    private String issueDate;

    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class Claims {
        private String iss;       // Emissor (servidor Domino)
        private String sub;       // Subject (DN do usuario)
        private long iat;         // Emitido em (epoch seconds)
        private long exp;         // Expira em (epoch seconds)
        private List<String> aud; // Audiencia
        private String CN;        // Common Name
        private String scope;     // Escopos/databases autorizados
        private String email;     // Email

        // getters e setters
    }
    // getters e setters
}
```

### Resposta do /userinfo

O endpoint `GET /userinfo` retorna informacoes do usuario autenticado:

```json
{
  "sub": "CN=John Doe,O=MyOrg",
  "name": "John Doe",
  "given_name": "John",
  "family_name": "Doe",
  "preferred_username": "jdoe",
  "email": "john.doe@company.com",
  "scope": "MAIL $DATA",
  "names": ["GroupA", "GroupB", "Admin"]
}
```

O campo `names` e crucial — ele contem os grupos do Domino Directory. Usamos esses grupos para mapear roles no Spring Security:

```java
public List<String> getRoles() {
    if (names == null || names.isEmpty()) {
        return List.of("ROLE_USER");
    }
    return names.stream()
        .map(group -> "ROLE_" + group.trim().toUpperCase())
        .toList();
}
```

Assim, um usuario que pertence ao grupo "Admin" no Domino recebe automaticamente `ROLE_ADMIN` no Spring Security.

## Passo 3: Fluxo completo de login

Juntando as pecas, o fluxo de login na aplicacao Vaadin fica assim:

```java
login.addLoginListener(event -> {
    String username = event.getUsername();
    String password = event.getPassword();

    // 1. Autenticar via LDAP
    boolean ldapOk = authenticateLdap(username, password);
    if (!ldapOk) {
        login.setError(true);
        return;
    }

    // 2. Obter token JWT do DRAPI
    User user = dominoAuthService.authenticate(username, password);
    if (user == null || user.getToken() == null) {
        showErrorMessage("Erro ao conectar com o servidor.");
        return;
    }

    // 3. Verificar se token nao esta expirado
    long exp = user.getTokenData().getClaims().getExp();
    long now = Instant.now().getEpochSecond();
    if (now >= exp) {
        showErrorMessage("Token expirado. Tente novamente.");
        return;
    }

    // 4. Criar Authentication do Spring Security com roles do Domino
    List<SimpleGrantedAuthority> authorities = user.getRoles().stream()
        .map(SimpleGrantedAuthority::new)
        .collect(Collectors.toList());

    Authentication authToken = new UsernamePasswordAuthenticationToken(
        username, null, authorities);

    // 5. Registrar no SecurityContext e na sessao HTTP
    SecurityContext context = SecurityContextHolder.createEmptyContext();
    context.setAuthentication(authToken);

    HttpSession httpSession = getHttpSession();
    httpSession.setAttribute(
        HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY, context);

    // 6. Armazenar User (com token) na sessao Vaadin
    VaadinSession.getCurrent().setAttribute("user", user);
    VaadinSession.getCurrent().setAttribute(User.class, user);

    // 7. Navegar para a pagina principal
    UI.getCurrent().navigate("home");
});
```

## Passo 4: Validacao continua do JWT

O token JWT do DRAPI tem tempo de vida limitado (tipicamente 20 minutos). Precisamos validar se o token ainda e valido a cada requisicao — mas sem sobrecarregar o servidor.

### O filtro de validacao

```java
@Component
@Order(1) // Executar antes dos filtros do Spring Security
public class JwtValidationFilter extends OncePerRequestFilter {

    private final DominoAuthService dominoAuthService;
    private final SessionProperties sessionProperties;
    private static final String LAST_VALIDATION_KEY = "lastJwtValidation";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        String path = request.getRequestURI();

        // Ignorar recursos publicos
        if (path.startsWith("/VAADIN/") || path.startsWith("/images/") ||
            path.equals("/") || path.equals("/login")) {
            filterChain.doFilter(request, response);
            return;
        }

        HttpSession session = request.getSession(false);
        if (session == null) {
            filterChain.doFilter(request, response);
            return;
        }

        User user = (User) session.getAttribute("user");
        if (user == null || user.getToken() == null) {
            filterChain.doFilter(request, response);
            return;
        }

        // Debounce: validar apenas a cada N segundos (ex: 60s)
        Long lastValidation = (Long) session.getAttribute(LAST_VALIDATION_KEY);
        long now = Instant.now().getEpochSecond();

        if (lastValidation != null &&
            (now - lastValidation) < sessionProperties.getJwtValidationIntervalSeconds()) {
            filterChain.doFilter(request, response);
            return;
        }

        session.setAttribute(LAST_VALIDATION_KEY, now);

        // Verificar expiracao do token
        if (isJwtExpired(user)) {
            session.invalidate();
            response.sendRedirect("/?expired=true");
            return;
        }

        filterChain.doFilter(request, response);
    }

    private boolean isJwtExpired(User user) {
        TokenData tokenData = user.getTokenData();
        if (tokenData == null) return true;

        // Verificar campo exp dos claims
        if (tokenData.getClaims() != null && tokenData.getClaims().getExp() > 0) {
            return Instant.now().getEpochSecond() >= tokenData.getClaims().getExp();
        }

        // Fallback: validar diretamente com o servidor
        return !dominoAuthService.isTokenValid(user.getToken());
    }
}
```

A tecnica de **debounce** e essencial: sem ela, cada requisicao HTTP (incluindo carregamento de recursos do Vaadin) dispararia uma validacao. Com o intervalo de 60 segundos, a validacao acontece no maximo uma vez por minuto.

### Validacao com o servidor como fallback

Quando os claims do token nao estao disponiveis, validamos diretamente com o DRAPI:

```java
public boolean isTokenValid(String token) {
    if (token == null || token.isBlank()) return false;
    try {
        webClient.get()
            .uri("/userinfo")
            .header("Authorization", "Bearer " + token)
            .retrieve()
            .bodyToMono(String.class)
            .block();
        return true; // 200 OK = token valido
    } catch (Exception e) {
        return false; // 401 = token expirado
    }
}
```

## Passo 5: Token em todas as chamadas ao DRAPI

Com o token armazenado na sessao Vaadin, todas as classes de servico o utilizam para fazer chamadas autenticadas:

```java
public abstract class AbstractService<T> {

    @Autowired
    protected WebClient webClient;

    protected String getUserToken() {
        User user = (User) UI.getCurrent().getSession().getAttribute("user");
        if (user == null || user.getToken() == null) {
            throw new IllegalStateException("User not authenticated");
        }
        return user.getToken();
    }

    // Exemplo de chamada autenticada
    public Response<T> findByUnid(String unid) {
        String raw = webClient.get()
            .uri("/document/{unid}?dataSource={ds}&mode=default", unid, scope)
            .header("Authorization", "Bearer " + getUserToken())
            .retrieve()
            .bodyToMono(String.class)
            .block();

        T model = objectMapper.readValue(raw, modelClass);
        return new Response<>(model, "OK", 200, true);
    }
}
```

## Passo 6: Cache de token administrativo

Para operacoes que exigem privilegios administrativos (como leitura de ACLs), mantemos um token de admin em cache com renovacao automatica:

```java
@Service
public class AdminTokenService {

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
        // Renovar se faltar menos de 60 segundos
        return expInstant.minusSeconds(60).isBefore(Instant.now());
    }
}
```

## Gestao de timeouts: a sincronia que importa

Um detalhe que levou tempo para acertar: a relacao entre o timeout da sessao HTTP e a expiracao do JWT.

```properties
# Sessao HTTP: 18 minutos
server.servlet.session.timeout=18m

# JWT do DRAPI: 20 minutos
session.jwt-expiration-seconds=1200

# Intervalo de validacao: 60 segundos
session.jwt-validation-interval-seconds=60
```

A regra e: **sessao HTTP deve expirar ANTES do JWT**. Assim:

- Se a sessao HTTP expira (18 min), o Spring Security redireciona para o login de forma limpa
- Se por algum motivo o JWT expira primeiro, o `JwtValidationFilter` detecta e invalida a sessao
- O intervalo de validacao de 60 segundos evita sobrecarga no DRAPI

## Resumo do fluxo completo

```
[Login Form] ──> [LDAP Bind] ──> [POST /auth] ──> [GET /userinfo]
                     |                 |                  |
              Valida senha     Obtem JWT token    Carrega perfil
                                     |               + grupos
                                     v                  |
                              [Sessao HTTP]              |
                              [Sessao Vaadin] <──────────┘
                                     |
                              [Chamadas DRAPI]
                              Authorization: Bearer {token}
                                     |
                              [JwtValidationFilter]
                              Verifica exp a cada 60s
                                     |
                              Se expirado → Login
```

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| LDAP + JWT: por que os dois? | LDAP valida credenciais, DRAPI gera token com escopo |
| Token expira durante o uso | JwtValidationFilter com debounce de 60s |
| Sobrecarga de validacao | Verificar claims.exp localmente antes de chamar /userinfo |
| Sessao vs JWT desincronizados | Sessao HTTP (18 min) < JWT (20 min) |
| Grupos do Domino como roles | Mapear names[] para ROLE_* no Spring Security |
| Operacoes admin sem sessao | Cache de token admin com renovacao automatica |

---

No proximo post, vamos abordar como reproduzimos o modelo de ACL (Access Control List) do Domino no Spring Security, mantendo as mesmas permissoes que os usuarios ja tinham no ambiente legado.
