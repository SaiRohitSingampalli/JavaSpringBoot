# 18 — Spring Security 6: Authentication, Authorization, JWT, OAuth2

## 1. Core concepts
| Term | Meaning |
|------|---------|
| **Authentication** (AuthN) | *Who are you?* Verify identity (password, token, certificate) |
| **Authorization** (AuthZ) | *What are you allowed to do?* Roles, permissions, ownership |
| **Principal** | The authenticated user/identity |
| **Credentials** | Password / token proving identity |
| **GrantedAuthority** | Permission string (`ROLE_ADMIN`, `orders:write`, `SCOPE_read`) |
| **Role** | An authority with `ROLE_` prefix by convention (`hasRole("ADMIN")` checks `ROLE_ADMIN`) |
| **SecurityContext** | Holds the current `Authentication`; stored in `SecurityContextHolder` (ThreadLocal) |

**401 vs 403**: 401 Unauthorized = not authenticated / bad credentials. 403 Forbidden = authenticated but lacks permission.

## 2. What you get just by adding `spring-boot-starter-security`
- Every endpoint requires authentication.
- Default user `user` with a random password printed in the logs (`spring.security.user.name/password` to set).
- Form login page `/login`, logout `/logout`, HTTP Basic.
- CSRF protection, session fixation protection, security headers (X-Content-Type-Options, X-Frame-Options DENY, HSTS on HTTPS, Cache-Control).
- BCrypt password storage via `DelegatingPasswordEncoder`.

## 3. Architecture — the filter chain ⭐
```
Client request
   │
Servlet container
   │  Filter: DelegatingFilterProxy ("springSecurityFilterChain")
   ▼
FilterChainProxy ── picks the first matching SecurityFilterChain (by request matcher)
   │
   ▼  SecurityFilterChain (ordered filters, simplified)
   ├─ DisableEncodeUrlFilter / ForceEagerSessionCreationFilter
   ├─ SecurityContextHolderFilter        ← loads SecurityContext (from session / nothing for stateless)
   ├─ HeaderWriterFilter                 ← security headers
   ├─ CorsFilter
   ├─ CsrfFilter
   ├─ LogoutFilter
   ├─ OAuth2AuthorizationRequestRedirectFilter / OAuth2LoginAuthenticationFilter (oauth2Login)
   ├─ UsernamePasswordAuthenticationFilter (formLogin)
   ├─ BearerTokenAuthenticationFilter    (oauth2ResourceServer / JWT)
   ├─ BasicAuthenticationFilter          (httpBasic)
   ├─ RequestCacheAwareFilter
   ├─ SecurityContextHolderAwareRequestFilter
   ├─ AnonymousAuthenticationFilter      ← sets anonymous auth if none
   ├─ SessionManagementFilter
   ├─ ExceptionTranslationFilter         ← AuthenticationException → 401 entry point; AccessDeniedException → 403 handler
   └─ AuthorizationFilter                ← checks authorizeHttpRequests rules
   │
   ▼
DispatcherServlet → controller (method security via AOP: @PreAuthorize)
```
Print your chain: `logging.level.org.springframework.security=TRACE` or `@EnableWebSecurity(debug = true)` (dev only).

## 4. Authentication flow (username/password)
```
Filter (e.g., UsernamePasswordAuthenticationFilter / BasicAuthenticationFilter)
   │ creates UsernamePasswordAuthenticationToken(unauthenticated)
   ▼
AuthenticationManager (ProviderManager)
   │ iterates AuthenticationProviders
   ▼
DaoAuthenticationProvider
   ├─ UserDetailsService.loadUserByUsername(username) → UserDetails (username, hashed password, authorities, enabled, locked...)
   └─ PasswordEncoder.matches(raw, hashed)
   ▼
Authenticated Authentication (principal + authorities)
   ▼
SecurityContextHolder.getContext().setAuthentication(auth)  (+ saved in session if stateful)
```
Failure → `AuthenticationException` (BadCredentials, Locked, Disabled...) → 401.

## 5. Modern configuration (Spring Security 6 — lambda DSL)
`WebSecurityConfigurerAdapter` was **removed**; you declare `SecurityFilterChain` beans. `antMatchers/mvcMatchers` → `requestMatchers`. `authorizeRequests` → `authorizeHttpRequests`. `@EnableGlobalMethodSecurity` → `@EnableMethodSecurity`.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity                                  // @PreAuthorize etc. (prePostEnabled = true by default)
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http, JwtAuthFilter jwtFilter) throws Exception {
        return http
            .securityMatcher("/api/**")                // this chain only handles /api/**
            .csrf(AbstractHttpConfigurer::disable)     // stateless token API (no cookies) → CSRF not needed
            .cors(Customizer.withDefaults())           // uses CorsConfigurationSource bean
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.POST, "/api/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.DELETE, "/api/**").hasAuthority("orders:delete")
                .anyRequest().authenticated())         // ORDER MATTERS: most specific first
            .exceptionHandling(e -> e
                .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))
                .accessDeniedHandler((req, res, ex) -> res.sendError(HttpServletResponse.SC_FORBIDDEN)))
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    @Order(Ordered.LOWEST_PRECEDENCE)
    SecurityFilterChain web(HttpSecurity http) throws Exception {   // fallback chain e.g. for actuator / docs
        return http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/actuator/health/**", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .anyRequest().authenticated())
            .httpBasic(Customizer.withDefaults())
            .build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();   // {bcrypt}$2a$10$... — supports upgrades
    }

    @Bean
    AuthenticationManager authenticationManager(AuthenticationConfiguration cfg) throws Exception {
        return cfg.getAuthenticationManager();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration c = new CorsConfiguration();
        c.setAllowedOrigins(List.of("https://shop.example.com"));
        c.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        c.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        c.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource src = new UrlBasedCorsConfigurationSource();
        src.registerCorsConfiguration("/**", c);
        return src;
    }
}
```
Authorization rule helpers: `permitAll()`, `denyAll()`, `authenticated()`, `anonymous()`, `hasRole`, `hasAnyRole`, `hasAuthority`, `hasAnyAuthority`, `access(AuthorizationManager)` (custom logic, e.g., IP checks), `fullyAuthenticated()`, `rememberMe()`.

## 6. Users from the database
```java
@Entity @Table(name = "app_user")
public class AppUser {
    @Id @GeneratedValue private Long id;
    @Column(unique = true, nullable = false) private String email;
    private String passwordHash;
    private boolean enabled = true;
    @ElementCollection(fetch = FetchType.EAGER) @Enumerated(EnumType.STRING)
    private Set<Role> roles = Set.of(Role.USER);
}

@Service
@RequiredArgsConstructor
public class DbUserDetailsService implements UserDetailsService {
    private final AppUserRepository repo;
    @Override
    public UserDetails loadUserByUsername(String email) {
        AppUser u = repo.findByEmailIgnoreCase(email)
            .orElseThrow(() -> new UsernameNotFoundException("User not found"));
        return User.withUsername(u.getEmail())
            .password(u.getPasswordHash())
            .disabled(!u.isEnabled())
            .roles(u.getRoles().stream().map(Enum::name).toArray(String[]::new))   // adds ROLE_ prefix
            .build();
    }
}
```
Boot auto-wires a `DaoAuthenticationProvider` using your `UserDetailsService` + `PasswordEncoder` beans. Other options: `InMemoryUserDetailsManager` (demos), `JdbcUserDetailsManager` (standard schema), LDAP.

### Registration
```java
@Transactional
public void register(RegisterRequest r) {
    if (repo.existsByEmailIgnoreCase(r.email())) throw new EmailTakenException();
    AppUser u = new AppUser(r.email(), passwordEncoder.encode(r.password()));
    repo.save(u);
}
```

## 7. Password storage
- **Never** store plain text or reversible encryption; never use MD5/SHA-1/SHA-256 alone (too fast → brute-forceable).
- Use adaptive, salted, slow hashes: **BCrypt** (default, cost factor 10), **Argon2id**, scrypt, PBKDF2.
- Salt is random per password and stored inside the hash string.
- `DelegatingPasswordEncoder` prefixes the algorithm id (`{bcrypt}`), allowing migration to stronger algorithms (`upgradeEncoding`).
- Compromised password check: `HaveIBeenPwnedRestApiPasswordChecker` (Security 6.3+).

## 8. Stateful (session) vs stateless (token) authentication
| | Session (cookie) | Token (JWT bearer) |
|--|--|--|
| Server state | Session store (memory/Redis via Spring Session) | None (self-contained token) |
| Sent as | `JSESSIONID` cookie automatically by browser | `Authorization: Bearer <token>` header |
| CSRF risk | **Yes** — need CSRF protection | No (if not stored in cookies) |
| Revocation | Easy (invalidate session) | Hard — short expiry + refresh tokens / denylist |
| Scaling | Sticky sessions or shared store | Easy horizontal scaling |
| Best for | Server-rendered web apps, BFF | APIs, mobile, microservices |

## 9. JWT (JSON Web Token)
Structure: `base64url(header).base64url(payload).signature`
```json
// header
{ "alg": "RS256", "typ": "JWT", "kid": "key-2026-10" }
// payload (claims)
{ "iss": "https://auth.shop.com", "sub": "42", "aud": "shop-api",
  "exp": 1760870400, "iat": 1760869500, "jti": "a1b2...",
  "scope": "orders:read orders:write", "roles": ["USER"] }
```
- **Signed (JWS), not encrypted** — anyone can read the payload. Never put secrets/PII in it.
- **HS256**: shared secret (same key signs and verifies) — fine for a single service; secret must be ≥256 bits.
- **RS256/ES256**: private key signs (auth server), public key verifies (resource servers fetch via **JWKS** endpoint) — preferred for microservices.
- Validate: signature, `exp`, `nbf`, `iss`, `aud`; reject `alg: none`.
- **Access token**: short-lived (5–15 min). **Refresh token**: long-lived, stored securely (httpOnly cookie / secure storage), rotated on use, revocable server-side.
- Logout with JWT: delete client tokens, revoke refresh token, optionally maintain a denylist of `jti` until expiry.
- Browser storage: `localStorage` is vulnerable to XSS; httpOnly + Secure + SameSite cookies are vulnerable to CSRF (mitigate with SameSite/CSRF tokens). BFF pattern keeps tokens server-side.

### 9.1 Option A — Spring's built-in resource server (recommended)
```xml
<artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
```
```yaml
# Tokens issued by an external IdP (Keycloak/Okta/Auth0/Cognito/Spring Authorization Server)
spring.security.oauth2.resourceserver.jwt.issuer-uri: https://auth.shop.com/realms/shop
# or jwk-set-uri, or a local public key: public-key-location: classpath:public.pem
```
```java
http.oauth2ResourceServer(o -> o.jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter())));

// Map "roles" claim to ROLE_ authorities (default maps "scope"/"scp" → SCOPE_xxx)
@Bean
JwtAuthenticationConverter jwtAuthConverter() {
    JwtGrantedAuthoritiesConverter roles = new JwtGrantedAuthoritiesConverter();
    roles.setAuthoritiesClaimName("roles");
    roles.setAuthorityPrefix("ROLE_");
    JwtAuthenticationConverter c = new JwtAuthenticationConverter();
    c.setJwtGrantedAuthoritiesConverter(roles);
    return c;
}
```
Issue your own tokens (single app / course style) with `JwtEncoder` (Nimbus):
```java
@Bean JwtEncoder jwtEncoder(RSAKey rsaKey) { return new NimbusJwtEncoder(new ImmutableJWKSet<>(new JWKSet(rsaKey))); }
@Bean JwtDecoder jwtDecoder(RSAKey rsaKey) throws JOSEException { return NimbusJwtDecoder.withPublicKey(rsaKey.toRSAPublicKey()).build(); }

@PostMapping("/api/auth/login")
public TokenResponse login(@RequestBody LoginRequest r) {
    Authentication auth = authenticationManager.authenticate(
        new UsernamePasswordAuthenticationToken(r.email(), r.password()));
    Instant now = Instant.now();
    JwtClaimsSet claims = JwtClaimsSet.builder()
        .issuer("shop-api").subject(auth.getName()).issuedAt(now).expiresAt(now.plus(15, ChronoUnit.MINUTES))
        .claim("roles", auth.getAuthorities().stream().map(GrantedAuthority::getAuthority)
                             .map(a -> a.replace("ROLE_", "")).toList())
        .build();
    String token = jwtEncoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
    return new TokenResponse(token, "Bearer", 900);
}
```

### 9.2 Option B — custom JWT filter (very common in tutorials/interviews, using jjwt)
```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String header = req.getHeader(HttpHeaders.AUTHORIZATION);
        if (header == null || !header.startsWith("Bearer ")) { chain.doFilter(req, res); return; }

        String token = header.substring(7);
        try {
            String username = jwtService.extractUsername(token);            // also validates signature & expiry
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails user = userDetailsService.loadUserByUsername(username);
                var auth = new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
                auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(req));
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        } catch (JwtException e) {
            res.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid token");
            return;
        }
        chain.doFilter(req, res);
    }
}

@Service
public class JwtService {
    private final SecretKey key;
    public JwtService(@Value("${jwt.secret}") String base64Secret) {
        this.key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(base64Secret));   // ≥ 32 bytes
    }
    public String generate(UserDetails user) {
        return Jwts.builder().subject(user.getUsername())
            .claim("roles", user.getAuthorities().stream().map(GrantedAuthority::getAuthority).toList())
            .issuedAt(new Date()).expiration(Date.from(Instant.now().plus(15, ChronoUnit.MINUTES)))
            .signWith(key).compact();
    }
    public String extractUsername(String token) {
        return Jwts.parser().verifyWith(key).build().parseSignedClaims(token).getPayload().getSubject();
    }
}
```
Prefer Option A in real projects (less custom security code = fewer bugs).

## 10. OAuth 2.0 & OpenID Connect
**OAuth 2.0** = delegated **authorization** framework (access tokens to call APIs). **OIDC** = identity layer on top (ID token, `/userinfo`) for **authentication** ("Login with Google").

Roles: **Resource Owner** (user), **Client** (your app), **Authorization Server** (Keycloak, Okta, Auth0, Google, Spring Authorization Server), **Resource Server** (your API).

### Grant types (flows)
| Flow | Use |
|------|-----|
| **Authorization Code + PKCE** | Web apps, SPAs, mobile — user login. The standard. |
| **Client Credentials** | Service-to-service (no user) |
| Refresh Token | Get new access token |
| Device Code | TVs/CLIs |
| ~~Implicit~~, ~~Password~~ | Deprecated (OAuth 2.1 removes them) |

**Authorization Code + PKCE**:
1. Client redirects user to `/authorize?response_type=code&client_id=..&redirect_uri=..&scope=openid profile&state=..&code_challenge=..`
2. User logs in & consents at the auth server.
3. Redirect back with `?code=...&state=...`.
4. Client exchanges code (+ `code_verifier`) at `/token` → access token (+ refresh token, + ID token for OIDC).
5. Client calls API with `Authorization: Bearer <access_token>`; resource server validates JWT via JWKS.

### Spring roles
| You are building | Starter | Config |
|------------------|---------|--------|
| API validating tokens | `oauth2-resource-server` | `.oauth2ResourceServer(o -> o.jwt(...))` (or opaque token introspection) |
| Web app with "Login with Google/Keycloak" | `oauth2-client` | `.oauth2Login(...)` + `spring.security.oauth2.client.registration.google.client-id/secret` |
| Service calling another service with client credentials | `oauth2-client` | `OAuth2AuthorizedClientManager` + RestClient interceptor (`OAuth2ClientHttpRequestInterceptor`, 6.4+) |
| Your own auth server | Spring Authorization Server | |

## 11. Method-level security
```java
@EnableMethodSecurity
...
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) {}

@PreAuthorize("hasAuthority('SCOPE_orders:read') and #customerId == authentication.name")
public List<Order> ordersFor(String customerId) {}

@PreAuthorize("@orderSecurity.isOwner(#orderId, authentication)")      // delegate to a bean
public OrderResponse get(Long orderId) {}

@PostAuthorize("returnObject.ownerEmail == authentication.name")
public Document load(Long id) {}

@PostFilter("filterObject.owner == authentication.name")               // filters returned collection
public List<Document> all() {}

@Secured("ROLE_ADMIN")                    // legacy-ish, needs securedEnabled = true
@RolesAllowed("ADMIN")                    // JSR-250, jsr250Enabled = true
```
Also: meta-annotations (`@IsAdmin` composed annotation), `@AuthorizeReturnObject` (6.3), custom `AuthorizationManager`.

**Object-level authorization** (IDOR prevention): always check the resource belongs to the caller — `/api/orders/123` must verify order 123 belongs to the logged-in user, not just that they're authenticated.

## 12. Accessing the current user
```java
@GetMapping("/me")
public UserResponse me(@AuthenticationPrincipal UserDetails user) { ... }
public Profile me(@AuthenticationPrincipal Jwt jwt) { jwt.getSubject(); jwt.getClaimAsString("email"); }
public void m(Authentication auth) { auth.getName(); auth.getAuthorities(); }
SecurityContextHolder.getContext().getAuthentication();   // anywhere on the request thread
```
`@Async`: propagate with `DelegatingSecurityContextAsyncTaskExecutor` or `SecurityContextHolder.setStrategyName(MODE_INHERITABLETHREADLOCAL)` (careful with pools).

## 13. CSRF
**Cross-Site Request Forgery**: a malicious site makes the victim's browser send an authenticated request (cookies are attached automatically) to your site.
- Protection: synchronizer token (hidden field / header `X-XSRF-TOKEN` matched against session/cookie), `SameSite=Lax/Strict` cookies, checking Origin.
- Spring enables CSRF by default for state-changing methods (POST/PUT/PATCH/DELETE).
- **Disable only for stateless APIs that don't use cookie authentication.** SPAs using session cookies: `csrf(c -> c.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))` (or `.spa()` in 6.4+).

## 14. Other security topics a backend dev should know
- **OWASP Top 10** (2021): Broken Access Control, Cryptographic Failures, Injection, Insecure Design, Security Misconfiguration, Vulnerable & Outdated Components, Identification & Authentication Failures, Software & Data Integrity Failures, Security Logging & Monitoring Failures, SSRF. (A 2025 edition was released — Supply-chain failures and mishandling of exceptional conditions are new entries.)
- **SQL injection** → parameterized queries/JPA parameters, never string concatenation.
- **XSS** → output encoding, Content-Security-Policy header, don't render untrusted HTML.
- **Mass assignment** → DTOs with only allowed fields.
- **Secrets** → env vars/Vault/K8s secrets; never in git; rotate.
- **HTTPS everywhere**, HSTS.
- **Rate limit** login endpoints; account lockout / captcha; MFA (Spring Security 6.x supports one-time-token login & passkeys/WebAuthn).
- **Dependency scanning** (OWASP Dependency-Check, Snyk, Dependabot), SBOM.
- **Security headers** (Spring adds defaults), CSP.
- **Audit logging** of auth events (`AuthenticationSuccessEvent`, `AbstractAuthenticationFailureEvent` listeners).
- **Actuator**: expose only health/info publicly, secure the rest.
- **mTLS / service mesh** for service-to-service in zero-trust networks.

## 15. Testing security
```java
@WebMvcTest(OrderController.class)
@Import(SecurityConfig.class)
class OrderControllerSecurityTest {
    @Autowired MockMvc mvc;

    @Test void anonymousGets401() throws Exception {
        mvc.perform(get("/api/orders")).andExpect(status().isUnauthorized());
    }
    @Test @WithMockUser(roles = "USER")
    void userCannotDelete() throws Exception {
        mvc.perform(delete("/api/admin/users/1").with(csrf())).andExpect(status().isForbidden());
    }
    @Test void withJwt() throws Exception {
        mvc.perform(get("/api/orders").with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_orders:read"))))
           .andExpect(status().isOk());
    }
}
```
`@WithMockUser`, `@WithUserDetails`, `SecurityMockMvcRequestPostProcessors.jwt()/user()/csrf()/httpBasic()/oauth2Login()`.

## 16. Interview Q&A
**Q. Authentication vs authorization?** Verifying identity vs verifying permissions.

**Q. Explain the Spring Security architecture.** DelegatingFilterProxy → FilterChainProxy → SecurityFilterChain filters; AuthenticationManager/ProviderManager → AuthenticationProviders → UserDetailsService + PasswordEncoder; result stored in SecurityContextHolder; AuthorizationFilter and method security enforce access; ExceptionTranslationFilter converts errors to 401/403.

**Q. How do you configure security in Spring Security 6?** Declare `SecurityFilterChain` beans with the lambda DSL (`authorizeHttpRequests`, `requestMatchers`); no WebSecurityConfigurerAdapter.

**Q. How does JWT authentication work?** Login → server issues signed token with claims & expiry → client sends Bearer token → filter validates signature/claims → builds Authentication → SecurityContext.

**Q. How do you revoke a JWT?** Short expiry + refresh token rotation/revocation; denylist by `jti`; key rotation for emergencies.

**Q. HS256 vs RS256?** Symmetric shared secret vs asymmetric key pair (public key verification, better for many services).

**Q. OAuth2 vs OIDC vs JWT?** OAuth2 = authorization framework; OIDC = authentication layer on OAuth2 (ID token); JWT = token format often used by both.

**Q. Which OAuth flow for service-to-service?** Client credentials.

**Q. Why disable CSRF for REST APIs?** CSRF exploits automatic cookie sending; stateless APIs authenticated via Authorization headers aren't vulnerable. Keep it if you use cookies.

**Q. `hasRole` vs `hasAuthority`?** `hasRole("ADMIN")` checks authority `ROLE_ADMIN`; `hasAuthority` checks the exact string.

**Q. How are passwords stored?** BCrypt/Argon2 salted adaptive hashes via `PasswordEncoder`; DelegatingPasswordEncoder supports algorithm upgrades.

**Q. What is CORS and how is it configured with Spring Security?** Browser same-origin policy relaxation via response headers; enable `http.cors()` with a `CorsConfigurationSource` so preflights pass the security chain.

**Q. How do you secure method calls?** `@EnableMethodSecurity` + `@PreAuthorize`/`@PostAuthorize` with SpEL.
