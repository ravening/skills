# API and Contracts

## Scope

This file covers the API surface of a Spring Boot service: how endpoints are documented, versioned, made safe to retry, throttled, and how protocol-specific risks are reviewed for REST, GraphQL, and gRPC. It does NOT cover authentication and authorization mechanics (OAuth2, JWT, session, mass-assignment binding), which remain owned by [reference/security.md](security.md). When an API concern touches authentication — for example, idempotency-key trust, version skew on auth headers, or rate-limit bypass via auth tokens — defer to that file for the underlying review depth.

Cross-references: [reference/security.md](security.md) for API authentication and authorization. Rate limiting overlaps with the broken-access-control / abuse topics in `security.md`; the catalog of throttling controls lives here, while the abuse-resistance perspective lives there.

## Review areas

Examine these categories when an API surface is present:

- OpenAPI / Swagger completeness and accuracy
- API versioning strategy and version skew handling
- Idempotency for mutating endpoints
- Rate limiting and throttling at the application boundary
- GraphQL-specific exposure and abuse risks
- gRPC-specific exposure and abuse risks (only when gRPC is present)

## OpenAPI / Swagger completeness review

The contract is the API. If `springdoc-openapi-*` (or an equivalent generator) is wired in, the agent should treat the produced OpenAPI document as the single source of truth and check it against the actual controller surface.

What to look for:

- Every `@RestController` endpoint is reflected in the generated OpenAPI document with an accurate path, HTTP method, and operation ID.
- Request and response bodies use real DTOs rather than `Object`, `Map<String, Object>`, raw `JsonNode`, or `byte[]` placeholders that erase the schema.
- Path variables, query parameters, headers, and cookies are declared with correct types, required flags, and constraints (`@NotNull`, `@Size`, `@Pattern`, `@Min`, `@Max`).
- Error responses are documented (4xx and 5xx) with a consistent error DTO rather than only the happy path.
- Authentication requirements are declared via security schemes and applied per operation; public endpoints are deliberately marked, not silently public.
- Examples, when present, do not embed real secrets, internal hostnames, or production identifiers.
- Deprecated endpoints are marked `deprecated: true` with sunset metadata and not just left undocumented.

What to record: missing operations, schema-erased bodies, undeclared error responses, public endpoints with no security scheme, and stale examples. Severity is typically P2 for documentation gaps and P1 when missing security scheme declarations cause the document to misrepresent the actual access posture.

## API versioning strategy

Pick one strategy and apply it consistently. Mixed strategies inside one service create version skew that callers cannot reason about.

Three approaches the agent should recognize:

- **URI versioning** — the version is part of the path, e.g. `/v1/orders`, `/v2/orders`. Easy to route, easy to cache, and visible in logs and dashboards. Downside: every version change is a path change for the caller.
- **Header versioning** — the version is sent in a custom header (e.g. `X-API-Version: 2`) or in `Accept-Version`. Keeps URLs stable but pushes versioning into the client and is invisible to most HTTP caches and CDN configurations.
- **Media-type versioning** — the version travels in the `Accept` header as a vendor media type, e.g. `Accept: application/vnd.acme.order.v2+json`. Most expressive (per-resource versioning, content negotiation) but the highest cognitive load for clients and tooling.

What to look for:

- One strategy is chosen and applied across all controllers. Mixed strategies (some endpoints URI-versioned, others header-versioned) are a finding.
- Default behavior when no version is supplied: the service either rejects the request or pins to a documented default. A silent pin to "latest" is a finding because it changes contract without a client opt-in.
- Deprecation path: deprecated versions are documented in the OpenAPI document and announced via a `Deprecation` or `Sunset` header. Endpoints removed without a deprecation window are a finding.
- Breaking changes inside a single version (renaming a field, changing a type, narrowing an enum) are a contract break and should be a finding regardless of which strategy is in use. Authentication-related breaking changes (e.g. switching from session to bearer-token requirements) cross over into [reference/security.md](security.md).

Severity is typically P2 for inconsistent strategy and P1 for silent breaking changes inside an existing version.

## Idempotency keys for mutating endpoints

Mutating endpoints (`POST`, `PUT`, `PATCH`, `DELETE`) that touch money, inventory, account state, or external side effects must tolerate duplicate delivery. Networks retry, mobile clients retry, and queue consumers replay. An endpoint that processes the same request twice and produces two charges or two orders is a defect even if the code looks correct in isolation.

What to look for:

- Mutating endpoints accept an `Idempotency-Key` header (or equivalent) and the controller documents it as required for non-naturally-idempotent operations.
- A persistent idempotency store (database table or distributed cache with durability appropriate to the operation) records the key, the request fingerprint, and the prior result. In-memory-only stores are a finding for any endpoint whose operation must survive a restart.
- The replay path returns the original response (status code and body) for a matching key + matching request fingerprint, returns a conflict for a matching key + diverging fingerprint, and processes normally for a new key.
- Key TTL is documented and matches the realistic retry window of the callers (typically 24 hours or longer for human-driven flows, shorter for synchronous machine-to-machine flows).
- The idempotency check happens before the side effect, not after. A check that runs after a database write but before an external call still allows the external call to fire twice.
- Keys are scoped per caller (per API key, per tenant, per user) so one caller's keys cannot collide with another's. The trust model for idempotency keys cross-references [reference/security.md](security.md): treating a caller-supplied key as authoritative without authenticating the caller can let one tenant short-circuit another tenant's response.

Severity is typically P0 for money-movement or inventory endpoints with no idempotency, P1 for other mutating endpoints with no idempotency, and P2 for in-memory-only stores or missing TTL documentation.

## Rate limiting and throttling

Rate limiting protects the service from abusive callers, accidental retry storms, and cost-amplifying requests. Throttling shapes traffic so that one expensive caller cannot starve the rest. Both belong at the application boundary, ideally also reinforced at the gateway.

What to look for:

- Every public endpoint has a documented limit (per IP, per API key, per user, or per tenant — chosen deliberately, not by default). Endpoints with no limit are a finding when they are reachable from the internet.
- The chosen algorithm fits the traffic shape: token bucket for bursty interactive traffic, leaky bucket for steady throughput, fixed window for simple quotas, sliding window for fairness across boundary moments.
- Limit state is shared across application instances. A per-instance counter on a horizontally scaled service multiplies the effective limit by the number of replicas and is a finding.
- The service responds with `429 Too Many Requests`, includes `Retry-After`, and exposes `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` (or RFC 9110 `RateLimit-*` equivalents) so clients can back off intelligently.
- Authenticated callers and anonymous callers have different limits; authentication failures count against the anonymous bucket so credential-stuffing cannot be amortized across attempts. This intersects directly with [reference/security.md](security.md) under broken access control and authentication.
- Expensive endpoints (search, export, report generation, large list endpoints) carry tighter limits or a separate cost-weighted bucket. A single weight-1 limit applied uniformly across cheap and expensive endpoints is a finding.
- Limits are observable: counters and rejection reasons are emitted as metrics so operators can tell the difference between "limit too low" and "limit working correctly".

Severity is typically P1 for unlimited public endpoints, P1 for per-instance counters on multi-replica deployments, and P2 for missing `Retry-After` or rate-limit headers.

## GraphQL-specific checks

GraphQL is loaded only when `spring-boot-starter-graphql`, `graphql-java`, or an equivalent dependency is present.

GraphQL changes the abuse model compared to REST: a single request can fan out into thousands of resolver invocations. The contract is the schema, and the schema is executable, so introspection and query shape must be governed.

What to look for:

- **Depth limits** — the server caps the maximum depth of any query. Without a depth cap, a recursive schema (e.g. `User { friends: [User] }`) lets one request traverse the social graph until the server runs out of memory or time. A finding is "no configured depth limiter" or "depth limit set to a value that allows the worst recursive type to explode."
- **Complexity limits** — every field has an estimated cost, and the server rejects queries whose total cost exceeds a budget. A naive depth limit is not enough on its own because a shallow query that selects 10 000 items at depth 2 is still expensive. A finding is "no complexity scoring" or "complexity scoring that ignores list multipliers."
- **Introspection exposure** — `__schema` and `__type` are extremely useful in development and dangerous in production because they reveal the entire attack surface. Production environments should disable introspection or restrict it to authenticated internal callers; a finding is "introspection enabled for anonymous callers in production."
- **Batching attacks** — many GraphQL servers accept arrays of operations in a single HTTP request. Without per-operation accounting, an attacker submits N expensive operations and only pays the cost of one HTTP request, multiplying their effective rate. A finding is "batching enabled with no per-operation rate-limit accounting."
- **Field-level authorization** — authorization decisions happen on resolvers, not only on operations. A query that selects `user.email` for another user is an authorization decision per field. Field-level checks defer to [reference/security.md](security.md) for the underlying authorization review depth.
- **Persisted queries / allowlisting** — production traffic uses persisted queries or an allowlist so the server is not exposed to arbitrary client-supplied query shapes. Absence is a P2 finding for internal services and a P1 finding for public APIs.

Severity is typically P0 for introspection exposed to anonymous internet callers in production, P1 for missing depth or complexity limits, and P1 for batching with no per-operation accounting.

## gRPC-specific checks

This guidance applies only when gRPC is present in the target codebase.

Signals that gRPC is present include `grpc-spring-boot-starter`, `io.grpc:*` dependencies, `*.proto` files under `src/main/proto/`, generated stub classes, `@GrpcService` annotations, or a gRPC server port configured in `application.yml`. If none of those are present, skip this section.

What to look for:

- **Transport security** — production gRPC servers use TLS. Plaintext gRPC (`usePlaintext()`) outside of a meshed sidecar context is a finding. Cross-references [reference/security.md](security.md) for the underlying transport-security review.
- **Message size limits** — the default 4 MB inbound message size is small, and codebases often raise it. A raised limit without a documented business reason is a finding because oversized messages are the easiest denial-of-service vector against gRPC. Both client and server limits matter.
- **Deadlines and timeouts** — every server method enforces a deadline (or rejects requests without one). A server that accepts deadline-less calls inherits whatever timeout the client chose, which may be infinite. A finding is "no deadline propagation enforcement" or "default infinite deadlines."
- **Streaming abuse** — server-streaming, client-streaming, and bidirectional-streaming methods need explicit limits on stream lifetime, message count per stream, and total bytes per stream. Unbounded streams let one client hold a server resource indefinitely.
- **Reflection exposure** — gRPC reflection is the equivalent of GraphQL introspection. Production services should disable reflection or restrict it to internal callers. A finding is "reflection enabled for external callers in production."
- **Authentication and authorization** — gRPC interceptors carry the authentication chain. Missing or order-sensitive interceptors (e.g. logging before authentication) defer to [reference/security.md](security.md) for review depth.
- **Versioning at the proto level** — proto changes follow the standard rules (field numbers never reused, optional fields added, required fields never removed). A breaking proto change is a finding regardless of how the HTTP contract is versioned because gRPC consumers do not see the URI.
- **Rate limiting** — gRPC requests count against the same throttling budget described in the rate-limiting section above. A service that throttles its REST surface but leaves gRPC unthrottled is a finding.

Severity is typically P0 for plaintext gRPC exposed to untrusted networks, P1 for missing deadlines or unbounded streams, and P1 for reflection exposed to external callers in production.

## Severity guidance

### P0
- Money-movement, inventory, or account-state mutating endpoints with no idempotency control.
- GraphQL introspection enabled for anonymous internet callers in production.
- Plaintext gRPC exposed to untrusted networks.
- Endpoints documented as authenticated in OpenAPI but reachable without authentication in practice (contract / reality mismatch on the security boundary).

### P1
- Public endpoints with no rate limit reachable from the internet.
- Per-instance rate-limit counters on a horizontally scaled deployment.
- Mutating endpoints (non-money) with no idempotency control.
- GraphQL with no depth or complexity limit, or with batching enabled and no per-operation rate-limit accounting.
- gRPC servers with no enforced deadlines, unbounded streams, or reflection exposed to external callers.
- Silent breaking changes inside an existing API version.

### P2
- Inconsistent versioning strategy across controllers.
- OpenAPI document missing operations, declaring `Object` / `Map` bodies, or omitting error responses.
- Missing `Retry-After` or rate-limit response headers.
- Idempotency stores that are in-memory-only or have no documented TTL.
- Missing persisted queries / allowlisting on internal GraphQL services.
- Cost-uniform rate limits applied across cheap and expensive endpoints.

### P3
- Hardening improvements: tighter depth/complexity budgets, more granular per-tenant limits, richer OpenAPI examples, deprecation metadata on long-deprecated endpoints, additional observability on rate-limit rejections.

## Evidence rules

Always include:

- file path
- line number when possible
- why the issue matters
- exact remediation
