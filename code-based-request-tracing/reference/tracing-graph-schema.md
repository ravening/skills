# Tracing Graph Schema

## Scope

This file defines the on-disk schema for `tracing-graph.json`, the machine-readable artifact the Agent emits alongside `tracing-report.md`. It specifies the canonical JSON shape, the six required top-level keys mandated by the requirements, the additive metadata keys the design adds for downstream tooling, and the field-by-field semantics (types, closed enums, nullability, encoding) every entry must satisfy.

It does not cover how to render the Markdown report (see [output-format.md](output-format.md)), how target strings are resolved (see [target-resolution.md](target-resolution.md)), or how traces are walked (see [trace-construction.md](trace-construction.md)). Those files describe the inputs that flow into this schema; this file describes the output shape they all converge on.

## Canonical JSON example

The following is the canonical shape of `tracing-graph.json`. Every key and value type shown here is part of the schema; the literal values are illustrative.

```json
{
  "schemaVersion": "1.0",
  "runTimestamp": "2025-01-15T12:34:56Z",
  "outputDir": "./tracing-output/",
  "services": [
    {
      "name": "orders-service",
      "repoPath": "/abs/path/to/orders-service",
      "springBoot": true,
      "nonSpringBootReason": null
    }
  ],
  "endpoints": [
    {
      "id": "ep_orders_post_orders",
      "serviceName": "orders-service",
      "httpMethod": "POST",
      "path": "/orders",
      "evidenceFile": "src/main/java/com/acme/orders/web/OrderController.java",
      "evidenceLine": 42
    }
  ],
  "calls": [
    {
      "id": "call_001",
      "callerServiceName": "orders-service",
      "kind": "rest-template",
      "targetRaw": "${payments.base-url}/charges",
      "targetResolved": "https://payments.internal.acme.com/charges",
      "targetEndpointId": "ep_payments_post_charges",
      "httpMethod": "POST",
      "evidenceFile": "src/main/java/com/acme/orders/service/OrderService.java",
      "evidenceLine": 84,
      "status": "resolved-service",
      "mappingApplied": "mapping_002"
    }
  ],
  "opaqueHosts": [
    { "host": "gateway.acme.com", "firstSeenCallId": "call_007" }
  ],
  "kafkaTopics": [
    { "name": "orders.events", "consumers": ["orders-service"], "producers": [] }
  ],
  "hostMappings": [
    { "id": "mapping_002", "hostPattern": "payments.internal.acme.com", "urlPrefix": null, "service": "payments-service" }
  ],
  "traces": [
    {
      "entryPointId": "ep_orders_post_orders",
      "nodes": [
        { "kind": "endpoint", "id": "ep_orders_post_orders" },
        { "kind": "call", "id": "call_001" },
        { "kind": "endpoint", "id": "ep_payments_post_charges" }
      ],
      "markers": []
    }
  ],
  "skippedRepositories": [
    { "path": "/abs/path/to/legacy", "reason": "not-spring-boot" }
  ],
  "nameConflicts": []
}
```

## Top-level keys

The top-level value is a single JSON object. Its keys split into two groups.

### Required keys (from requirement 9.2)

These six keys MUST be present on every successful run, even when their array is empty (an empty array still satisfies "the key exists"). An empty run with no accepted repos produces six empty arrays plus the metadata keys below.

| Key | Type | Purpose |
|---|---|---|
| `services` | array of Service | One entry per accepted Service_Repo, in input order after disambiguation. |
| `endpoints` | array of Endpoint | One entry per detected Inbound_Endpoint across all accepted Services. |
| `calls` | array of Call | One entry per detected Outbound_Call (including `kafka-template` publishes) across all accepted Services. |
| `opaqueHosts` | array of OpaqueHost | One entry per distinct hostname that appears as the resolved target of at least one call with `status: opaque-host`. |
| `kafkaTopics` | array of KafkaTopic | One entry per distinct topic name that appears in any `@KafkaListener` consumer or `kafka-template` producer. |
| `hostMappings` | array of HostMapping | The host mappings supplied by the user for this run, echoed back verbatim with assigned ids. Empty array if the user supplied none. |

### Additive metadata keys

The design adds these keys so downstream tooling can interpret the file without re-parsing the Markdown report. They are part of the schema but go beyond what requirement 9.2 mandates.

| Key | Type | Purpose |
|---|---|---|
| `schemaVersion` | string | Schema identifier; current value is `"1.0"`. Bumped on any breaking change to this document. |
| `runTimestamp` | string (ISO 8601 UTC) | The single source of truth for when the run executed. The only field expected to differ across otherwise-identical reruns. |
| `outputDir` | string | The output directory the Agent wrote to, as supplied by the user (or `./tracing-output/` if defaulted). |
| `traces` | array of Trace | The walk results, one entry per entry point selected (default: every `endpoints[]` entry). Allows downstream tooling to reconstruct sequence diagrams without re-walking the graph. |
| `skippedRepositories` | array of SkippedRepo | Repos that were not analyzed, with reason codes. Empty array if every input repo was accepted. |
| `nameConflicts` | array of NameConflict | Records of Service-name collisions that were resolved by appending `@<repoDirName>`. Empty array if no collisions occurred. |

The Agent MUST emit all six required keys and SHOULD emit all metadata keys. A consumer that ignores unknown keys remains forward-compatible with future additions under this `schemaVersion`.

## Field-by-field semantics

### `services[]` — Service

```json
{
  "name": "orders-service",
  "repoPath": "/abs/path/to/orders-service",
  "springBoot": true,
  "nonSpringBootReason": null
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | The disambiguated Service name (post `@<repoDirName>` suffix when applicable). Unique across `services[]`. Used as the join key for `endpoints[].serviceName` and `calls[].callerServiceName`. |
| `repoPath` | string | yes | The absolute, symlink-resolved path of the Service_Repo root. Forward slashes only (see "Path encoding" below). |
| `springBoot` | boolean | yes | `true` if the repo was detected as Spring Boot via `pom.xml`, `build.gradle`, or `build.gradle.kts`; `false` otherwise. A `false` value implies the corresponding `endpoints[]` and `calls[]` arrays contain zero entries with this `name`. |
| `nonSpringBootReason` | string \| null | yes | `null` when `springBoot == true`. When `springBoot == false`, a short reason code such as `"no-spring-boot-dependency"` or `"no-build-file"`. |

A repo that was skipped entirely (`not-found`, `not-readable`, `not-spring-boot`) does **not** appear in `services[]`; it appears only under `skippedRepositories[]`. The `services[]` array contains only repos the Agent accepted as Services.

### `endpoints[]` — Endpoint

```json
{
  "id": "ep_orders_post_orders",
  "serviceName": "orders-service",
  "httpMethod": "POST",
  "path": "/orders",
  "evidenceFile": "src/main/java/com/acme/orders/web/OrderController.java",
  "evidenceLine": 42
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Synthetic, stable identifier for the endpoint. Stable across reruns over the same source state, so consumers can diff runs by id. Convention: `ep_<service>_<method>_<path-slug>`. Unique across `endpoints[]`. |
| `serviceName` | string | yes | Must equal the `name` of some entry in `services[]`. |
| `httpMethod` | string (enum HttpMethod) | yes | One of `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `OPTIONS`, `HEAD`, `TRACE`. Uppercase. For `@RequestMapping` without an explicit `method=`, default `GET` and surface the warning in the evidence table; the JSON value is still `GET`. |
| `path` | string | yes | The normalized concatenation of class-level and method-level mapping paths. See "Path normalization" below. `{var}` placeholders are preserved verbatim. |
| `evidenceFile` | string | yes | Repo-relative path to the source file containing the handler method. Forward slashes only. |
| `evidenceLine` | integer | yes | 1-based line number where the handler method's mapping annotation (or the method declaration when the annotation is on the class) appears. |

The five required fields per requirement 9.4 are `serviceName`, `httpMethod`, `path`, `evidenceFile`, and `evidenceLine`. The `id` is additive metadata that enables the cross-references in `calls[].targetEndpointId` and `traces[].nodes[]`.

### `calls[]` — Call

```json
{
  "id": "call_001",
  "callerServiceName": "orders-service",
  "kind": "rest-template",
  "targetRaw": "${payments.base-url}/charges",
  "targetResolved": "https://payments.internal.acme.com/charges",
  "targetEndpointId": "ep_payments_post_charges",
  "httpMethod": "POST",
  "evidenceFile": "src/main/java/com/acme/orders/service/OrderService.java",
  "evidenceLine": 84,
  "status": "resolved-service",
  "mappingApplied": "mapping_002"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Synthetic, stable identifier. Convention: `call_<zero-padded-sequence>`. Unique across `calls[]`. |
| `callerServiceName` | string | yes | Must equal the `name` of some entry in `services[]`. |
| `kind` | string (enum CallKind) | yes | Closed set: `rest-template`, `web-client`, `feign-client`, `kafka-listener`, `kafka-template`. See "Closed enum: `kind`" below. |
| `targetRaw` | string | yes | The verbatim source expression of the call target — the URL string literal, the `${prop}` placeholder, the concatenation expression, or the topic name as written. Never null and never paraphrased. |
| `targetResolved` | string \| null | yes | The fully resolved target string when resolution succeeded; `null` when `status == "unresolved"`. For `status == "resolved-service"`, holds the mapped Service name. For `status == "opaque-host"`, holds the resolved hostname or URL. For `status == "kafka-topic"`, holds the topic name. |
| `targetEndpointId` | string \| null | yes | Non-null **only** when `status == "resolved-service"` AND a path match was found against an `endpoints[]` entry of the resolved Service. For every other status, MUST be `null`. See "Nullability rules" below. |
| `httpMethod` | string (enum HttpMethod) \| null | yes | The HTTP method for HTTP-shaped calls. MUST be `null` when `kind` is `kafka-listener` or `kafka-template`. See "Nullability rules" below. |
| `evidenceFile` | string | yes | Repo-relative path to the source file containing the call site (or the Feign-interface method, or the `@KafkaListener`-annotated method). Forward slashes only. |
| `evidenceLine` | integer | yes | 1-based line number of the call site or annotation. |
| `status` | string (enum CallStatus) | yes | Closed set: `resolved-service`, `opaque-host`, `kafka-topic`, `unresolved`. See "Closed enum: `status`" below. |
| `mappingApplied` | string \| null | yes | The `id` of the `hostMappings[]` entry that produced the resolution, when one applied; `null` otherwise. Non-null implies `status == "resolved-service"`. |

The eight required fields per requirement 9.5 are `callerServiceName`, `kind`, `targetRaw`, `targetResolved`, `httpMethod`, `evidenceFile`, `evidenceLine`, and `status`. The `id`, `targetEndpointId`, and `mappingApplied` are additive metadata for cross-referencing.

### `opaqueHosts[]` — OpaqueHost

```json
{ "host": "gateway.acme.com", "firstSeenCallId": "call_007" }
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `host` | string | yes | The distinct hostname (no scheme, no path). One entry per distinct host across all calls with `status == "opaque-host"`. |
| `firstSeenCallId` | string | yes | The `calls[].id` of the first call that introduced this host, in `calls[]` order. Lets readers locate evidence quickly. |

### `kafkaTopics[]` — KafkaTopic

```json
{ "name": "orders.events", "consumers": ["orders-service"], "producers": [] }
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | The topic name as resolved (or as a literal pattern when `topicPattern` was used). |
| `consumers` | array of string | yes | Sorted, de-duplicated list of Service names that have a `@KafkaListener` for this topic. |
| `producers` | array of string | yes | Sorted, de-duplicated list of Service names that publish to this topic via `kafka-template`. Empty array when no producer is detected statically (common, since `KafkaTemplate.send` does not always carry a literal topic argument). |

### `hostMappings[]` — HostMapping

```json
{ "id": "mapping_002", "hostPattern": "payments.internal.acme.com", "urlPrefix": null, "service": "payments-service" }
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Synthetic, stable identifier referenced by `calls[].mappingApplied`. Convention: `mapping_<zero-padded-sequence>` in input order. |
| `hostPattern` | string \| null | yes | Exact host or wildcard host pattern (e.g. `*.payments.acme.com`). Exactly one of `hostPattern` and `urlPrefix` is non-null. |
| `urlPrefix` | string \| null | yes | URL prefix used for prefix-match resolution (e.g. `https://gateway.acme.com/v1/inventory`). Exactly one of `hostPattern` and `urlPrefix` is non-null. |
| `service` | string | yes | The Service name this pattern resolves to. Must equal the `name` of some entry in `services[]`. |

### `traces[]` — Trace

```json
{
  "entryPointId": "ep_orders_post_orders",
  "nodes": [
    { "kind": "endpoint", "id": "ep_orders_post_orders" },
    { "kind": "call", "id": "call_001" },
    { "kind": "endpoint", "id": "ep_payments_post_charges" }
  ],
  "markers": []
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `entryPointId` | string | yes | The `endpoints[].id` of the entry point this trace starts at. |
| `nodes` | array of TraceNode | yes | The walk result in visit order. A TraceNode is `{ "kind": "endpoint", "id": <endpoints[].id> }`, `{ "kind": "call", "id": <calls[].id> }`, or `{ "kind": "leaf", "target": <string>, "markerKind": <string> }` for terminal opaque/unresolved/kafka-topic leaves. |
| `markers` | array of string | yes | Zero or more of `cycle-detected`, `max-depth-reached`, `match-ambiguous`. Empty array when the walk completed normally with unambiguous matches. |

### `skippedRepositories[]` — SkippedRepo

```json
{ "path": "/abs/path/to/legacy", "reason": "not-spring-boot" }
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `path` | string | yes | The input path as the user supplied it (not normalized), so the user can match it against their input list. |
| `reason` | string | yes | One of `not-found`, `not-readable`, `not-spring-boot`. See [multi-repo-input.md](multi-repo-input.md) for definitions. |

### `nameConflicts[]` — NameConflict

```json
{ "originalName": "payments", "repos": ["/abs/path/to/payments-prod", "/abs/path/to/payments-staging"], "resolvedNames": ["payments@payments-prod", "payments@payments-staging"] }
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `originalName` | string | yes | The colliding name before disambiguation. |
| `repos` | array of string | yes | The absolute paths of all repos that resolved to `originalName`. |
| `resolvedNames` | array of string | yes | The disambiguated names, in the same order as `repos`. Each is `<originalName>@<repoDirName>`. |

## Closed enums

### `kind` — CallKind

```text
{ "rest-template", "web-client", "feign-client", "kafka-listener", "kafka-template" }
```

The set is closed: the Agent MUST NOT emit any other value. Mapping from detection source:

- `rest-template` — invocations on a `RestTemplate` reference (`getForObject`, `postForEntity`, `exchange`, etc.).
- `web-client` — `WebClient` builder chains terminating in `.uri(...)`.
- `feign-client` — methods declared on `@FeignClient` interfaces (one entry per method).
- `kafka-listener` — methods annotated with `@KafkaListener` (consumer side; recorded as an outbound-shaped call into the topic node).
- `kafka-template` — `KafkaTemplate.send(...)` invocations (producer side).

### `status` — CallStatus

```text
{ "resolved-service", "opaque-host", "kafka-topic", "unresolved" }
```

The set is closed. Mapping from resolution outcome:

- `resolved-service` — the target was fully resolved to a hostname or URL AND a host mapping (or path-suffix match against another Service's inbound endpoint) connected it to a known Service. `targetResolved` holds the Service name; `targetEndpointId` is non-null when a path match also succeeded; `mappingApplied` holds the mapping id when a mapping produced the resolution.
- `opaque-host` — the target was fully resolved to a hostname or URL but no host mapping matched. `targetResolved` holds the resolved hostname or URL.
- `kafka-topic` — the call is a Kafka consumer (`kafka-listener`) or producer (`kafka-template`) bound to a topic. `targetResolved` holds the topic name (or pattern).
- `unresolved` — the target expression contains values that could not be resolved from sources within the same Service_Repo (e.g. environment variables, method arguments, remote config server values). `targetResolved` is `null`; `targetRaw` preserves the original expression verbatim.

## Nullability rules

The following nullability invariants MUST hold for every entry in `calls[]`. They are derived from the design's data model and from requirement 9.5's enum semantics.

1. **`httpMethod` is `null` for Kafka kinds.**
   - When `kind == "kafka-listener"` OR `kind == "kafka-template"`, `httpMethod` MUST be `null`.
   - For all other `kind` values (`rest-template`, `web-client`, `feign-client`), `httpMethod` MUST be a non-null `HttpMethod` enum value.

2. **`targetEndpointId` is non-null only when `status == "resolved-service"`.**
   - When `status != "resolved-service"`, `targetEndpointId` MUST be `null`.
   - When `status == "resolved-service"` AND `findEndpointMatch` returned a single matching endpoint, `targetEndpointId` is that endpoint's `id`.
   - When `status == "resolved-service"` AND no path match was found (the host resolves but no inbound endpoint matches the path), `targetEndpointId` is `null` and the trace records the `resolved-service-no-endpoint-match` leaf kind.
   - When `status == "resolved-service"` AND multiple endpoints matched, the call still records a single `targetEndpointId` (the first match in `endpoints[]` order); the additional candidates are surfaced via the `match-ambiguous` marker on the corresponding trace, not via this field.

3. **`mappingApplied` is non-null only when a host mapping produced the resolution.**
   - Non-null `mappingApplied` implies `status == "resolved-service"`.
   - `status == "resolved-service"` does not imply non-null `mappingApplied`: a path-suffix match against another Service's inbound endpoint can also produce `resolved-service` without a mapping.

4. **`targetResolved` is `null` if and only if `status == "unresolved"`.**
   - Every other status implies a non-null `targetResolved`.

5. **`nonSpringBootReason` in `services[]` is `null` if and only if `springBoot == true`.**

## `evidenceLine` is 1-based

Every `evidenceLine` integer in `endpoints[]` and `calls[]` is **1-based**, matching the convention used by Java compilers, IDEs, and stack traces. Line `1` is the first line of the file. A value of `0` is invalid.

When a span covers multiple lines (e.g. a builder chain across several lines), the recorded line is the line of the **first relevant token**: the mapping annotation for endpoints, the call site or annotation for calls.

## Path encoding

All path strings in the schema use **forward slashes (`/`) only**, regardless of the host operating system. This applies to:

- `services[].repoPath` (absolute paths)
- `endpoints[].evidenceFile` and `calls[].evidenceFile` (repo-relative)
- `endpoints[].path` and any URL-shaped value in `calls[].targetResolved`

Backslashes from Windows-style paths MUST be converted to forward slashes before serialization. This keeps the artifact portable across operating systems and lets downstream tooling treat paths uniformly.

### Path normalization for `endpoints[].path`

The `path` field is normalized as follows before serialization:

1. Concatenate class-level and method-level mapping paths: `"/" + classPath + "/" + methodPath`.
2. Collapse repeated slashes to a single slash.
3. Strip any trailing slash, except when the path is exactly `/`.
4. Preserve `{var}` placeholders verbatim (do not URL-encode them).

A class-level path of `/api/orders` and a method-level path of `/{id}/items` produces `/api/orders/{id}/items`. An absent class-level path is treated as the empty string, producing `/{id}/items`. Empty concatenations resolve to `/`.

## File encoding

`tracing-graph.json` MUST be written as **UTF-8 without a BOM (byte-order mark)**. Specifically:

- The file is valid UTF-8.
- The file does NOT begin with the bytes `EF BB BF` (the UTF-8 BOM).
- Newlines may be either `LF` or `CRLF`; consumers MUST tolerate both. The Agent SHOULD prefer `LF` for portability.
- The JSON itself is a single top-level object; no leading or trailing whitespace beyond a single trailing newline is required.

This matches the conventions of the JSON specification (RFC 8259) and avoids interoperability issues with tools that reject BOMs in JSON input.

## Schema invariants summary

A valid `tracing-graph.json` satisfies all of the following:

- The top-level object contains all six required keys (`services`, `endpoints`, `calls`, `opaqueHosts`, `kafkaTopics`, `hostMappings`), each as an array (possibly empty).
- Every `endpoints[].serviceName`, `calls[].callerServiceName`, and `hostMappings[].service` equals the `name` of some entry in `services[]`.
- Every non-null `calls[].targetEndpointId` equals the `id` of some entry in `endpoints[]`.
- Every non-null `calls[].mappingApplied` equals the `id` of some entry in `hostMappings[]`.
- Every `traces[].entryPointId` equals the `id` of some entry in `endpoints[]`.
- Every `kind` value is in the closed CallKind set; every `status` value is in the closed CallStatus set; every `httpMethod` value (when non-null) is in the HttpMethod set.
- Every nullability rule in the section above holds.
- The file is valid UTF-8 without BOM and parses as JSON.

A consumer that validates these invariants can rely on the schema without further inspection of the producer.

## Cross-references

- [SKILL.md](../SKILL.md) — workflow Step 7 invokes this file.
- [multi-repo-input.md](multi-repo-input.md) — defines the inputs that populate `services[]`, `skippedRepositories[]`, and `nameConflicts[]`.
- [detection-patterns.md](detection-patterns.md) — defines what produces entries in `endpoints[]` and `calls[]`.
- [target-resolution.md](target-resolution.md) — defines how `calls[].targetRaw`, `targetResolved`, `status`, and `mappingApplied` are populated.
- [trace-construction.md](trace-construction.md) — defines how `traces[]` is populated and which markers are emitted.
- [output-format.md](output-format.md) — companion artifact (`tracing-report.md`) consumes the same data this schema describes.
