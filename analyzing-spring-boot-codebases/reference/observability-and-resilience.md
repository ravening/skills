# Observability and Resilience Review

## Scope

This file covers fault tolerance, distributed tracing, metrics, structured logging, and SLO posture for a Spring Boot service. It does NOT cover container or Kubernetes posture (see [reference/cloud-native.md](cloud-native.md)), API surface review (see [reference/api-and-contracts.md](api-and-contracts.md)), or general code maintainability (see [reference/code-review.md](code-review.md)).

Cross-references: [reference/architecture.md](architecture.md) for the async / event flow diagram that visualizes the boundaries discussed here.

## Review areas

Assess these categories:

- Resilience4j configuration and use — retry, circuit breaker, bulkhead, rate limiter, time limiter
- Structured logging and MDC propagation
- OpenTelemetry and Micrometer setup for tracing and metrics
- SLO and SLI definition and measurement
- Distributed tracing across asynchronous boundaries — `@Async`, executors, Kafka and Rabbit listeners, scheduled jobs

## Resilience4j depth

Look at how the application protects itself against failures of the things it calls — other services, databases, message brokers, third-party APIs. Resilience4j is the de facto choice in Spring Boot, and each pattern has a distinct purpose. Confusing them or stacking them in the wrong order is a common source of incidents.

- **Retry**: only safe for idempotent operations. Check that the retry policy excludes non-retryable exceptions (e.g. `4xx` client errors, validation failures) and uses exponential backoff with jitter rather than a tight loop. Retrying a non-idempotent `POST` on a transient timeout can double-charge customers or duplicate orders. Confirm that retries are bounded — a missing `maxAttempts` or a retry policy that retries forever is a P0 outage waiting to happen.
- **Circuit breaker**: protects the caller from a slow or failing downstream. Check that thresholds (`failureRateThreshold`, `slowCallRateThreshold`, `slowCallDurationThreshold`) are tuned to the downstream's real latency budget rather than left at defaults. A wide-open circuit breaker (e.g. 99% failure threshold) provides no protection. A circuit breaker without a defined fallback (`@CircuitBreaker(fallbackMethod = ...)`) just rethrows, which still cascades failure.
- **Bulkhead**: limits concurrent calls to a downstream so one slow dependency does not consume all worker threads. Check that bulkhead capacity is set per dependency, not per service, and that the bulkhead type (semaphore vs. thread-pool) matches the call style (blocking vs. async). A missing bulkhead in front of a slow third-party API is the classic cause of full-thread-pool outages.
- **Rate limiter**: caps the rate at which the application *makes* outbound calls (distinct from API throttling, which caps inbound traffic — see [reference/api-and-contracts.md](api-and-contracts.md)). Use it to respect a third-party's documented quota and avoid being banned. Check that the limit matches the contract with the downstream, not an arbitrary number.
- **Time limiter**: caps how long the application waits for a single call. Without a time limiter, a hung downstream can pin a thread indefinitely. Time limiter values must be shorter than the upstream HTTP client's read timeout, otherwise the time limiter never fires.

When several patterns wrap the same call, order matters. The conventional outer-to-inner order is bulkhead → time limiter → circuit breaker → retry. Retries inside a circuit breaker count as one logical call from the breaker's perspective; retries outside the breaker can defeat it. Flag any configuration where the order is reversed or where decorators are applied inconsistently across similar call sites.

Findings here typically map to P0 (no protection on a critical outbound call, retries on non-idempotent operations) through P2 (defaults left in place, missing fallbacks).

## Structured logging and MDC

Plain-text log lines are fine for a single instance, but a Spring Boot service in production needs structured logs that downstream tooling can index. Look for a JSON encoder (e.g. Logback's `LogstashEncoder`, `logstash-logback-encoder`, or an OpenTelemetry log exporter) and confirm that log fields include at least: timestamp, level, logger, thread, message, exception, service name, environment, and correlation IDs.

MDC (Mapped Diagnostic Context) is the mechanism for attaching per-request context — `traceId`, `spanId`, `userId`, `tenantId`, `requestId` — to every log line emitted while handling that request. Verify:

- An inbound filter or interceptor populates MDC at the start of each request (or that Micrometer / OpenTelemetry auto-instrumentation does it for you).
- MDC is cleared at the end of the request. A leak here causes one user's correlation ID to appear in another user's logs — a privacy and debuggability disaster.
- MDC propagates across thread boundaries. Plain `ExecutorService` does not propagate MDC. Look for `MdcThreadPoolTaskExecutor`, `TaskDecorator`-based propagation on `ThreadPoolTaskExecutor`, or use of `ContextSnapshot` from Micrometer Context Propagation. Without this, `@Async` work logs without correlation IDs and traces become unjoinable.
- Sensitive data (passwords, tokens, full PII) is not logged. Check log statements around auth filters, request bodies, and exception handlers — a stack trace that includes the full request body often leaks secrets. This is also a security concern; cross-link to [reference/security.md](security.md).

Findings here typically map to P1 (missing correlation IDs in production, MDC leak across requests) through P3 (inconsistent field names across log statements).

## OpenTelemetry and Micrometer

Micrometer is the metrics facade. OpenTelemetry is the tracing and (increasingly) metrics standard. In a modern Spring Boot service, both are usually present: Micrometer for metrics, OpenTelemetry (or the older Spring Cloud Sleuth on legacy services) for traces.

For metrics, check that:

- `spring-boot-starter-actuator` is present and `/actuator/prometheus` (or whichever scrape endpoint is used) is reachable from the metrics collector but not exposed publicly. Cross-link to [reference/security.md](security.md) for actuator exposure.
- The standard Spring Boot meters (HTTP server requests, JVM, JDBC pool, Tomcat, Hikari, Kafka client, etc.) are enabled and tagged consistently. Look for a `MeterFilter` or `MeterRegistryCustomizer` that adds common tags (service, environment, version) and removes high-cardinality tags (raw URI paths, user IDs).
- Custom business metrics use `Counter`, `Timer`, `Gauge`, or `DistributionSummary` — not log scraping. Each business metric should have a clear, stable name and a bounded tag set. Unbounded tag cardinality (e.g. tagging by `userId`) is a P1 finding because it inflates the metrics backend cost and can crash collectors.

For traces, check that:

- The OpenTelemetry agent or SDK is configured with a real exporter (OTLP, Jaeger, Zipkin) pointing at the collector for the target environment. A configured exporter with no receiver is silent failure.
- Sampling is intentional. `parentbased_always_on` in production fills the backend; `parentbased_traceidratio` with a low ratio loses signal during incidents. Confirm the sampling decision is documented.
- Spans for outbound calls (HTTP client, database, message broker) are created automatically by instrumentation. If the application uses a non-instrumented client (e.g. raw `HttpURLConnection` or a custom Kafka producer), spans are missing and the trace appears to end at the controller. Flag these gaps.
- Trace context propagates on outbound HTTP, JDBC, and messaging. The W3C `traceparent` header should appear on outbound HTTP; for messaging, propagation usually rides in headers (`b3-*` or `traceparent`).

Findings here typically map to P1 (no metrics exporter, no trace exporter, unbounded tag cardinality) through P3 (default sampler in non-critical services).

## SLO and SLI guidance

An SLI is the thing measured (e.g. "fraction of `POST /orders` responses returned in under 300 ms with a 2xx status"). An SLO is the objective for that SLI over a window (e.g. "99.5% over rolling 30 days"). Look for evidence the team has defined these, even informally — a runbook, a dashboard with thresholds, an alerting rule tied to an error budget.

Practical checks:

- For each user-facing endpoint or critical background flow, is there at least a latency SLI and an availability SLI? "Latency under X for Y%" and "successful response for Z%" are the minimum useful pair.
- Are SLIs measured from the user's perspective (e.g. at the ingress or load balancer) or only inside the service? Internal-only SLIs miss network and proxy failures and overstate availability.
- Are alerts tied to SLO burn rate (fast burn, slow burn) rather than raw thresholds? A static "alert when error rate > 1%" fires on every minor blip; a burn-rate alert ("at this rate we'll exhaust the 30-day error budget in 6 hours") is actionable.
- Is there an error budget policy — what does the team do when the budget is exhausted? Even a sentence in the runbook ("freeze new feature work until burn rate drops") is useful.

If SLOs are entirely absent, that is a P2 finding with a clear remediation: define one latency SLI and one availability SLI per critical flow, set initial targets, and wire them to the existing metrics. If SLOs exist but are clearly aspirational (the service routinely violates them with no response), that is also P2 — the issue is the missing error-budget policy, not the numbers.

## Distributed tracing across asynchronous boundaries

The most common observability gap in Spring Boot services is the trace ending the moment work crosses a thread boundary. Trace context lives in a thread-local; once you submit work to an executor, schedule a job, or hand a message to a broker, that thread-local does not follow. Without explicit propagation, `@Async` methods, scheduled jobs, and message listeners log and trace as if they were unrelated requests.

What to verify, by boundary:

- **`@Async` methods and `ThreadPoolTaskExecutor`**: confirm a `TaskDecorator` is configured on the executor that captures and restores the MDC and the OpenTelemetry context. Spring Cloud Sleuth used to do this automatically; with OpenTelemetry, this is often `ContextSnapshot.captureAll()` wrapped via a `TaskDecorator`. A bare `Executors.newFixedThreadPool(...)` used as an `@Async` executor will lose context every time.
- **Custom `ExecutorService`**: same issue as above. If the team uses a hand-rolled executor for parallel work, check that submitted tasks wrap themselves in a context-restoring `Runnable` / `Callable` (Micrometer's `ContextExecutorService` is the canonical wrapper).
- **`@Scheduled` jobs**: scheduled jobs start a fresh trace by definition (there is no parent). Confirm that a new trace is created intentionally (so the work is observable) and that the job logs include a stable identifier (job name, run ID) so its logs can be correlated even without an inbound traceparent.
- **Kafka listeners (`@KafkaListener`)**: trace context should be propagated as Kafka record headers on the producer side and extracted on the consumer side. Spring Kafka with OpenTelemetry instrumentation does this automatically; a hand-rolled producer or consumer that does not touch headers breaks the trace. Verify producers set `traceparent` (or `b3-*`) headers and consumers extract them before invoking the listener method.
- **Rabbit listeners (`@RabbitListener`)**: same model — context rides in AMQP headers. Verify producer and consumer instrumentation matches.
- **Reactive code (`Mono` / `Flux`)**: reactive context propagation is its own concern. Confirm the application uses Reactor's `Context` or Micrometer Context Propagation hooks (`Hooks.enableAutomaticContextPropagation()`) rather than relying on thread-locals, which do not work reliably in a reactive pipeline.

Cross-reference: when these boundaries are present in the target codebase, the async / event flow diagram in [reference/architecture.md](architecture.md) should show every hop, and each hop should be matched by a span in the trace backend. If the diagram has hops the trace backend does not show, that is a concrete observability gap to record.

Findings here typically map to P1 (broken trace propagation across `@Async` or message boundaries in a critical flow) through P3 (missing job-name field on `@Scheduled` log lines).

## Severity guidance

### P0
- no resilience protection (no circuit breaker, no time limiter) on a critical outbound call known to fail
- retries enabled on non-idempotent operations causing confirmed duplicate side effects
- unbounded retry policy in production

### P1
- circuit breaker thresholds left at defaults on a real production dependency
- missing bulkhead in front of a slow third-party API that has caused thread-pool exhaustion
- no metrics exporter or no trace exporter configured in production
- broken trace context propagation across `@Async`, Kafka, or Rabbit boundaries on a critical flow
- unbounded metric tag cardinality
- MDC leaking across requests
- missing correlation IDs in production logs

### P2
- patterns applied in the wrong order (e.g. retry outside circuit breaker)
- missing fallback method on `@CircuitBreaker`
- no SLOs defined for user-facing endpoints
- SLOs defined but no error-budget policy
- internal-only SLIs that miss ingress failures
- default sampling in production without justification

### P3
- inconsistent log field names across modules
- missing job-name or run-ID fields on `@Scheduled` log lines
- minor tag-naming inconsistencies in metrics
- aspirational SLOs that need recalibration

## Evidence rules

Always include:

- file path
- line number when possible
- why the issue matters
- exact remediation
