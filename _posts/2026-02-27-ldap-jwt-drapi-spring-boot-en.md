---
layout: post
title: "Authentication with LDAP and JWT from Domino REST API in Spring Boot"
date: 2026-02-27
categories: [domino, drapi, spring-boot, security, ldap, jwt]
lang: en
---

One of the greatest strengths of HCL Domino is its integrated security model: native LDAP directory, ACL-based access control, and centralized authentication. When we decided to build our Spring Boot + Vaadin application integrated with Domino via REST API, the first question was: **how do we leverage this existing security infrastructure instead of building a new one?**

The answer: dual authentication using **LDAP** (credential validation) and **JWT** (DRAPI access tokens). This post describes how we implemented it in practice.

## The authentication architecture

The flow we built uses three layers:

```
User enters username/password
        |
        v
  [1] LDAP Bind ──── Validate credentials against Domino Directory
        |
        v
  [2] POST /auth ──── Obtain JWT token from Domino REST API
        |
        v
  [3] GET /userinfo ── Load user profile and groups
        |
        v
  Token stored in session
        |
        v
  All DRAPI calls use: Authorization: Bearer {token}
```

Why two steps? LDAP validates that the user exists and the password is correct in the Domino Directory. DRAPI generates a JWT token with specific scope and permissions for REST API access. They are complementary.

## Step 1: LDAP configuration

The Domino Directory works as a native LDAP server. We configured Spring Security to authenticate against it.

### Configuration properties

```properties
# application.properties
ldap.urls=ldaps://domino-server-1:636,ldaps://domino-server-2:636
ldap.user-search-base=O=MyOrg
ldap.user-search-filter=(|(cn={0})(uid={0}))
ldap.manager-dn=CN=Admin User,O=MyOrg
ldap.manager-password=${DOMINO_PASSWORD}
```

Key points:
- Use `ldaps://` (port 636) for secure connections. Domino supports LDAP over SSL natively
- Configure **two servers** for failover — `LdapContextSource` accepts multiple URLs
- The filter `(|(cn={0})(uid={0}))` allows login using either the full name or the short username
- The manager password comes from an environment variable — never hardcoded

### Configuration class

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

        http.csrf(csrf -> csrf.disable()); // Vaadin handles CSRF

        // JWT filter before Spring Security's default filter
        http.addFilterBefore(jwtValidationFilter,
            UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

### LDAP Bind authentication

The authentication method searches for the user's DN and attempts an LDAP bind with the provided password:

```java
private boolean authenticateLdap(String username, String password) {
    try {
        // 1. Find the user's DN in the directory
        String userDn = findUserDn(username);
        if (userDn == null) {
            return false; // User not found
        }

        // 2. Attempt bind with user credentials
        ldapTemplate.getContextSource().getContext(userDn, password);
        return true; // Successful bind = correct password

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

## Step 2: Obtaining the JWT from DRAPI

After LDAP confirms the credentials, we call the Domino REST API's `/auth` endpoint to get a JWT token.

### The DRAPI authentication service

```java
@Service
public class DominoAuthService {

    private final WebClient webClient;
    private final ObjectMapper objectMapper;

    public User authenticate(String username, String password) {
        User user = new User(username);

        // 1. POST /auth to get the token
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

        // 2. GET /userinfo to get profile and groups
        User userInfo = webClient.get()
            .uri("/userinfo")
            .header("Authorization", "Bearer " + user.getToken())
            .retrieve()
            .bodyToMono(User.class)
            .block();

        if (userInfo != null) {
            user.setName(userInfo.getName());
            user.setEmail(userInfo.getEmail());
            user.setNames(userInfo.getNames()); // Domino groups
        }

        return user;
    }
}
```

### TokenData structure

The `POST /auth` response returns a JSON with this structure:

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

The Java class that maps this response:

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
        private String iss;       // Issuer (Domino server)
        private String sub;       // Subject (user's DN)
        private long iat;         // Issued at (epoch seconds)
        private long exp;         // Expires at (epoch seconds)
        private List<String> aud; // Audience
        private String CN;        // Common Name
        private String scope;     // Authorized scopes/databases
        private String email;

        // getters and setters
    }
    // getters and setters
}
```

### The /userinfo response

The `GET /userinfo` endpoint returns the authenticated user's information:

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

The `names` field is crucial — it contains the Domino Directory groups. We use these groups to map roles in Spring Security:

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

This way, a user who belongs to the "Admin" group in Domino automatically receives `ROLE_ADMIN` in Spring Security.

## Step 3: Complete login flow

Putting the pieces together, the login flow in the Vaadin application looks like this:

```java
login.addLoginListener(event -> {
    String username = event.getUsername();
    String password = event.getPassword();

    // 1. Authenticate via LDAP
    boolean ldapOk = authenticateLdap(username, password);
    if (!ldapOk) {
        login.setError(true);
        return;
    }

    // 2. Get JWT token from DRAPI
    User user = dominoAuthService.authenticate(username, password);
    if (user == null || user.getToken() == null) {
        showErrorMessage("Server connection error.");
        return;
    }

    // 3. Verify token is not already expired
    long exp = user.getTokenData().getClaims().getExp();
    long now = Instant.now().getEpochSecond();
    if (now >= exp) {
        showErrorMessage("Token expired. Please try again.");
        return;
    }

    // 4. Create Spring Security Authentication with Domino roles
    List<SimpleGrantedAuthority> authorities = user.getRoles().stream()
        .map(SimpleGrantedAuthority::new)
        .collect(Collectors.toList());

    Authentication authToken = new UsernamePasswordAuthenticationToken(
        username, null, authorities);

    // 5. Register in SecurityContext and HTTP session
    SecurityContext context = SecurityContextHolder.createEmptyContext();
    context.setAuthentication(authToken);

    HttpSession httpSession = getHttpSession();
    httpSession.setAttribute(
        HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY, context);

    // 6. Store User (with token) in Vaadin session
    VaadinSession.getCurrent().setAttribute("user", user);
    VaadinSession.getCurrent().setAttribute(User.class, user);

    // 7. Navigate to home page
    UI.getCurrent().navigate("home");
});
```

## Step 4: Continuous JWT validation

The DRAPI JWT token has a limited lifetime (typically 20 minutes). We need to validate whether the token is still valid on each request — but without overloading the server.

### The validation filter

```java
@Component
@Order(1) // Run before Spring Security filters
public class JwtValidationFilter extends OncePerRequestFilter {

    private final DominoAuthService dominoAuthService;
    private final SessionProperties sessionProperties;
    private static final String LAST_VALIDATION_KEY = "lastJwtValidation";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        String path = request.getRequestURI();

        // Skip public resources
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

        // Debounce: validate only every N seconds (e.g., 60s)
        Long lastValidation = (Long) session.getAttribute(LAST_VALIDATION_KEY);
        long now = Instant.now().getEpochSecond();

        if (lastValidation != null &&
            (now - lastValidation) < sessionProperties.getJwtValidationIntervalSeconds()) {
            filterChain.doFilter(request, response);
            return;
        }

        session.setAttribute(LAST_VALIDATION_KEY, now);

        // Check token expiration
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

        // Check exp field from claims
        if (tokenData.getClaims() != null && tokenData.getClaims().getExp() > 0) {
            return Instant.now().getEpochSecond() >= tokenData.getClaims().getExp();
        }

        // Fallback: validate directly with the server
        return !dominoAuthService.isTokenValid(user.getToken());
    }
}
```

The **debounce** technique is essential: without it, every HTTP request (including Vaadin resource loading) would trigger a validation. With a 60-second interval, validation happens at most once per minute.

### Server-side validation as fallback

When token claims are not available, we validate directly against DRAPI:

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
        return true; // 200 OK = token is valid
    } catch (Exception e) {
        return false; // 401 = token expired
    }
}
```

## Step 5: Token in every DRAPI call

With the token stored in the Vaadin session, all service classes use it to make authenticated calls:

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

    // Example authenticated call
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

## Step 6: Administrative token caching

For operations that require administrative privileges (like reading ACLs), we keep an admin token cached with automatic renewal:

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
        // Renew if less than 60 seconds remaining
        return expInstant.minusSeconds(60).isBefore(Instant.now());
    }
}
```

## Managing timeouts: the synchronization that matters

One detail that took time to get right: the relationship between HTTP session timeout and JWT expiration.

```properties
# HTTP session: 18 minutes
server.servlet.session.timeout=18m

# DRAPI JWT: 20 minutes
session.jwt-expiration-seconds=1200

# Validation interval: 60 seconds
session.jwt-validation-interval-seconds=60
```

The rule is: **HTTP session must expire BEFORE the JWT**. This way:

- If the HTTP session expires (18 min), Spring Security redirects to login cleanly
- If for any reason the JWT expires first, the `JwtValidationFilter` detects it and invalidates the session
- The 60-second validation interval prevents overloading DRAPI

## Complete flow summary

```
[Login Form] ──> [LDAP Bind] ──> [POST /auth] ──> [GET /userinfo]
                     |                 |                  |
              Validate password  Get JWT token    Load profile
                                     |              + groups
                                     v                  |
                              [HTTP Session]             |
                              [Vaadin Session] <─────────┘
                                     |
                              [DRAPI calls]
                              Authorization: Bearer {token}
                                     |
                              [JwtValidationFilter]
                              Check exp every 60s
                                     |
                              If expired → Login
```

## Lessons learned

| Challenge | Solution |
|---|---|
| LDAP + JWT: why both? | LDAP validates credentials, DRAPI generates scoped token |
| Token expires during use | JwtValidationFilter with 60s debounce |
| Validation overhead | Check claims.exp locally before calling /userinfo |
| Session vs JWT out of sync | HTTP session (18 min) < JWT (20 min) |
| Domino groups as roles | Map names[] to ROLE_* in Spring Security |
| Admin operations without session | Cached admin token with automatic renewal |

---

In the next post, we will cover how we reproduced the Domino ACL (Access Control List) model in Spring Security, maintaining the same permissions users already had in the legacy environment.
