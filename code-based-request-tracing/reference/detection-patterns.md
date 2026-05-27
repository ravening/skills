---
name: detection-patterns
description: Inbound endpoint and outbound call detection rules used to populate the Tracing_Graph.
---

# Detection Patterns

This file specifies the static-analysis signals the Agent uses to populate the Tracing_Graph for each Service_Repo. It has two halves:

- **Inbound endpoints** — the minimum the Agent must extract to seed Trace entry points. Defers to the peer skill for the full controller/mapping catalog.
- **Outbound calls** — owned wholly by this skill: the Spring Boot scope check, the four detection kinds plus `kafka-template` outbound publishes, the per-call recorded fields, and the false-positive guidance for filtering test-code lookalikes and dead configuration.

The Agent applies these patterns only after confirming the Service_Repo is in scope (see [Spring Boot scope check](#spring-boot-scope-check)). Out-of-scope repos are marked `springBoot: false` and contribute zero endpoints and zero calls to the graph.

## Inbound endpoints

### Cross-reference (do not restate)

The full catalog of Spring Boot controller and mapping patterns — including documentation conventions, contract review, and adjacent topics like idempotency and rate limiting — lives in the peer skill at [`../../analyzing-spring-boot-codebases/reference/api-and-contracts.md`](../../analyzing-spring-boot-codebases/reference/api-and-contracts.md). Layered-architecture vocabulary (controller / service / repository) is in [`../../analyzing-spring-boot-codebases/reference/architecture.md`](../../analyzing-spring-boot-codebases/reference/architecture.md).

This skill **does not** duplicate that catalog. It lists only the minimum the Agent must extract to anchor traces.

### Class-level annotations to scan

The Agent recognizes a Java class as an inbound controller when **any** of the following holds:

- The class is annotated with `@RestController`.
- The class is annotated with `@Controller` **and** either the class itself or the individual handler method carries `@ResponseBody`.

Any other class — including `@Service`, `@Component`, `@Configuration`, and `@RestControllerAdvice` — is **not** treated as a source of Inbound_Endpoints for tracing purposes. (Exception handlers in `@RestControllerAdvice` do not represent inbound entry points; they are skipped here even though the peer skill discusses them.)

### Mapping annotations to recognize

On a recognized controller class, the Agent extracts one Inbound_Endpoint per handler method that carries any of:

- `@RequestMapping`
- `@GetMapping`
- `@PostMapping`
- `@PutMapping`
- `@DeleteMapping`
- `@PatchMapping`

Class-level `@RequestMapping` (or one of the typed shortcuts at the class level, which is uncommon but legal) contributes the class path prefix to every method on that class.

### Path concatenation rule

The Agent forms the full Inbound_Endpoint path by concatenating the class-level path with the method-level path and normalizing the result:

```text
fullPath = normalize("/" + classPath + "/" + methodPath)
```

`normalize` collapses repeated slashes, ensures a single leading `/`, and strips a trailing `/` unless the path is exactly `"/"`. If either segment is absent or empty, treat it as the empty string before concatenation. If both are absent, the path is `"/"`.

`{var}` placeholders (e.g. `/users/{id}`) are preserved verbatim in the recorded path. They are used later by `findEndpointMatch` (see `trace-construction.md`) to match outbound URLs that resolve to literal segments at the same positions.

If a mapping annotation declares multiple paths (e.g. `@RequestMapping(path = { "/foo", "/bar" })`), the Agent records one Inbound_Endpoint per declared path.

If the resulting path is the empty string, treat it as `"/"` and emit a warning in the evidence table.

### HTTP method extraction

The HTTP method is derived as follows:

- For the typed shortcuts (`@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`), the method is fixed by the annotation type: `GET`, `POST`, `PUT`, `DELETE`, `PATCH` respectively.
- For `@RequestMapping`, the method is taken from the `method = RequestMethod.X` attribute. If the attribute lists multiple methods (e.g. `method = { RequestMethod.GET, RequestMethod.POST }`), the Agent records one Inbound_Endpoint per method.
- For `@RequestMapping` **without** a `method` attribute, Spring's runtime behavior is "match any HTTP method." Tracing assumes `GET` as the default and **flags this as a warning** in the evidence row (`Marker: default-method-assumed-get`). The user can correct the assumption with a follow-up annotation review.

### Recorded fields per Inbound_Endpoint

Every detected Inbound_Endpoint contributes one row to `tracing-graph.json#/endpoints[]` with exactly these five fields, plus the synthetic `id` used internally by the graph:

| Field | Type | Description |
|---|---|---|
| `serviceName` | string | The owning Service's resolved name (after conflict disambiguation). |
| `httpMethod` | enum `HttpMethod` | One of `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `OPTIONS`, `HEAD`, `TRACE`. |
| `path` | string | Normalized full path (leading slash, no trailing slash, `{var}` preserved). |
| `evidenceFile` | string | Repo-relative path to the `.java` file containing the handler method, forward slashes only. |
| `evidenceLine` | integer | 1-based line number of the mapping annotation (or the method declaration if the annotation spans multiple lines — pick the line of the annotation's `@` token). |

The synthetic `id` is stable across runs given the same `(serviceName, httpMethod, path)` tuple, so reruns produce identical references in `traces[]` and the evidence table.

### What this section does not cover

- The broader review concerns layered on top of inbound endpoints (OpenAPI completeness, versioning strategy, idempotency, rate limiting, GraphQL/gRPC specifics) belong to the peer skill's [`api-and-contracts.md`](../../analyzing-spring-boot-codebases/reference/api-and-contracts.md). This skill does not re-evaluate them.
- Servlet filters, `HandlerInterceptor`s, `WebFilter`s, and routing functions (`RouterFunction<ServerResponse>` from Spring WebFlux) are **not** treated as Inbound_Endpoints in v0.1. If a service exposes its surface only through `RouterFunction`s, that surface will be missed; record this as a known limitation in the run metadata.

## Outbound calls

Outbound detection is owned wholly by this skill. The Agent runs the [Spring Boot scope check](#spring-boot-scope-check) **first** for every Service_Repo; only repos that pass the check contribute Inbound_Endpoints and Outbound_Calls to the graph. The Agent then restricts the [file scope](#file-scope-java-only) to `.java` source files, recognizes the [four detection kinds](#detection-kinds) plus `kafka-template` outbound publishes, records the [per-call fields](#recorded-fields-per-outbound_call) for each detected call, and finally applies the [false-positive guidance](#false-positive-guidance) to filter out lookalikes that would otherwise inflate the call count.

### Spring Boot scope check

Before any endpoint or call detection runs, the Agent must determine whether a Service_Repo is a Spring Boot project. The check inspects build descriptors at the repo root (and, where applicable, in the parent or aggregator `pom.xml` referenced by a `<parent>` element):

- **Maven (`pom.xml`)** — pass if any of the following holds:
  - The `<parent>` element is `org.springframework.boot:spring-boot-starter-parent` (any version).
  - A `<dependency>` entry has `groupId` `org.springframework.boot` and `artifactId` starting with `spring-boot-starter` (e.g. `spring-boot-starter-web`, `spring-boot-starter-webflux`).
  - The `<plugin>` `org.springframework.boot:spring-boot-maven-plugin` is declared in `<build><plugins>` or `<pluginManagement>`.
- **Gradle Groovy (`build.gradle`)** — pass if any of the following holds:
  - The `plugins { ... }` block declares `id 'org.springframework.boot'` (or `id "org.springframework.boot"`).
  - `apply plugin: 'org.springframework.boot'` appears at the top level.
  - The `dependencies { ... }` block contains an entry whose coordinate matches `org.springframework.boot:spring-boot-starter*`.
- **Gradle Kotlin (`build.gradle.kts`)** — pass if any of the following holds:
  - The `plugins { ... }` block declares `id("org.springframework.boot")`.
  - The `dependencies { ... }` block contains an entry whose coordinate matches `org.springframework.boot:spring-boot-starter*` (any starter, including `implementation("org.springframework.boot:spring-boot-starter-web")`).

If **none** of these signals is present, the Service_Repo is **not** a Spring Boot project. The Agent records the corresponding `services[]` entry with `springBoot: false` and a `nonSpringBootReason` populated from the check (e.g. `"no spring-boot starter or plugin found in pom.xml/build.gradle/build.gradle.kts"`), then **skips** both Inbound_Endpoint and Outbound_Call detection for that service. Such services contribute zero rows to `endpoints[]` and zero rows to `calls[]`. They still appear as nodes in the component diagram, rendered with the non-spring-boot marker, so the user can see which repos were excluded.

If a repo contains **multiple** Spring Boot modules (a multi-module Maven or Gradle build), each module is treated as a separate Service_Repo for scope-check purposes; modules that don't pass the check are skipped, and modules that do pass are scanned independently and named per the resolution rules in [`multi-repo-input.md`](./multi-repo-input.md).

### File scope (.java only)

Within an in-scope Service_Repo, the Agent scans **only files with the `.java` extension**. The following are explicitly **not** scanned for endpoints or calls:

- `.kt`, `.kts`, `.scala`, `.groovy` — Kotlin / Scala / Groovy sources are out of scope for v0.1, even when they are Spring Boot Kotlin services. Kotlin support is a known limitation and is recorded in run metadata when any non-`.java` source files are observed in `src/main/`.
- `.xml` — including `pom.xml`, Spring XML configuration, MyBatis mappers, and any other XML. (`pom.xml` is read by the scope check above and by build-time inspections only.)
- `application.yml`, `application.yaml`, `application.properties` — read **only** by the property-resolution pass described in [`target-resolution.md`](./target-resolution.md), never as a source of endpoints or calls.
- Any file under `src/test/`, `src/integrationTest/`, or any directory whose path segment is `test`, `tests`, or `it`. Test sources frequently contain controller and `RestTemplate`/`WebClient` lookalikes that must not pollute the graph; see [False-positive guidance](#false-positive-guidance).
- Generated sources under `target/`, `build/`, `out/`, or `generated-sources/` directories. These are byproducts of the build and would double-count anything present in `src/main/`.

### Detection kinds

The Agent recognizes five kinds of outbound activity. The first four are listed in the requirements (`rest-template`, `web-client`, `feign-client`, `kafka-listener`); the fifth, `kafka-template`, is the outbound counterpart to `kafka-listener` and represents publishing to a Kafka topic.

| Kind | Detection signal | Captured fields |
|---|---|---|
| `rest-template` | Method invocations on a `RestTemplate` field/var: `getForObject`, `getForEntity`, `postForObject`, `postForEntity`, `put`, `delete`, `exchange`, `execute` | URL string argument (or first `String`/`URI` argument), HTTP method (from method name; for `exchange` / `execute`, from the `HttpMethod` argument) |
| `web-client` | Builder chains rooted on a `WebClient` reference: `.get()`, `.post()`, `.put()`, `.delete()`, `.patch()`, `.method(HttpMethod.X)`, followed by `.uri(...)` | URI argument (string literal, format string, or `UriBuilder` lambda — recorded as written), HTTP method from the verb call or `method(...)` |
| `feign-client` | Interfaces annotated with `@FeignClient(name=, url=, path=)`; per-method mapping annotations (`@GetMapping`, `@PostMapping`, `@RequestMapping`, etc.) on each abstract method | `name`, `url`, `path`, plus per-method `(httpMethod, methodPath)` from the mapping annotation |
| `kafka-listener` | Methods annotated with `@KafkaListener(topics=, topicPattern=, groupId=)` | Topics list (or pattern), group id |
| `kafka-template` | Method invocations on a `KafkaTemplate` field/var: `send`, `sendDefault`, `executeInTransaction` | Topic argument (or default topic from configured property), key argument when literal |

Per-kind detection notes:

- **`rest-template`** — The Agent recognizes a field or local variable as a `RestTemplate` when its declared type is `org.springframework.web.client.RestTemplate`, or when it is injected via `@Autowired` / constructor parameter / `@Bean` factory method whose return type is `RestTemplate`. URL extraction follows the call:
  - For `getForObject`, `getForEntity`, `postForObject`, `postForEntity`, `put`, `delete` — the first argument is the URL.
  - For `exchange(url, HttpMethod.X, ...)` and `execute(url, HttpMethod.X, ...)` — the first argument is the URL and the second is the HTTP method.
  - The HTTP method maps as: `getForObject`/`getForEntity` → `GET`, `postForObject`/`postForEntity` → `POST`, `put` → `PUT`, `delete` → `DELETE`, `exchange`/`execute` → from the `HttpMethod` argument.

- **`web-client`** — The Agent recognizes a reference as a `WebClient` when its declared or inferred type is `org.springframework.web.reactive.function.client.WebClient` (typically constructed via `WebClient.create(...)`, `WebClient.builder()...build()`, or injected as a field). The detection requires both a verb call and a `.uri(...)` call in the same fluent chain; chains that build a `WebClient` via `.builder()` but never invoke `.uri(...)` (e.g. configuration beans that expose a preconfigured `WebClient`) are not call sites and are skipped. The HTTP method comes from the verb method name, except for `.method(HttpMethod.X)` where it comes from the `HttpMethod` argument.

- **`feign-client`** — The Agent recognizes interfaces annotated with `org.springframework.cloud.openfeign.FeignClient`. Each abstract method on the interface that carries a Spring mapping annotation contributes one Outbound_Call row. The captured `targetRaw` is the concatenation of the `@FeignClient(url = ...)` (or `name = ...` if `url` is absent), the `@FeignClient(path = ...)` if present, and the method-level mapping path; resolution follows the rules in [`target-resolution.md`](./target-resolution.md). The HTTP method comes from the method-level annotation type or `@RequestMapping(method = ...)`. Per-method evidence points to the method declaration (line of the mapping annotation's `@` token).

- **`kafka-listener`** — The Agent records one Outbound_Call row per `@KafkaListener` method, even though the call is structurally an *inbound* event consumption rather than an outbound HTTP call. This is a deliberate modeling choice: in the Tracing_Graph, Kafka topics are nodes and the listener edge represents the consumer relationship, parallel to the producer edge from `kafka-template`. The captured `targetRaw` is the `topics` list (joined with `,`) or the `topicPattern` value; `httpMethod` is `null`; `kafkaTopics[].consumers` is updated with the calling service name.

- **`kafka-template`** — The Agent recognizes a reference as a `KafkaTemplate` when its declared type is `org.springframework.kafka.core.KafkaTemplate<...>` (any type parameters). Each `send(...)` or `sendDefault(...)` invocation contributes one Outbound_Call row with `kind: kafka-template`, `httpMethod: null`, and `targetRaw` set to the topic argument (the first `String` argument for `send(topic, ...)`, or the configured default topic name resolved via `target-resolution.md` for `sendDefault(...)`). `kafkaTopics[].producers` is updated with the calling service name.

### Recorded fields per Outbound_Call

Every detected Outbound_Call contributes one row to `tracing-graph.json#/calls[]`. The fields recorded **at detection time** (before target resolution and host-mapping application) are:

| Field | Type | Description |
|---|---|---|
| `callerServiceName` | string | The owning Service's resolved name (after conflict disambiguation), matching the `services[].name` of the repo containing the call. |
| `kind` | enum `CallKind` | One of `rest-template`, `web-client`, `feign-client`, `kafka-listener`, `kafka-template`. |
| `targetRaw` | string | The verbatim source expression for the target — URL string, `@FeignClient(url=...)` value, `topics={...}` list, etc. — preserved as written, including any `${prop}` or constant references. |
| `httpMethod` | enum `HttpMethod` \| null | One of `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `OPTIONS`, `HEAD`, `TRACE` for HTTP kinds. `null` for `kafka-listener` and `kafka-template`. |
| `evidenceFile` | string | Repo-relative path to the `.java` file containing the call site or `@FeignClient` interface, forward slashes only. |
| `evidenceLine` | integer | 1-based line number of the call invocation, the mapping annotation, or the `@KafkaListener` / `@FeignClient` annotation (the line of the annotation's or method's `@` / first token). |

The fields `targetResolved`, `targetEndpointId`, `status`, and `mappingApplied` are populated by later passes ([`target-resolution.md`](./target-resolution.md) and host-mapping application) and are not part of detection itself. The synthetic `id` is assigned at serialization and is stable across runs given the same `(callerServiceName, kind, evidenceFile, evidenceLine)` tuple.

### False-positive guidance

Outbound detection is heuristic; the patterns above match common real-world signatures, but a few constructs look enough like outbound calls to slip through. The Agent must filter out these lookalikes before emitting `calls[]` rows.

- **Test-code lookalikes.** Test classes (under `src/test/`, `src/integrationTest/`, or any directory whose path segment is `test`, `tests`, or `it`) frequently contain `RestTemplate`, `WebClient`, and `KafkaTemplate` usages — typically through `TestRestTemplate`, `WebTestClient`, or embedded-broker fixtures — that exercise the service rather than represent outbound traffic. The Agent skips these per the [file scope](#file-scope-java-only) rule. In addition, even when scanning `src/main/`, the Agent **excludes** invocations on these test-only types: `org.springframework.boot.test.web.client.TestRestTemplate`, `org.springframework.test.web.reactive.server.WebTestClient`, and any class in a package matching `*.test.*` or `*.testsupport.*`. If such a class appears inside `src/main/` (an anti-pattern but observed in some repos), it is recorded under the run's `notes` field rather than as a call.

- **Unused `WebClient` builders.** A `WebClient.builder()...build()` chain that is **only** assigned to a bean or field, with no subsequent `.get()`/`.post()`/.../`.uri(...)` chain invoked anywhere in the same `.java` file or in any other `.java` file that imports the bean, is **not** a call site. Such builders are commonly exposed as `@Bean` factories that downstream code never actually consumes (dead configuration). The Agent records a call only when both a verb method *and* `.uri(...)` are observed in the same fluent chain. Builder chains that produce a `WebClient` instance without an immediate request invocation are skipped silently (no row, no warning).

- **Unused `@FeignClient` interfaces.** A `@FeignClient` interface that is never injected (no `@Autowired`, no constructor parameter, no field declaration of that interface type, and not referenced via `@EnableFeignClients` for a different package scope) is dead code. The Agent still records its methods as Outbound_Calls — the interface declares an *intent* to call out, and the static analysis cannot prove the interface is unreachable — but flags each row with `Marker: feign-client-unused` in the evidence table so the user can review. This is the only false-positive class where the Agent emits the row anyway; for the others, the row is suppressed.

- **Reflective and dynamic invocations.** `RestTemplate#execute(...)` calls whose `HttpMethod` argument is a runtime-computed value (e.g. read from a request, a map lookup) cannot be classified to a single HTTP method. The Agent records the row with `httpMethod: null` and `Marker: http-method-dynamic`. Similarly, `WebClient.method(httpMethodVar)` with a non-literal argument records `httpMethod: null` and the same marker.

- **Mock and stub libraries.** Classes from `org.mockito.*`, `com.github.tomakehurst.wiremock.*`, and similar test-double libraries that mimic `RestTemplate` / `WebClient` API surfaces are excluded by the test-scope rule above. If a production class shadows one of these names (e.g. a custom `RestTemplate` wrapper in the service's own package), the Agent treats it as a real outbound only when its declared type or its imports resolve to `org.springframework.web.client.RestTemplate`; locally defined types with the same simple name are ignored.

When in doubt — when a recognition signal matches but the surrounding context suggests dead code or a test fixture — the Agent prefers omission over a false positive. Missing calls can be discovered through user review; spurious calls are harder to detect and erode trust in the graph.
