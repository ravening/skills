# Target Resolution

## Scope

This file covers how the Agent turns the raw target expression captured for an Outbound_Call (a URL string, a hostname, or a Kafka topic name as written in source) into a structured, classified target on the Tracing_Graph. It owns four concerns:

1. The resolution-source priority used to reduce a source expression to a concrete string, all confined to the same Service_Repo.
2. How concatenation is handled and how `{var}` placeholders are preserved for later path matching.
3. The four `status` outcomes (`resolved-service`, `opaque-host`, `kafka-topic`, `unresolved`) and exactly when each is assigned.
4. The Host_Mapping YAML format and the matching rules that flip an `opaque-host` to a `resolved-service` while preserving provenance.

This file does NOT cover detection of the call site itself (see [detection-patterns.md](detection-patterns.md)) or the cross-service stitching algorithm (see [trace-construction.md](trace-construction.md)). It also does not duplicate any inbound-controller pattern work — those live in [`../../analyzing-spring-boot-codebases/reference/api-and-contracts.md`](../../analyzing-spring-boot-codebases/reference/api-and-contracts.md).

## Resolution sources and priority

For each Outbound_Call, the Agent resolves the target string using the sources below, in this strict priority order. Resolution is **confined to the same Service_Repo** as the calling code. The Agent does not consult a different repo's properties, constants, or configuration to resolve a call.

1. **Literal string in source.** A string literal passed directly to the call is the resolved target as-is.
2. **`@Value` injection plus property files.** When the call uses a field or constructor parameter annotated with `@Value("${prop.name}")` (or referenced via `${prop.name}` inside a SpEL expression), the Agent reads `prop.name` from `src/main/resources/application.yml`, `application.yaml`, or `application.properties` (default profile only — `application-<profile>.yml` files are not consulted). The first file containing the key wins.
3. **`static final` constants.** When the call references a constant (`static final String BASE_URL = "..."`) defined in the same package or class as the call site, the Agent inlines that constant value.
4. **`@ConfigurationProperties` getters.** When the call reads from a Spring `@ConfigurationProperties`-bound class via an explicit getter (e.g. `paymentsProps.getBaseUrl()`), the Agent treats it like the equivalent `@Value` lookup: resolve the bound prefix + property name from the same property files as step 2.

Sources higher in the list take precedence: if a literal is present, properties are not consulted; if `@Value` resolves the value, constants and configuration-properties classes are not consulted.

What is **out of scope** for resolution:

- Method arguments and local-variable expressions whose values come from caller code (record as `unresolved`).
- Environment variables, OS-level overrides, or remote config-server values not committed to the repo (record as `unresolved`).
- Profile-specific property files (`application-prod.yml`, `application-staging.yml`).
- Any other repo's properties, constants, or configuration.

When none of the four sources reduces the expression to a concrete string, the Agent records the original source expression verbatim in `targetRaw`, sets `targetResolved` to `null`, and assigns `status: unresolved`.

## Concatenation handling and `{var}` placeholders

Outbound URLs are commonly built from a base plus a path with embedded variables, for example:

```java
restTemplate.getForObject(baseUrl + "/users/" + id, User.class);
```

The Agent resolves each component independently:

- Constant components (literals, `@Value`-injected fields, `static final` constants, `@ConfigurationProperties` getters) are inlined using the priority order above.
- Method-argument components and local variables that cannot be inlined are preserved as `{name}` placeholders, where `name` is the source identifier (or `{var}` when no usable name is available). These placeholders are kept in the resolved string so that path matching against another service's Inbound_Endpoint can still work — controller paths use the same `{name}` placeholder syntax.

The result for the example above (with `baseUrl` resolving to `https://payments.internal.acme.com`) is:

- `targetRaw`: `baseUrl + "/users/" + id`
- `targetResolved`: `https://payments.internal.acme.com/users/{id}`

Concatenation rules:

- Repeated slashes produced by joining components are collapsed to a single slash.
- A trailing slash on the joined path is removed unless the path is exactly `/`.
- If concatenation produces an empty string, the resolved path is `/` and a warning row is emitted in the evidence table (per [output-format.md](output-format.md)).

If **any** component of the concatenation cannot be resolved or expressed as a `{name}` placeholder (for example, the host portion comes from a method argument), the entire call is recorded as `unresolved` — the Agent does not partially resolve a target host.

## Status assignment

Each Outbound_Call is assigned exactly one `status` after resolution and after host-mapping is applied. The closed enum is `{ resolved-service, opaque-host, kafka-topic, unresolved }`. The rules below are evaluated top to bottom; the first match wins.

| Status | Assigned when |
|---|---|
| `kafka-topic` | The call kind is `kafka-listener` (consumer node) or `kafka-template` (outbound publish to a topic). The `targetRaw` is the topic name or pattern as written in source; `targetResolved` is the resolved topic name after `@Value` / properties inlining (or the same as `targetRaw` if it was already a literal). Host mappings do not apply. |
| `unresolved` | The call kind is one of the HTTP kinds (`rest-template`, `web-client`, `feign-client`) and any portion of the host or URL prefix could not be reduced to a concrete string from the four resolution sources. `targetResolved` is `null`; `targetRaw` preserves the original source expression verbatim. |
| `resolved-service` | The call kind is one of the HTTP kinds, the target reduced to a concrete URL or hostname, and either (a) a Host_Mapping entry matched, or (b) a path-suffix match against another Service's Inbound_Endpoint connected the call to a known Service. `targetEndpointId` is non-null; if a mapping produced the resolution, `mappingApplied` is the matching mapping entry's id. |
| `opaque-host` | The call kind is one of the HTTP kinds, the target reduced to a concrete URL or hostname, and no Host_Mapping matched and no path-suffix match against another Service's endpoint connected it. The hop is recorded as "Service A calls hostname X" with no inferred downstream Service. |

Notes:

- `targetEndpointId` is non-null **only** when `status == resolved-service`.
- `httpMethod` is non-null for all HTTP kinds and `null` for `kafka-listener` and `kafka-template`.
- The Agent never invents a downstream Service to "complete" a hop. An unresolved or opaque hop stays unresolved or opaque unless a user-supplied Host_Mapping resolves it.

## Host_Mapping YAML format

The user optionally supplies a Host_Mapping file whose path is given as a skill input. The file is YAML and lists one or more mapping entries:

```yaml
mappings:
  - hostPattern: "orders.internal.acme.com"
    service: orders-service
  - hostPattern: "*.payments.acme.com"
    service: payments-service
  - urlPrefix: "https://gateway.acme.com/v1/inventory"
    service: inventory-service
```

Each entry has exactly one of `hostPattern` or `urlPrefix`, plus a required `service` field naming a known Service in the Tracing_Graph. The Agent assigns each entry a stable id (e.g. `mapping_001`, `mapping_002`) in load order; this id is what `calls[].mappingApplied` references.

Field semantics:

- `hostPattern` — a hostname or wildcard hostname. Wildcards use a leading `*.` (e.g. `*.payments.acme.com`) and match exactly one or more leading DNS labels. No glob, no regex.
- `urlPrefix` — a normalized URL prefix (scheme + host + port + path prefix). Matching is performed on the resolved URL after the same normalization rules used for path concatenation (collapse repeated slashes; preserve `{var}` placeholders; trailing slash removed except when the prefix is exactly the root).
- `service` — must match a `services[].name` in the Tracing_Graph after name-conflict disambiguation.

If `service` does not match any known Service name, the mapping is recorded but not applied; the run-metadata block in `tracing-report.md` flags the unused mapping under "Host mappings applied" with a `(no service)` note.

## Matching rules

When the Agent resolves an HTTP-kind Outbound_Call to a concrete URL or hostname, it consults the Host_Mapping list using these rules, in this order:

1. **Exact host match wins over wildcard.** An exact `hostPattern` match (e.g. `orders.internal.acme.com`) takes precedence over any wildcard `hostPattern` (e.g. `*.acme.com`) that would also match the same host.
2. **`urlPrefix` matches by URL prefix after normalization.** The resolved URL is normalized (collapse repeated slashes, strip trailing slash except for root), then compared to the normalized `urlPrefix`. The match succeeds when the resolved URL starts with the prefix at a path-segment boundary. A `urlPrefix` is more specific than a `hostPattern` covering the same host: if both could match, the `urlPrefix` wins.
3. **First matching entry wins among equally specific matches.** If two `hostPattern` entries with the same specificity match (for example, two identical wildcards), the entry that appears first in the YAML file wins.

When a mapping resolves a hop:

- `targetResolved` becomes the matched Service's name **for graph-stitching purposes**, but the **fully resolved URL or hostname** is preserved in the evidence row of the report so the reader can verify the mapping. In `tracing-graph.json`, `targetResolved` is the resolved URL or hostname (not the service name) and `targetEndpointId` carries the connected endpoint id.
- `mappingApplied` is set to the matching mapping entry's id.
- `targetRaw` is preserved verbatim — the original source expression is never overwritten by mapping or resolution.

This three-field provenance (`targetRaw`, `targetResolved`, `mappingApplied`) lets a reader trace any resolved-service hop back to (a) the exact source expression, (b) the resolved URL after property inlining, and (c) the user-supplied mapping rule that produced the service binding.

## Explicit non-goals

This skill deliberately does NOT do the following, and the Agent must not attempt them even when sources appear to be available:

- Parse Azure Application Gateway listener rules, backend pools, URL maps, or routing configuration.
- Parse ARM templates, Bicep files, or Terraform `azurerm_application_gateway*` resources (or any other gateway-config artifact in any IaC dialect).
- Infer a downstream Service from a hostname alone when no Host_Mapping is supplied. Hostnames without a mapping stay `opaque-host`.
- Consult external service registries, DNS records, or runtime data (Application Insights, Azure Monitor, service mesh control planes) to resolve hops.
- Read profile-specific property files (`application-<profile>.yml`) to vary resolution by environment. The default profile only is consulted; environment-specific behavior is intentionally out of scope for v0.1.
- Cross-repo property lookup: even if Service A imports a shared library from Service B's repo, the Agent does not read Service B's properties to resolve Service A's calls.

These boundaries keep the resolution behavior deterministic and reproducible from source code alone.
