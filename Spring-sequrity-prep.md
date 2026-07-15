# Spring Security Interview Prep — 50 Questions (What / Why / How + Example)

---

## Section 1: Fundamentals

**1. What is Spring Security?**
- **What:** A framework providing authentication, authorization, and protection against common attacks (CSRF, session fixation, etc.) for Spring applications.
- **Why:** Rolling your own security (password handling, session management, access control) is easy to get wrong; Spring Security is battle-tested and integrates natively with Spring Boot.

**2. Authentication vs Authorization?**
- **Authentication** — "Who are you?" (verifying identity, e.g., login).
- **Authorization** — "What are you allowed to do?" (access control, e.g., only `ADMIN` can delete an employee record).

**3. What is the `SecurityFilterChain`?**
- **What:** A chain of servlet filters that every HTTP request passes through before reaching your controller — each filter handles one concern (auth, CSRF, session, exception translation, etc.).
- **How (Spring Boot 3+ style):**
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated())
        .formLogin(Customizer.withDefaults());
    return http.build();
}
```

**4. What replaced `WebSecurityConfigurerAdapter`?**
- **What:** Deprecated in Spring Security 5.7+ and removed in 6.x. Replaced by defining a `SecurityFilterChain` `@Bean` directly (component-based configuration), which is more testable and avoids inheritance-based config.

**5. What is the `SecurityContext` and `SecurityContextHolder`?**
- **What:** `SecurityContext` holds the currently authenticated user's details (`Authentication` object). `SecurityContextHolder` is a static accessor to get/set it — by default stored in a `ThreadLocal`, so it's tied to the current request's thread.
```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
```

**6. What is the `Authentication` object?**
- **What:** Represents the authenticated principal — holds `principal` (usually a `UserDetails`), `credentials`, `authorities` (roles/permissions), and authentication status.

**7. What is `UserDetails` and `UserDetailsService`?**
- **What:** `UserDetails` is Spring Security's contract for user info (username, password, authorities, account status flags). `UserDetailsService` is the interface you implement to load a user (typically from your DB) by username.
```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    @Autowired private EmployeeRepository repo;

    public UserDetails loadUserByUsername(String username) {
        Employee emp = repo.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("Not found"));
        return User.builder()
            .username(emp.getEmail())
            .password(emp.getPasswordHash())
            .roles(emp.getRole())
            .build();
    }
}
```

---

## Section 2: Password Handling

**8. Why should you never store plain-text passwords?**
- **Why:** If your DB is ever breached, plain-text passwords expose every user immediately, and since people reuse passwords, it compromises their accounts elsewhere too.

**9. What is `PasswordEncoder` and why `BCryptPasswordEncoder` specifically?**
- **What:** Interface for hashing/verifying passwords. `BCryptPasswordEncoder` is preferred because it's a **slow, adaptive** hash (built-in salt + configurable work factor), which makes brute-force attacks impractical — unlike fast hashes like plain SHA-256/MD5, which are designed for speed and are bad for passwords.
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
// Usage
String hashed = passwordEncoder.encode("plainPassword");
boolean matches = passwordEncoder.matches("plainPassword", hashed);
```

**10. Why is a salt important in password hashing?**
- **Why:** Prevents attackers from using precomputed rainbow tables — even if two users have the same password, their hashes differ because of unique salts. BCrypt generates and stores the salt automatically as part of the hash output.

---

## Section 3: Session vs Token-Based Auth

**11. Session-based vs Token-based (JWT) authentication — what's the difference?**
- **Session-based** — server creates a session on login, stores state (session ID in a cookie), server must track sessions (stateful).
- **Token-based (JWT)** — server issues a signed token containing claims; server verifies the signature on each request without storing session state (stateless) — better for scaling across multiple servers/microservices.

**12. What is a JWT (JSON Web Token) and its structure?**
- **What:** A compact, URL-safe token with three parts: **Header** (algorithm/type), **Payload** (claims — user ID, roles, expiry), **Signature** (verifies the token hasn't been tampered with).
- **Why:** Self-contained — the server doesn't need to look up session data, it just verifies the signature.
```
header.payload.signature
```

**13. How do you validate a JWT on each request?**
- **How:** A custom filter extracts the token from the `Authorization: Bearer <token>` header, verifies the signature + expiry, then manually sets the `Authentication` in the `SecurityContext` if valid.
```java
public class JwtAuthFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
        String token = extractToken(req);
        if (token != null && jwtUtil.isValid(token)) {
            UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(
                jwtUtil.getUsername(token), null, jwtUtil.getAuthorities(token));
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(req, res);
    }
}
```

**14. Why is JWT authentication "stateless" and why does that matter for microservices?**
- **Why it matters:** No server-side session store means any instance of a horizontally-scaled service can validate a request independently — no sticky sessions or shared session store needed. Directly relevant to a microservices platform like Humine.

**15. What are the downsides of JWT vs sessions?**
- Can't easily "revoke" a token before its expiry (since the server doesn't track state) without extra infrastructure (blocklist/short expiry + refresh tokens). Larger payload than a session ID cookie. If the signing key leaks, all tokens are compromised.

**16. What is a Refresh Token and why use one alongside an access token?**
- **What:** A long-lived token used to obtain a new short-lived access token without forcing the user to log in again.
- **Why:** Keeps access tokens short-lived (limits damage if leaked) while avoiding constant re-authentication.

---

## Section 4: Authorization

**17. `hasRole()` vs `hasAuthority()`?**
- `hasRole("ADMIN")` — automatically expects/prepends `ROLE_` prefix (checks for `ROLE_ADMIN`).
- `hasAuthority("ADMIN")` — checks the exact authority string, no prefix added. Roles are really just a specialized authority convention.

**18. Method-level security — what and how?**
- **What:** Securing individual service/controller methods instead of (or in addition to) URL patterns.
- **How:** Enable with `@EnableMethodSecurity`, then annotate methods.
```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteEmployee(Long id) { ... }

@PreAuthorize("#employeeId == authentication.principal.id or hasRole('ADMIN')")
public Employee getEmployee(Long employeeId) { ... }
```

**19. `@PreAuthorize` vs `@PostAuthorize`?**
- `@PreAuthorize` — check runs **before** method execution (can reject the call entirely).
- `@PostAuthorize` — check runs **after** the method executes, useful when the check depends on the returned object (e.g., "only return this if `returnObject.ownerId == principal.id`").

**20. What is `@Secured` and how is it different from `@PreAuthorize`?**
- `@Secured` — simpler, older, only supports role checks, no SpEL expressions.
- `@PreAuthorize` — supports full Spring Expression Language (SpEL), far more flexible (as in Q18's example).

**21. How do you restrict access at the URL/endpoint level vs method level — when to use which?**
- **URL-level** (`authorizeHttpRequests`) — coarse-grained, good for broad patterns like "everything under `/admin/**` needs ADMIN role."
- **Method-level** (`@PreAuthorize`) — fine-grained, needed when access depends on business logic (e.g., "employee can only view their own leave records").

**22. What is RBAC (Role-Based Access Control)?**
- **What:** Access is granted based on a user's assigned role(s) (e.g., `EMPLOYEE`, `MANAGER`, `HR_ADMIN`), rather than per-user permissions — simpler to manage at scale.

---

## Section 5: CSRF, CORS & Common Attacks

**23. What is CSRF (Cross-Site Request Forgery)?**
- **What:** An attack where a malicious site tricks a logged-in user's browser into making an unwanted request to your app (using their existing session cookie), without their knowledge.
- **Why session-based apps are vulnerable:** Browsers auto-attach cookies to requests, so the malicious request looks "authenticated" from the server's perspective.

**24. How does Spring Security protect against CSRF?**
- **How:** Requires a unique, unpredictable CSRF token to be sent with state-changing requests (`POST`/`PUT`/`DELETE`), which an attacker's cross-origin request can't know/replicate.
```java
http.csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()));
```

**25. Why is CSRF protection often disabled for stateless REST APIs using JWT?**
- **Why:** CSRF exploits *cookie-based* auth (browser auto-sends cookies). If you're using JWTs sent explicitly via the `Authorization` header (not auto-attached by the browser), there's no ambient credential for an attacker to hijack — so CSRF risk is much lower, and it's commonly disabled for such stateless token-based APIs.
```java
http.csrf(csrf -> csrf.disable()); // Common for stateless JWT APIs — but understand WHY before doing this
```

**26. What is CORS and how is it different from CSRF?**
- **CORS (Cross-Origin Resource Sharing)** — a *browser-enforced* mechanism controlling which origins are allowed to call your API from JavaScript. It's a **relaxation** mechanism (you explicitly allow origins), whereas CSRF protection is a **defense** mechanism.
```java
@Bean
public WebMvcConfigurer corsConfigurer() {
    return new WebMvcConfigurer() {
        public void addCorsMappings(CorsRegistry registry) {
            registry.addMapping("/api/**").allowedOrigins("https://humine.decathlon.in");
        }
    };
}
```

**27. What is Session Fixation and how does Spring Security prevent it?**
- **What:** An attacker sets/knows a victim's session ID before login, then hijacks the session once the victim authenticates (if the session ID doesn't change post-login).
- **How prevented:** Spring Security by default creates a **new session ID** upon successful authentication (`sessionFixation().migrateSession()` or `.newSession()`).

**28. What is XSS (Cross-Site Scripting) and how does it relate to Spring apps?**
- **What:** Injecting malicious scripts into pages viewed by other users (e.g., via unsanitized user input rendered in HTML). Spring Security itself doesn't fully prevent XSS (that's mostly about output encoding at the view layer), but it does add security headers like `Content-Security-Policy` and `X-XSS-Protection` to reduce risk.

**29. What is Clickjacking and how does Spring Security's default header help?**
- **What:** Tricking a user into clicking something on your site by embedding it in an invisible iframe on an attacker's page.
- **How:** Spring Security adds `X-Frame-Options: DENY` by default, preventing your pages from being framed by other sites.

---

## Section 6: OAuth2 & Modern Auth

**30. What is OAuth2 in simple terms?**
- **What:** An authorization framework letting a user grant a third-party app limited access to their resources on another service, without sharing their password (e.g., "Sign in with Google").

**31. OAuth2 vs OpenID Connect (OIDC)?**
- OAuth2 is purely about **authorization** (access tokens for resources). OIDC is built on top of OAuth2 and adds **authentication** (an ID token confirming *who* the user is, using JWT).

**32. What are the main OAuth2 grant types (flows)?**
- **Authorization Code** — most common for web apps (redirect-based, exchanges a code for a token server-side).
- **Client Credentials** — service-to-service (no user involved), common in microservices.
- **Refresh Token** — get a new access token without re-authenticating.
- (PKCE extension is now standard for public/mobile clients, replacing the older Implicit flow.)

**33. What is `spring-boot-starter-oauth2-resource-server` used for?**
- **What:** Configures your Spring Boot app as a **resource server** that validates incoming JWTs (typically issued by an external identity provider like Keycloak/Auth0/Okta) rather than issuing tokens itself.
```java
http.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
```

**34. What is an Identity Provider (IdP) and why use one instead of building your own auth?**
- **Why:** Offloads password storage, MFA, social login, and compliance burden to a specialized, audited service (Keycloak, Auth0, Okta, Azure AD) — reduces your app's attack surface significantly.

---

## Section 7: Filters & Internals

**35. How do you add a custom filter to the Spring Security filter chain?**
```java
http.addFilterBefore(new JwtAuthFilter(), UsernamePasswordAuthenticationFilter.class);
```

**36. What is `OncePerRequestFilter` and why extend it for custom filters?**
- **Why:** Guarantees the filter's logic executes exactly once per request, even in scenarios involving internal request forwarding/dispatching that could otherwise trigger a filter multiple times.

**37. What is the `AuthenticationManager` and `AuthenticationProvider`?**
- **What:** `AuthenticationManager` is the entry point that attempts authentication (delegates to one or more `AuthenticationProvider`s). `AuthenticationProvider` contains the actual logic to verify credentials (e.g., `DaoAuthenticationProvider` checks username/password against `UserDetailsService` + `PasswordEncoder`).

**38. What is `AccessDeniedException` vs `AuthenticationException` — when does each occur?**
- `AuthenticationException` — thrown when authentication itself fails (bad credentials, expired token) → typically results in `401 Unauthorized`.
- `AccessDeniedException` — thrown when the user IS authenticated but lacks permission for the resource → `403 Forbidden`.

**39. How do you customize the response for unauthorized/forbidden requests?**
```java
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint((req, res, e) -> res.sendError(HttpServletResponse.SC_UNAUTHORIZED))
    .accessDeniedHandler((req, res, e) -> res.sendError(HttpServletResponse.SC_FORBIDDEN)));
```

---

## Section 8: Practical / Scenario-Based (High-Value for 3-5 YOE Interviews)

**40. How would you secure a REST API where employees can only view their own leave records, but HR admins can view everyone's?**
- **How:** Combine role-based check with ownership check via `@PreAuthorize` SpEL, comparing the authenticated principal's ID to the requested resource's owner ID (as shown in Q18), or enforce it in the service layer by always scoping DB queries to the authenticated user unless the role is ADMIN.

**41. How would you prevent an IDOR vulnerability (like the one you fixed on salary document downloads) using Spring Security?**
- **How:** Never trust a client-supplied ID alone — after loading the resource, explicitly verify `resource.getOwnerId().equals(currentUser.getId())` (or that the user's role permits cross-user access) before returning it. This check belongs in the service layer, not just relying on "hard to guess" IDs or front-end restrictions.

**42. How do you handle password reset securely?**
- **How:** Generate a single-use, time-limited, cryptographically random token, store its hash (not the raw token) with an expiry, email a link containing the raw token, and invalidate it after first use or expiry.

**43. What's the risk of returning different error messages for "user not found" vs "wrong password" on login?**
- **Why it matters:** User enumeration — an attacker can use the different responses to figure out which usernames/emails are registered. Best practice: return a generic "invalid credentials" message for both cases.

**44. How would you rate-limit login attempts to prevent brute-force attacks?**
- **How:** Track failed attempts per username/IP (in-memory cache, Redis, or DB), temporarily lock the account or add exponential backoff after N failures — conceptually similar to the tiered rate limiter you built for the notification service.

**45. How do you securely store a JWT on the client side (web app)?**
- **Trade-off:** `localStorage` is accessible to JS → vulnerable to XSS. `httpOnly` secure cookies aren't accessible to JS (safer from XSS) but need CSRF protection since they're auto-sent by the browser. Many modern apps use short-lived access tokens in memory + httpOnly refresh token cookies as a balance.

**46. How would you implement "logout" with JWT, given tokens can't be easily invalidated server-side?**
- **How:** Client discards the token (basic case), or maintain a server-side blocklist of revoked token IDs (`jti` claim) checked on each request until expiry, or use short expiries + refresh token rotation so a stolen token has a small window of validity.

**47. What is Multi-Factor Authentication (MFA) and how might you add it to a Spring Boot app?**
- **What:** Requiring a second verification factor (OTP, authenticator app) beyond just a password.
- **How:** Typically implemented as an intermediate authentication step — after password validation succeeds, issue a temporary "pending MFA" state/token, require an OTP verification endpoint before issuing the final full-access token.

**48. How do you secure sensitive configuration like DB passwords and JWT signing secrets in a Spring Boot app?**
- **How:** Never hardcode/commit them — use environment variables, a secrets manager (AWS Secrets Manager, HashiCorp Vault), or Spring Cloud Config with encryption, injected at runtime via `application.yml` placeholders (`${DB_PASSWORD}`).

**49. What is the principle of least privilege, and how does it apply to a microservices setup like Humine?**
- **What:** Each user/service should have only the minimum permissions necessary to do its job.
- **How it applies:** A notification service shouldn't have write access to payroll data; an HR admin role shouldn't automatically have infra/DB-level access — access should be scoped tightly per service and per role.

**50. How would you test Spring Security configuration (e.g., that an endpoint correctly requires ADMIN role)?**
- **How:** Use `@WithMockUser(roles = "ADMIN")` (or `@WithMockUser` with no role for negative tests) in Spring MVC tests with `MockMvc`, asserting expected status codes (`200` vs `403`) without needing a real login flow.
```java
@Test
@WithMockUser(roles = "EMPLOYEE")
void nonAdminCannotDeleteEmployee() throws Exception {
    mockMvc.perform(delete("/api/employees/1")).andExpect(status().isForbidden());
}
```

---

## Quick tips for your interviews
- **JWT vs session (Q11-16) and CSRF-vs-CORS (Q23-26)** are the two most commonly confused topic pairs — interviewers often ask you to explain the *difference* explicitly, not just define each separately.
- **Q40-41 (ownership-based authorization)** ties directly to your real IDOR fix on salary document downloads — lead with that story, it's a strong, concrete answer.
- Be ready to explain **why** you'd disable CSRF for a stateless JWT API (Q25) rather than just stating it as a rule — interviewers probe here to see if you understand the underlying threat model, not just memorized config.

Want this added into the mixed mock-panel round with your Spring Boot, Core Java, SQL, and Streams docs?
