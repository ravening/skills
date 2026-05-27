# Data Layer

## Scope

This reference covers Hibernate and JPA depth, schema migration safety, and connection pool sizing for Spring Boot services that use `spring-boot-starter-data-jpa`, raw Hibernate, Flyway, Liquibase, or HikariCP. It does NOT cover the entity-relationship or domain-model diagram itself, generic code review topics, security-only concerns such as SQL injection, or cloud-specific managed-database posture. SQL injection lives in `reference/security.md`. Reactive data access (R2DBC, Spring Data Reactive) and NoSQL stores other than as accessed through JPA are out of scope here.

Cross-references: see [reference/architecture.md](architecture.md) for the entity-relationship and domain-model diagram guidance, [reference/security.md](security.md) for injection and credential exposure, and [reference/cloud-native.md](cloud-native.md) for managed database deployment posture.

## Review areas

Assess these categories:

- `open-in-view` setting and request-scoped session lifetime
- fetch strategies, including `LAZY`, `EAGER`, and `EntityGraph`
- second-level cache correctness
- batch sizes and bulk operations
- dialect-specific issues
- migration safety with Flyway and Liquibase
- HikariCP pool sizing and leak detection

## `open-in-view` setting and its risks

Spring Boot enables `spring.jpa.open-in-view` by default. With it enabled, the persistence context stays open for the full HTTP request, which lets controller-layer code or view rendering trigger lazy loads long after the service layer returned. This hides N+1 queries, blurs the boundary between service and presentation, holds a database connection for the duration of the request, and makes load testing misleading.

Look for:

- whether `spring.jpa.open-in-view` is set explicitly in `application.yml` or `application.properties`, and what the value is
- whether DTOs returned by the service layer still contain unfetched proxies that resolve only during JSON serialization
- whether controllers or response mappers touch lazy associations
- whether the team has chosen to leave the default on with a documented rationale, or whether they have inherited the default unintentionally

What good looks like: `open-in-view` is explicitly disabled, services return fully initialized DTOs, and any required associations are loaded inside a transactional service method via `EntityGraph` or eager fetch joins. What bad looks like: the property is unset (so the default `true` applies), DTO serialization triggers extra queries, and lazy-loading exceptions appear only under load.

Record in the report: the file path of the configuration, the value of `spring.jpa.open-in-view` (or note it is unset), an example controller or DTO that depends on the open session, and the recommended remediation.

## Fetch strategies — LAZY, EAGER, EntityGraph

Fetch strategy choices drive most JPA performance problems. `EAGER` on `@ManyToOne` and `@OneToOne` is the default and is rarely what you want at scale; `EAGER` on `@OneToMany` or `@ManyToMany` is almost always wrong because every load pulls the full collection and Hibernate cannot cleanly join multiple bag collections. `LAZY` is the right default for associations, paired with explicit `EntityGraph`, fetch joins, or projection DTOs at the call sites that actually need the related data.

Look for:

- `@ManyToOne` and `@OneToOne` declared without `fetch = FetchType.LAZY`
- `@OneToMany` or `@ManyToMany` declared with `fetch = FetchType.EAGER`
- N+1 query smells where a repository returns a list and a downstream loop touches an association on each element
- repository methods that use `EntityGraph`, `JOIN FETCH`, or projection interfaces vs. those that rely on lazy traversal
- `MultipleBagFetchException` or `cannot simultaneously fetch multiple bags` errors in logs
- pagination combined with `JOIN FETCH` on a collection, which silently fetches everything into memory before slicing

What good looks like: associations default to `LAZY`, hot read paths use `EntityGraph` or projection DTOs, and pagination uses scalar queries with separate fetches for collections. What bad looks like: pervasive `EAGER`, services that hit the database once per element of a list, and `JOIN FETCH` combined with `Pageable`.

Record in the report: the entity and field, the fetch type, the call site that pays for the choice, the observed or estimated query multiplier, and the recommended fetch strategy with the specific repository or query change required.

## Second-level cache correctness

Hibernate's second-level cache (Ehcache, Caffeine, Infinispan, Hazelcast, or another provider) is correct only when invalidation, transaction isolation, and cluster topology are all aligned. The cache is a shared mutable surface that can serve stale data, mask transactional bugs, and produce surprising results across nodes if any one of those is wrong.

Look for:

- whether `hibernate.cache.use_second_level_cache` and `hibernate.cache.use_query_cache` are enabled, and which provider is configured
- which entities are annotated with `@Cache` or `@Cacheable`, and the `CacheConcurrencyStrategy` chosen (`READ_ONLY`, `NONSTRICT_READ_WRITE`, `READ_WRITE`, `TRANSACTIONAL`)
- whether mutable entities use `READ_ONLY` (a correctness bug) or whether read-mostly reference data uses `READ_WRITE` unnecessarily (a performance cost)
- whether collection caching is enabled on `@OneToMany` and whether the inverse side participates correctly
- whether bulk `UPDATE` or `DELETE` JPQL statements bypass the cache without explicit eviction
- whether the cache is local to each node while the application runs in a multi-instance deployment without a distributed or invalidating provider
- the query cache hit rate and the cost of caching parameterized queries that rarely repeat

What good looks like: caching is opt-in per entity, the concurrency strategy matches the entity's mutability, bulk DML is followed by explicit eviction, and the provider matches the deployment topology. What bad looks like: blanket caching, mismatched concurrency strategies, and silent staleness across nodes.

Record in the report: the entity, the cache region, the concurrency strategy, the deployment topology mismatch if any, and the remediation, including whether to remove caching, change the strategy, or add an explicit eviction call.

## Batch sizes and bulk operations

Hibernate issues one statement per modified entity by default. For inserts, updates, and deletes that touch many rows, this is the single largest source of throughput loss in JPA-backed services. Batch sizing and bulk-statement choice are both required for correctness and performance.

Look for:

- the value of `hibernate.jdbc.batch_size`, typically 20 to 50; an unset value (or 1) defeats batching
- whether `hibernate.order_inserts` and `hibernate.order_updates` are enabled, which is required for batching to actually combine statements when multiple entity types are flushed
- identity-generated primary keys on entities being batch-inserted, which silently disables JDBC batching for inserts because Hibernate has to round-trip per row to obtain the generated id
- service methods that loop over a collection and call `repository.save(entity)` per element without a periodic `flush()` and `clear()`, which grows the persistence context unboundedly
- bulk `UPDATE` or `DELETE` JPQL or native statements that bypass the persistence context, paired with whether the code clears or evicts affected entities afterwards
- `saveAll` usage where `batch_size` is unset, giving the appearance of batching without actual batching at the JDBC layer

What good looks like: an explicit `batch_size`, `order_inserts` and `order_updates` enabled, sequence-based id generation for batch-inserted entities, periodic `flush()` and `clear()` inside long loops, and a documented choice between persistence-context updates and bulk JPQL.

Record in the report: the file path of the offending loop or statement, the configured batch settings, the id-generation strategy, the recommended fix, and the expected throughput impact when measurable.

## Dialect-specific issues

The Hibernate dialect (and the underlying database) shapes correctness, not just SQL flavor. Dialect-specific behavior affects identifier generation, locking, JSON column handling, time-zone semantics, schema-migration syntax, and feature availability such as `MERGE` or upserts.

Look for:

- the configured dialect vs. the actual database engine and version; mismatches cause subtle correctness bugs and unreliable schema generation
- automatic schema management (`spring.jpa.hibernate.ddl-auto`) set to anything other than `none` or `validate` in non-development environments; `update` and `create-drop` in production are P0
- identifier generation choices that interact with the dialect: PostgreSQL sequences vs. MySQL `AUTO_INCREMENT`, with batching implications already noted above
- timestamp and time-zone handling, including columns declared as `TIMESTAMP` vs. `TIMESTAMP WITH TIME ZONE`, JVM default time zone vs. database server time zone, and whether `hibernate.jdbc.time_zone` is set explicitly
- JSON columns: native `JSON` or `JSONB` types backed by a custom `UserType` or `@Type` annotation, vs. serializing JSON into a `TEXT` column, which loses query and index capabilities
- pessimistic locking on databases that handle `SELECT ... FOR UPDATE` differently, including `SKIP LOCKED` availability and behavior
- vendor-specific features such as PostgreSQL partial indexes, MySQL utf8mb4, or SQL Server `IDENTITY` blocks, that appear in entity mappings or migrations

What good looks like: dialect, database version, and time zone are pinned and documented; `ddl-auto` is `validate` or `none` in production with migrations as the source of truth; JSON columns use the right native type; and locking semantics are explicit per query.

Record in the report: the dialect, the actual database engine and version, the specific mismatched setting or annotation, why it matters for that engine, and the exact remediation.

## Migration safety — Flyway and Liquibase

Schema migrations are the highest-risk database change because they run during deploy and can take or hold long-lived locks. Migration safety covers ordering, rollback strategy, lock contention, and the patterns required for online schema change without downtime.

Look for:

- whether the project uses Flyway, Liquibase, or both (using both causes contention for the migration history table; pick one)
- migration files that mix DDL and DML in the same transaction, where the DDL implicitly commits and a partial failure leaves the schema and data inconsistent
- destructive migrations (`DROP COLUMN`, `DROP TABLE`, `ALTER COLUMN` with type narrowing) that are not gated by an expand-then-contract sequence: first add the new column or table, deploy the application that writes to both, backfill, switch reads, then drop the old shape in a later migration
- migrations that rewrite large tables in a single statement on databases that take exclusive locks for the duration (e.g., MySQL on older versions for certain `ALTER TABLE` operations); these block production traffic
- migrations that add a `NOT NULL` column with no default on a populated table, which forces a full table rewrite and rejects existing rows
- creation of indexes without `CONCURRENTLY` on PostgreSQL, or without online options on databases that support them; this blocks writes during index build
- rollback strategy: Flyway has `undo` migrations only on the commercial edition, so the realistic rollback strategy in OSS Flyway is forward-only with a compensating migration; Liquibase supports rollback blocks but they must be authored explicitly and tested
- lock contention on the migration history table itself, especially when multiple application instances start at the same time; check whether one instance owns migration (e.g., a Kubernetes init container, a deploy job, or a leader election) vs. all instances racing
- baseline and out-of-order behavior: `flyway.outOfOrder=true` and missing baseline configuration on legacy databases cause migrations to be skipped or replayed unexpectedly
- whether migration scripts are reviewed for idempotency where the chosen tool requires it, and whether checksums are protected against post-merge edits

What good looks like: one migration tool, expand-then-contract patterns for destructive changes, online schema change for large tables (e.g., `pt-online-schema-change`, `gh-ost`, or native online DDL), a documented rollback strategy with explicit Liquibase rollback blocks or forward-only compensating migrations, and a single owner for migration execution at deploy time.

Record in the report: the migration file path, the specific risky statement, the database engine's locking behavior for that statement, the blast radius (table size and traffic profile), the safe alternative pattern, and the rollback strategy.

## HikariCP — pool sizing and leak detection

HikariCP is the default Spring Boot connection pool. Misconfiguration here usually shows up as request timeouts under load rather than as a clean error, which makes it easy to misdiagnose as application slowness.

Look for:

- `spring.datasource.hikari.maximum-pool-size` set without reference to the database's own connection limit; the sum of pool sizes across all application instances, plus other clients, must be below the database's configured maximum
- pool size set far higher than the number of available CPU cores on the database host; oversized pools degrade rather than improve throughput because the database becomes the bottleneck
- `minimum-idle` set equal to `maximum-pool-size` (effectively disabling pool shrinking) without justification
- `connection-timeout` (default 30 seconds) vs. the application's own request timeouts; a connection-acquire wait that exceeds the upstream timeout produces orphaned work
- `idle-timeout` and `max-lifetime` shorter than the database server's idle timeout and shorter than any infrastructure idle-timeout (load balancer, proxy), so connections are recycled by HikariCP rather than killed mid-query by infrastructure
- `leak-detection-threshold` unset (or zero, meaning disabled); enabling it (commonly 30 to 60 seconds) surfaces connections held open by code paths that forget to close them, including misused `EntityManager` or non-transactional repository access
- `validation-timeout` and `connection-test-query` aligned with the JDBC driver's capabilities (`isValid` is preferred for modern drivers)
- transactional boundaries that hold a database connection across a remote HTTP call or a long computation, which inflates pool pressure regardless of pool size
- read-only transactions vs. write transactions sharing the same pool, vs. having separate pools for read replicas
- pool metrics exposed via Micrometer and whether dashboards show `hikaricp_connections_active`, `hikaricp_connections_pending`, and acquisition timing

What good looks like: pool size derived from a measured concurrency budget rather than a guess; lifetime and timeouts shorter than every upstream idle timeout; leak detection enabled in non-production at minimum; transactional methods that do not call out over the network; and pool metrics flowing to dashboards.

Record in the report: the configuration file path, the offending property values, the database server limit and the application's instance count, the observed symptom (timeout, exhaustion, leak), and the recommended values with reasoning.

## Severity guidance

### P0
- destructive migration (`DROP`, type-narrowing `ALTER`) in a single forward step against a populated production table with no expand-then-contract sequence
- `spring.jpa.hibernate.ddl-auto` set to `update`, `create`, or `create-drop` in a production profile
- second-level cache concurrency strategy that is unsafe for the entity's mutability (e.g., `READ_ONLY` on a mutable entity), causing visible stale or wrong data
- HikariCP pool size, summed across instances, exceeds the database's `max_connections` and is causing connection refusals
- migration script combining DDL and DML in a way that leaves the schema and data inconsistent on partial failure

### P1
- pervasive `EAGER` on collection associations or N+1 patterns on hot read paths
- `open-in-view` enabled (default) with controllers or serialization triggering lazy loads
- `NOT NULL` column added without default on a large populated table, blocking deploy or causing extended downtime
- index creation that takes an exclusive lock on a high-traffic table (no `CONCURRENTLY` on PostgreSQL or no online option where supported)
- transactional method holding a database connection across a remote HTTP call
- HikariCP `max-lifetime` longer than an upstream infrastructure idle timeout, causing intermittent broken-pipe errors
- batch operation loop using `save` per element with `batch_size` unset and no periodic `flush`/`clear`

### P2
- missing `EntityGraph` or projection on a frequently queried read path that currently relies on lazy loading
- second-level cache enabled broadly without per-entity rationale
- `hibernate.jdbc.batch_size` unset where bulk inserts or updates exist
- dialect or time-zone configuration not pinned, with no observed bug yet
- `leak-detection-threshold` unset
- both Flyway and Liquibase configured, only one actually used

### P3
- documentation gaps around migration ownership and rollback strategy
- pool metrics not surfaced in dashboards
- minor naming or convention inconsistencies in migration files
- ergonomic improvements such as switching from `save` loops to `saveAll` paired with a configured `batch_size`

## Evidence rules

Always include:

- file path
- line number when possible
- why the issue matters
- exact remediation
