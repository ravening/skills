# Code Review

## Maintainability checks

Review for:

- oversized classes or methods
- unclear package boundaries
- business logic inside controllers
- repositories doing orchestration
- DTO and entity mixing
- repeated mapping logic
- weak naming or unclear responsibilities
- broad catch blocks
- null-heavy service contracts
- missing pagination on large list endpoints

## Spring-specific smells

Prioritize these checks:

- field injection instead of constructor injection
- `@Transactional` on private methods
- circular dependency workarounds with `@Lazy`
- `RestTemplate` where newer client patterns would be better
- N+1 query risk
- blocking calls in request threads
- missing timeout and retry strategy for integrations
- direct entity exposure in API contracts
- inconsistent exception translation

## Performance checks

Look for:

- unbounded `findAll()` usage
- repeated database reads that could be cached
- eager loading where lazy would be better, or the reverse
- expensive mapping inside loops
- chatty service-to-service calls
- missing bulk operations

## Testability checks

Review whether:

- controllers have focused tests
- services are testable without heavy mocking
- repositories have integration coverage where needed
- external integrations are isolated with stubs or containers
