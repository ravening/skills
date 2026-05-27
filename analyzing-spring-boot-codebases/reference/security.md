# Security Review

## Review areas

Assess these categories:

- access control and authorization
- authentication and token handling
- password storage and encoding
- CSRF and CORS posture
- endpoint exposure
- secrets in source control or config
- SQL, JPQL, and command injection risk
- SSRF risk in outbound HTTP calls
- unsafe deserialization or object mapping
- actuator and admin surface exposure
- sensitive data serialization and logging
- file upload validation
- dependency vulnerabilities

## JWT pitfalls

Treat JWT handling as a frequent source of authentication bypass. Walk the token lifecycle (issue, transmit, verify, refresh, revoke) and confirm the verifier rejects tokens that do not match the expected shape rather than trusting fields inside the token.

Look for:

- acceptance of `alg=none` or any unsigned token, including code paths that fall through to "no signature required" when the algorithm header is missing or unrecognised.
- algorithm confusion where an HMAC verifier accepts an `RS256` token (or vice versa) because the algorithm is read from the token header instead of pinned by the verifier configuration.
- `kid` (key id) injection: the verifier loads a key by following an attacker-controlled `kid` value, for example treating it as a file path, URL, JWKS lookup key without an allow-list, or database identifier.
- missing or partial validation of `iss`, `aud`, `exp`, `nbf`, and `iat`. A token that validates only the signature is not authenticated for your application; issuer and audience must be pinned, and expiry must be enforced with a bounded clock skew.
- weak signing material: short HMAC secrets, secrets reused across services or environments, secrets committed to source or baked into images, RSA or EC keys with insufficient size, and absent key rotation.
- excessive token lifetime, missing revocation strategy, and storage of tokens in places readable by client-side scripts when a session cookie would be appropriate.
- sensitive data placed in JWT claims under the assumption that the token is confidential; JWTs are signed, not encrypted, unless JWE is explicitly used.

Severity mapping: confirmed signature bypass, `alg=none` acceptance, algorithm confusion, or missing signature verification on protected endpoints map to **P0**. Missing `iss` / `aud` / `exp` enforcement, weak signing keys, and unsafe `kid` handling typically map to **P1**. Overly long lifetimes, weak rotation, and sensitive claims map to **P2**. Hardening such as tighter clock skew or richer claim assertions maps to **P3**.

## OAuth2 and OIDC flow review

Identify which flows the application uses (authorization code, authorization code with PKCE, client credentials, device code, refresh token), then check each against its expected guarantees. Treat any flow that handles user authentication in the browser without PKCE as suspect on modern Spring Security.

Look for:

- authorization code flow without PKCE for public or browser-based clients, or PKCE present but with `plain` rather than `S256` challenge method.
- redirect URI handling that performs partial matching, allows wildcards in host or path, accepts arbitrary query or fragment additions, or relies on a denylist instead of an exact-match allow-list. Confirm registered redirect URIs are reviewed and that the relying-party configuration matches.
- `state` and `nonce` parameters omitted, not bound to the user session, or not validated on callback, leaving the flow open to CSRF and replay against the token endpoint.
- client credentials flow used from a context that is actually acting on behalf of an end user, or client secrets shipped to a public client.
- refresh-token handling without rotation, without reuse detection, or without revocation when reuse is detected; refresh tokens stored in browser-accessible storage; long-lived refresh tokens with no idle or absolute expiry.
- ID token validation that skips signature, `iss`, `aud`, `nonce`, `exp`, or `azp` checks, or that trusts userinfo claims without re-validating the ID token.
- scopes that are far broader than the resource server requires, or scope checks performed only at the gateway and not at the resource server.
- logout that clears the local session but does not perform RP-initiated logout against the identity provider when single sign-on is in scope.

Severity mapping: missing PKCE on a public client, redirect URI bypass, missing `state` / `nonce` validation, or accepting tokens without verifying signature and `iss` / `aud` map to **P0** when exploitable. Refresh-token rotation absent, weak scope separation, or missing reuse detection typically map to **P1**. Overly broad scopes, weak logout, and minor flow hardening map to **P2** or **P3** depending on exposure.

## Session management

Review how server-side sessions are created, bound to the authenticated principal, propagated, and destroyed. Confirm the application is explicit about whether it uses sessions, stateless tokens, or a mix.

Look for:

- session fixation: the session identifier is not rotated on authentication, allowing a pre-authentication identifier to become an authenticated one. Confirm Spring Security's session-fixation protection is enabled and not overridden.
- session cookies missing `Secure`, `HttpOnly`, or an appropriate `SameSite` attribute; session cookies scoped to overly broad domains or paths; session identifiers exposed in URLs.
- concurrent session policy: unlimited concurrent sessions per principal where the threat model requires single-session enforcement, or no visibility into active sessions for administrative revocation.
- idle and absolute session timeouts that are missing, far too long, or inconsistent between the application and the reverse proxy.
- logout that invalidates the local session but leaves refresh tokens, remember-me cookies, CSRF tokens, or downstream tokens valid; logout endpoints reachable by `GET` without CSRF protection; logout that does not clear server-side session state in distributed session stores.
- remember-me tokens without rotation, persisted with weak hashing, or accepted across security-context changes such as password reset.
- mixing stateful sessions with stateless JWT auth in ways that create two parallel authentication paths with different guarantees.

Severity mapping: confirmed session fixation, missing logout invalidation on protected systems, or session identifiers exposed in transit map to **P0**. Missing `Secure` / `HttpOnly` / `SameSite` on session cookies, weak concurrent-session policy in sensitive contexts, or remember-me without rotation map to **P1**. Timeout tuning and minor cookie hardening map to **P2** or **P3**.

## Deserialization gadget chains

Treat any code path that turns untrusted bytes into Java objects as a high-risk surface. Spring applications most often expose this through Jackson, but legacy `ObjectInputStream` usage, XML decoders, and YAML loaders matter equally when present.

Look for:

- Jackson polymorphic typing enabled globally, for example default typing on the shared `ObjectMapper`, or `@JsonTypeInfo` with `Id.CLASS` / `Id.MINIMAL_CLASS` accepting an attacker-controlled type name.
- polymorphic typing scoped narrowly but without a strict allow-list of permitted subtypes, or with an allow-list that includes broad base types from the JDK or commonly vulnerable libraries.
- `@JsonTypeInfo` declared on a base type that is reachable from request-bound DTOs, so any endpoint accepting that DTO inherits the polymorphic surface.
- direct use of `ObjectInputStream` (or wrappers around it) to read data crossing a trust boundary: HTTP bodies, message-broker payloads, cache entries, session stores, file uploads. Look for any code reading serialized objects from the network without an `ObjectInputFilter`.
- XML decoders such as `XMLDecoder`, or YAML loaders configured without safe constructors, used on untrusted input.
- caches and session stores configured to serialize Java objects (rather than JSON or a constrained schema) when the cache or session content can be influenced by a less-trusted tier.

Severity mapping: confirmed gadget chain reachable from an unauthenticated or low-privilege endpoint maps to **P0**. Polymorphic typing or `ObjectInputStream` usage on untrusted input that has not been demonstrated exploitable but lacks an allow-list or filter typically maps to **P1**. Polymorphic typing narrowly scoped with an allow-list but accepting more types than required maps to **P2**. Hardening recommendations such as moving from polymorphic typing to explicit schemas map to **P3**.

## SpEL and Expression Language injection

Spring Expression Language and JSP / JSF expression languages execute code, so any place where user input reaches an expression evaluator is treated as remote code execution unless proven otherwise.

Look for:

- user-controlled values concatenated into SpEL passed to `ExpressionParser`, `@Value` defaults that interpolate request data, or `@PreAuthorize`, `@PostAuthorize`, `@Cacheable`, and `@EventListener(condition = ...)` expressions whose strings include input from headers, request parameters, or persisted user-controlled fields.
- template engines (Thymeleaf, Velocity, Freemarker) configured in modes that allow expression evaluation on user-controlled fragments, especially server-side fragment inclusion driven by request parameters.
- Spring Data query annotations or actuator-style runtime configuration that accepts SpEL from configuration sources writable by less-trusted operators.
- frameworks that evaluate JEXL, OGNL, MVEL, or JavaScript on user input, particularly when carried over from older Struts or expression-driven mappers.
- error pages, logging frameworks, or notification templates that re-evaluate user-supplied messages as expressions.

Severity mapping: any confirmed expression injection reachable from untrusted input is **P0**. Expression evaluation on input that crosses a trust boundary but is partially constrained (for example numeric coercion before evaluation) maps to **P1** until proven non-exploitable. Latent expression evaluation on operator-controlled config that is still writable through less-trusted channels maps to **P2**. Hardening such as switching from SpEL to fixed predicates maps to **P3**.

## Path traversal patterns

Wherever the application turns a string into a filesystem path, a resource lookup, or an archive entry, confirm that traversal sequences cannot escape the intended directory.

Look for:

- request parameters, header values, multipart filenames, or persisted identifiers used to build file paths via simple string concatenation, including `Paths.get(base, userInput)` without canonicalisation and containment checks.
- `MultipartFile.getOriginalFilename()` used directly as a destination filename, with no normalisation, no rejection of `..` segments, and no rejection of absolute paths or Windows drive prefixes.
- `ResourceLoader` / `ClassPathResource` / `FileSystemResource` constructed from user input without restriction to a known prefix.
- archive extraction (zip, tar) that writes entries by name without verifying the resolved path remains under the extraction root ("zip slip").
- response code paths that serve files by name, including download controllers, static-resource overrides, and Thymeleaf or JSP includes whose target is request-driven.
- symlink-following extractors or copies operating on directories that contain untrusted symlinks.

Severity mapping: confirmed read or write outside the intended directory, or an exploitable upload that lands as an executable artifact, is **P0**. Unvalidated filename usage on protected endpoints typically maps to **P1**. Containment checks present but weak (for example string `startsWith` on non-canonical paths) map to **P2**. Defensive hardening recommendations map to **P3**.

## Mass-assignment risks via `@ModelAttribute` and `@RequestBody` binding

Spring data binding will populate any settable property on the target type unless the application restricts it. Treat every `@ModelAttribute` and `@RequestBody` parameter as a contract that the client may push arbitrary fields against.

Look for:

- controllers binding directly to JPA entities or to domain aggregates rather than to purpose-built request DTOs. Any field on the entity (for example `id`, `role`, `enabled`, `tenantId`, `balance`, `createdAt`) becomes assignable from the request body.
- DTOs that include privileged fields (role, permission set, ownership, audit timestamps, foreign-key references to records the user does not own) without server-side overwrite after binding.
- absence of allow-lists: no `setDisallowedFields` / `setAllowedFields` on `WebDataBinder`, no Jackson configuration to fail on unknown properties, no Bean Validation groups separating create from update payloads.
- update endpoints that perform "load entity, bind request, save entity" without re-asserting fields the user must not change, allowing a `PATCH` or `PUT` to silently overwrite ownership or status.
- nested binding on collections and references, where the client supplies an `id` for a child entity and the server attaches it to the parent without authorising the relationship.
- `@ModelAttribute` on form flows where hidden fields are assumed to be tamper-proof.

Severity mapping: privilege escalation through mass assignment (for example setting `role=ADMIN` or reassigning `tenantId`) is **P0**. Binding to entities on protected endpoints without server-side overwrite is **P1**. DTOs that include avoidable fields without explicit allow-lists map to **P2**. Refactoring guidance toward dedicated request and response DTOs maps to **P3**.

## Race conditions in authentication and business logic

Concurrency bugs are easy to miss in review because each individual code path looks correct. Look explicitly at sequences where a check and an action are separated, and at shared mutable state under load.

Look for:

- check-then-act in authentication and authorization: verifying that a record exists, is active, or is owned by the caller, then mutating it in a separate statement, with no transactional boundary or pessimistic / optimistic locking. Login, password reset, MFA enrolment, and token exchange are common offenders.
- token issuance and refresh flows that accept the same refresh token concurrently and issue multiple valid sessions, especially when reuse detection and rotation are missing.
- account-lockout and rate-limit counters incremented without atomic operations or distributed coordination, allowing brute-force attempts to outrun the limiter.
- "double-spend" patterns in business logic: balance checks, inventory reservations, coupon redemption, voting, or one-shot actions implemented with read-modify-write at the application tier instead of database-level constraints, unique indexes, or `SELECT ... FOR UPDATE`.
- idempotency keys absent on mutating endpoints that retry, allowing duplicates to be created when the client or a proxy retries on timeout.
- shared in-process caches mutated without synchronisation under multi-threaded request handling, leading to stale authorization decisions or cross-user data leaks.
- scheduled jobs and `@Async` tasks that bypass the same locks the synchronous path enforces.

Severity mapping: exploitable authentication or authorization races (account takeover, MFA bypass, token reuse) are **P0**. Business-logic races with monetary or data-integrity impact are typically **P1**. Counter and rate-limit races without direct privilege impact map to **P2**. Defensive hardening such as adding idempotency keys preemptively maps to **P3**.

## Template injection in Thymeleaf and Freemarker

Server-side template engines evaluate expressions, so any path where user input becomes part of the template (rather than data passed to the template) is treated as code execution.

Look for:

- Thymeleaf templates assembled by string concatenation with user input, or rendered via `engine.process(userControlledFragment, context)` instead of resolving a fixed template name. Watch especially for `th:utext` on user-controlled values, dynamic fragment expressions (`th:insert`, `th:replace`) with user-driven targets, and inline JavaScript blocks containing unescaped variables.
- Freemarker templates loaded by name from user input, or `?eval` and `?interpret` applied to user-controlled strings; access to `Execute`, `ObjectConstructor`, or `freemarker.template.utility.JythonRuntime` not removed from the configuration's class resolver.
- template caches whose keys include user input, allowing an attacker to register a hostile template under a predictable key.
- error handlers that render request fields back into a template without escaping, or admin views that preview user-submitted templates with the production engine instead of a sandboxed configuration.
- mixing of template engines on the same path (for example a Thymeleaf result post-processed by Freemarker) where the second pass re-evaluates content the first pass treated as data.

Severity mapping: confirmed template injection reachable from untrusted input is **P0**. Dynamic fragment selection or `th:utext` on user data without proven exploitability typically maps to **P1**. Use of `?eval`, `?interpret`, or unrestricted class resolvers in Freemarker even on operator-controlled templates maps to **P2** because of the blast radius if the input boundary later moves. General hardening (removing unused utilities from the resolver, enforcing strict escaping modes) maps to **P3**.

## What to flag

### P0
- hardcoded production secrets
- open admin or actuator endpoints with sensitive exposure
- broken auth on protected business endpoints
- confirmed injection vulnerabilities
- unsafe file handling leading to remote code or data compromise

### P1
- overly broad `permitAll`
- CSRF disabled where browser session flows exist
- wildcard CORS in sensitive environments
- weak password hashing or insecure custom crypto
- token leakage in logs

### P2
- missing method-level authorization
- incomplete validation at boundaries
- insecure defaults that are not clearly exploitable
- excessive data exposure in DTOs or entity serialization

### P3
- security hardening improvements
- missing headers or observability improvements

## OWASP summary

Map findings to:

- Broken Access Control
- Cryptographic Failures
- Injection
- Insecure Design
- Security Misconfiguration
- Vulnerable Components
- Identification and Authentication Failures
- Software and Data Integrity Failures
- Security Logging and Monitoring Failures
- SSRF

## Evidence rules

Always include:

- file path
- line number when possible
- why the issue matters
- exact remediation
