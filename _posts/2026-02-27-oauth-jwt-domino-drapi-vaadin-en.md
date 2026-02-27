---
layout: post
title: "OAuth2/JWT Authentication with Domino REST API in Spring Boot + Vaadin"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, security]
lang: en
---

When you connect a Spring Boot/Vaadin application to Domino via DRAPI, authentication needs to work across three layers: LDAP to validate credentials, DRAPI to obtain a JWT token, and Spring Security to control access to routes. This post shows how we implemented this complete flow.

## The authentication flow

```
Usuario digita credenciais no LoginView
  │
  ▼  1. Valida no LDAP do Domino
LDAP bind: CN=usuario,O=Org
  │
  ▼  2. Troca credenciais por JWT
POST /auth → TokenData { bearer, claims { exp, sub, iss } }
  │
  ▼  3. Busca info do usuario
GET /userinfo → User { name, email, names[] (grupos Domino) }
  │
  ▼  4. Converte grupos Domino → Spring Security roles
names: ["Everyone", "Managers"] → ROLE_EVERYONE, ROLE_MANAGERS
  │
  ▼  5. Armazena no SecurityContext + VaadinSession
UsernamePasswordAuthenticationToken + httpSession
  │
  ▼  6. Navega para view protegida
@RolesAllowed("ROLE_MANAGERS") → acesso permitido
  │
  ▼  7. A cada 60s, valida se JWT expirou
JwtValidationFilter → se expirou, invalida sessao
```

## The LoginView: entry point

The LoginView uses Vaadin's `LoginForm` component with dual authentication -- LDAP + DRAPI:

```java
@Route(value = "", autoLayout = false)
public class LoginView extends VerticalLayout {

    private final DominoAuthService dominoAuthService;
    private final LdapTemplate ldapTemplate;
    private final LdapProperties ldapProperties;

    public LoginView(DominoAuthService dominoAuthService,
            LdapTemplate ldapTemplate, LdapProperties ldapProperties) {
        this.dominoAuthService = dominoAuthService;
        this.ldapTemplate = ldapTemplate;
        this.ldapProperties = ldapProperties;

        LoginForm login = new LoginForm();
        login.addLoginListener(e -> authenticate(
            e.getUsername(), e.getPassword()));
        add(login);
    }
```

### The authentication method

```java
    private void authenticate(String username, String password) {
        // 1. Valida credenciais via LDAP
        boolean ldapOk = authenticateLdap(username, password);
        if (!ldapOk) {
            login.setError(true);
            return;
        }

        // 2. Troca credenciais por JWT no DRAPI
        User user = dominoAuthService.authenticate(username, password);
        if (user == null || user.getToken() == null) {
            login.setError(true);
            return;
        }

        // 3. Converte grupos Domino em roles Spring Security
        List<SimpleGrantedAuthority> authorities = user.getRoles()
            .stream()
            .map(SimpleGrantedAuthority::new)
            .collect(Collectors.toList());

        // 4. Cria token de autenticacao Spring Security
        Authentication authToken =
            new UsernamePasswordAuthenticationToken(
                username, null, authorities);

        SecurityContext context =
            SecurityContextHolder.createEmptyContext();
        context.setAuthentication(authToken);

        // 5. Persiste na sessao HTTP
        HttpSession httpSession = request.getSession();
        httpSession.setAttribute(
            HttpSessionSecurityContextRepository
                .SPRING_SECURITY_CONTEXT_KEY, context);

        // 6. Armazena User no VaadinSession (acesso rapido)
        VaadinSession.getCurrent().setAttribute("user", user);
        VaadinSession.getCurrent().setAttribute(User.class, user);

        // 7. Navega para a view principal
        UI.getCurrent().navigate("home");
    }
```

### LDAP validation

```java
    private boolean authenticateLdap(String username, String password) {
        try {
            String userDn = findUserDn(username);
            if (userDn == null) return false;

            // Tenta bind com as credenciais do usuario
            ldapTemplate.getContextSource()
                .getContext(userDn, password);
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    private String findUserDn(String username) {
        String filter = ldapProperties.getUserSearchFilter()
            .replace("{0}", username);
        // Filter: (|(cn={0})(uid={0}))

        List<String> results = ldapTemplate.search(
            ldapProperties.getUserSearchBase(),
            filter,
            (AttributesMapper<String>) attrs ->
                attrs.get("cn") != null
                    ? attrs.get("cn").get().toString() : null);

        if (!results.isEmpty() && results.get(0) != null) {
            return "CN=" + results.get(0) + ","
                + ldapProperties.getUserSearchBase();
        }
        return null;
    }
```

## DominoAuthService: exchanging credentials for JWT

The service makes two calls to DRAPI: `/auth` to obtain the token and `/userinfo` for user data:

```java
@Service
public class DominoAuthService {

    private final WebClient webClient;
    private final ObjectMapper objectMapper;

    public User authenticate(String username, String password) {
        User user = new User(username);

        // POST /auth com credenciais
        Map<String, String> credentials = Map.of(
            "username", username,
            "password", password);

        String tokenResponse = webClient.post()
            .uri("/auth")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(credentials)
            .retrieve()
            .bodyToMono(String.class)
            .block();

        // Parse do TokenData (contém JWT + claims)
        TokenData tokenData = objectMapper.readValue(
            tokenResponse, TokenData.class);
        user.setTokenData(tokenData);
        user.setToken(tokenData.getBearer());

        // Log de expiracao para diagnostico
        if (tokenData.getClaims() != null) {
            long exp = tokenData.getClaims().getExp();
            long now = Instant.now().getEpochSecond();
            long ttl = exp - now;
            log.info("Token expira em {} segundos ({} min)",
                ttl, ttl / 60);
        }

        // GET /userinfo com o token obtido
        User userInfo = webClient.get()
            .uri("/userinfo")
            .header("Authorization",
                "Bearer " + user.getToken())
            .retrieve()
            .bodyToMono(User.class)
            .block();

        // Popula dados do usuario
        if (userInfo != null) {
            user.setName(userInfo.getName());
            user.setEmail(userInfo.getEmail());
            user.setNames(userInfo.getNames()); // grupos Domino
        }

        return user;
    }

    // Valida se o token ainda esta ativo
    public boolean isTokenValid(String token) {
        try {
            webClient.get()
                .uri("/userinfo")
                .header("Authorization", "Bearer " + token)
                .retrieve()
                .bodyToMono(String.class)
                .block();
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

## TokenData: the JWT payload

DRAPI returns an object with the bearer token and the decoded claims:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public class TokenData {

    private String bearer;           // O JWT em si
    private Claims claims;           // Claims decodificados

    @JsonProperty("expSeconds")
    private int expSeconds;          // Duracao em segundos

    @JsonProperty("issueDate")
    private String issueDate;        // Data de emissao

    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class Claims {
        private String iss;          // Emissor (servidor Domino)
        private String sub;          // Subject (usuario)
        private long iat;            // Issued At (timestamp)
        private long exp;            // Expiration (timestamp)
        private List<String> aud;    // Audience
        private String CN;           // Common Name
        private String scope;        // Scope OAuth2
        private String email;        // Email
    }
}
```

## User: from Domino groups to Spring Security roles

The `names` field from DRAPI contains the user's Domino groups. The `getRoles()` method converts them to Spring Security format:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public class User {

    private String username;
    private String name;
    private String email;
    private List<String> names = new ArrayList<>();  // Grupos Domino
    private TokenData tokenData;
    private String token;

    // Converte grupos Domino → ROLE_*
    public List<String> getRoles() {
        if (names == null || names.isEmpty()) {
            return List.of("ROLE_USER");
        }
        return names.stream()
            .map(group -> "ROLE_" + group.trim().toUpperCase())
            .toList();
    }

    // Exemplo:
    // names: ["Everyone", "Managers", "Sales"]
    // getRoles(): ["ROLE_EVERYONE", "ROLE_MANAGERS", "ROLE_SALES"]
}
```

## JwtValidationFilter: periodic validation (not on every request)

The filter checks JWT expiration every 60 seconds -- not on every HTTP request. This avoids overloading DRAPI:

```java
@Component
@Order(1)
public class JwtValidationFilter extends OncePerRequestFilter {

    private static final String LAST_VALIDATION_KEY
        = "lastJwtValidation";

    private final DominoAuthService dominoAuthService;
    private final SessionProperties sessionProperties;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        // Pula recursos estaticos e paginas publicas
        String path = request.getRequestURI();
        if (path.startsWith("/VAADIN/") || path.equals("/")
                || path.equals("/login")) {
            chain.doFilter(request, response);
            return;
        }

        HttpSession session = request.getSession(false);
        if (session == null) {
            chain.doFilter(request, response);
            return;
        }

        User user = (User) session.getAttribute("user");
        if (user == null || user.getToken() == null) {
            chain.doFilter(request, response);
            return;
        }

        // Verifica se ja passou o intervalo de validacao
        Long lastValidation = (Long) session.getAttribute(
            LAST_VALIDATION_KEY);
        long now = Instant.now().getEpochSecond();

        if (lastValidation != null && (now - lastValidation)
                < sessionProperties
                    .getJwtValidationIntervalSeconds()) {
            // Ainda nao e hora de validar
            chain.doFilter(request, response);
            return;
        }

        // Atualiza timestamp
        session.setAttribute(LAST_VALIDATION_KEY, now);

        // Verifica se JWT expirou
        if (isJwtExpired(user)) {
            session.invalidate();
            response.sendRedirect("/?expired=true");
            return;
        }

        chain.doFilter(request, response);
    }

    private boolean isJwtExpired(User user) {
        TokenData tokenData = user.getTokenData();
        if (tokenData == null) return true;

        // Metodo 1: verifica claim "exp" (timestamp Unix)
        if (tokenData.getClaims() != null
                && tokenData.getClaims().getExp() > 0) {
            long exp = tokenData.getClaims().getExp();
            long now = Instant.now().getEpochSecond();
            return now >= exp;
        }

        // Metodo 2: fallback — valida no servidor
        return !dominoAuthService.isTokenValid(user.getToken());
    }
}
```

## Configuration: Spring Security + Vaadin

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(jsr250Enabled = true)
public class SecurityConfig {

    private final JwtValidationFilter jwtValidationFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http)
            throws Exception {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/VAADIN/**", "/images/**").permitAll()
            .requestMatchers("/", "/login").permitAll()
            .anyRequest().authenticated());

        http.sessionManagement(session -> {
            session.sessionCreationPolicy(
                SessionCreationPolicy.IF_REQUIRED);
            session.invalidSessionUrl("/?expired=true");
        });

        // CSRF desabilitado (Vaadin tem protecao propria)
        http.csrf(csrf -> csrf.disable());

        // Filtro JWT ANTES do filtro de autenticacao padrao
        http.addFilterBefore(jwtValidationFilter,
            UsernamePasswordAuthenticationToken.class);

        return http.build();
    }
}
```

## Access control in Vaadin Views

The `VaadinSecurityConfig` intercepts all navigation and checks the `@RolesAllowed` annotations:

```java
@Component
public class VaadinSecurityConfig
        implements VaadinServiceInitListener {

    @Override
    public void serviceInit(ServiceInitEvent event) {
        event.getSource().addUIInitListener(uiEvent -> {
            uiEvent.getUI().addBeforeEnterListener(
                new RolesAllowedAccessControl());
        });
    }

    private static class RolesAllowedAccessControl
            implements BeforeEnterListener {

        @Override
        public void beforeEnter(BeforeEnterEvent event) {
            Class<?> target = event.getNavigationTarget();

            // @AnonymousAllowed → acesso livre
            if (target.isAnnotationPresent(
                    AnonymousAllowed.class)) {
                return;
            }

            Authentication auth = SecurityContextHolder
                .getContext().getAuthentication();
            boolean authenticated = auth != null
                && auth.isAuthenticated()
                && !(auth instanceof
                    AnonymousAuthenticationToken);

            // @RolesAllowed → verifica roles
            RolesAllowed roles = target.getAnnotation(
                RolesAllowed.class);
            if (roles != null) {
                if (!authenticated) {
                    event.forwardTo(LoginView.class);
                    return;
                }
                List<String> required =
                    Arrays.asList(roles.value());
                List<String> userRoles =
                    auth.getAuthorities().stream()
                        .map(GrantedAuthority::getAuthority)
                        .toList();

                boolean hasRole = required.stream()
                    .anyMatch(userRoles::contains);
                if (!hasRole) {
                    event.forwardTo("access-denied");
                }
                return;
            }

            // Default: requer autenticacao
            if (!authenticated) {
                event.forwardTo(LoginView.class);
            }
        }
    }
}
```

## Session and JWT configuration

```properties
# Sessao HTTP: 18 minutos (menor que o JWT de 20 min)
server.servlet.session.timeout=18m
session.timeout-seconds=1080

# JWT expira em 20 minutos
session.jwt-expiration-seconds=1200

# Validacao a cada 60 segundos (nao a cada request)
session.jwt-validation-interval-seconds=60
```

```java
@Component
@ConfigurationProperties(prefix = "session")
public class SessionProperties {
    private int timeoutSeconds = 1080;
    private int jwtExpirationSeconds = 1200;
    private int jwtValidationIntervalSeconds = 60;
}
```

**Important detail**: the HTTP session timeout (18 min) must be **shorter** than the JWT expiration (20 min). This ensures the session expires before the token, avoiding requests with an invalid token.

## The complete flow

```
1. LoginView → authenticateLdap() → LDAP bind OK
                                      │
2. LoginView → DominoAuthService.authenticate()
                 │
                 ├─ POST /auth → TokenData { bearer: "eyJ...", claims: { exp: 1234567890 } }
                 └─ GET /userinfo → User { names: ["Everyone", "Managers"] }
                                      │
3. LoginView → User.getRoles() → ["ROLE_EVERYONE", "ROLE_MANAGERS"]
                                      │
4. LoginView → SecurityContext + VaadinSession.setAttribute("user", user)
                                      │
5. UI.navigate("home") → VaadinSecurityConfig.beforeEnter()
   │                       ├─ @RolesAllowed("ROLE_EVERYONE") → OK ✓
   │                       └─ @RolesAllowed("ROLE_FINANCE") → 403 ✗
   │
6. A cada request HTTP:
   └─ JwtValidationFilter
       ├─ (now - lastValidation) < 60s → pula validacao
       └─ (now - lastValidation) >= 60s
           ├─ claims.exp > now → OK, continua
           └─ claims.exp <= now → session.invalidate(), redirect /login
```

## Lessons learned

| Challenge | Solution |
|---|---|
| Validating JWT on every request overloads DRAPI | Interval-based validation (60s) with timestamp in the session |
| HTTP session and JWT have different timeouts | Session (18 min) < JWT (20 min) to avoid inconsistency |
| Domino groups don't follow the Spring Security convention | Automatic conversion: group -> ROLE_{GROUP} |
| Vaadin doesn't use Spring Security MVC natively | VaadinServiceInitListener + BeforeEnterListener |
| Expired token should not block static resources | Filter skips /VAADIN/**, /images/**, /login |
| LDAP and DRAPI can fail independently | Dual authentication: LDAP validates, DRAPI provides the token |

---

With this pipeline, Domino authentication is transparent to the rest of the application. Views use `@RolesAllowed` just like any Spring Security app, and Domino groups are automatically converted into roles.
