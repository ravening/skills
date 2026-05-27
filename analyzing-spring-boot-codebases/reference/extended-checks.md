# Extended Checks

Use these when the user asks for a deeper review or when the codebase includes the relevant technology.

## Optional deep checks

- Flyway or Liquibase migration quality
- Dockerfile security and non-root runtime
- Kubernetes readiness and liveness support
- graceful shutdown behavior
- Resilience4j retries, circuit breakers, and timeouts
- async executor configuration
- scheduler safety and idempotency
- OpenAPI completeness
- Testcontainers and integration-test maturity
- observability via metrics, tracing, and structured logs
- concurrency risks from mutable singleton state

## Escalation rule

If any extended check reveals a production-risk flaw, promote it into the main findings list with normal severity.
