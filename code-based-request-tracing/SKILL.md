---
name: code-based-request-tracing
version: 0.1.0
description: |
  Reconstructs end-to-end request flows across many Java/Spring Boot
  microservice repositories from source code alone. Detects inbound
  HTTP endpoints (@RestController + mapping annotations) and outbound
  calls (RestTemplate, WebClient, @FeignClient, @KafkaListener) per
  service, then stitches them into cross-service traces. Azure
  Application Gateway hops are recorded opaquely; the user supplies
  optional host mappings to refine traces. Produces tracing-report.md
  (Mermaid diagrams + hop-by-hop evidence table) and tracing-graph.json.
  Use when reconstructing how a request flows across multiple Spring
  Boot services in separate Azure DevOps repositories using static
  analysis only.
keywords:
  - request tracing
  - cross-service
  - microservices
  - Spring Boot
  - Java
  - Azure DevOps
  - static analysis
  - Mermaid
  - sequence diagram
---

# Code-Based Request Tracing

Use this skill when the goal is to reconstruct end-to-end request flows across multiple Java/Spring Boot microservice repositories using only the source code. The skill is a peer of `analyzing-spring-boot-codebases` and reuses that skill's per-service detection vocabulary by reference; it does not duplicate or modify any file inside that peer skill.

## When to use

- Multiple Spring Boot services live in separate repositories (typically Azure DevOps) and you need a single picture of how a request flows across them.
- Runtime telemetry (Application Insights, Azure Monitor, distributed tracing) and gateway routing configuration are unavailable, incomplete, or out of scope.
- A static-only reconstruction is acceptable: opaque hops stay opaque unless the user supplies a host mapping.

## Out of scope

- **Polyglot and non-Spring-Boot frameworks.** This skill only analyzes Java code in Spring Boot projects. Services in other languages or frameworks are recorded as `non-spring-boot` and skipped for endpoint and call detection.
- **Gateway configuration parsing.** The skill does not read Azure Application Gateway listener rules, backend pools, URL maps, ARM/Bicep templates, Terraform `azurerm_application_gateway*` resources, or any other gateway routing artifact.
- **Runtime and observability data.** The skill does not consult Application Insights, Azure Monitor, OpenTelemetry traces, log aggregation, or any runtime telemetry.
- **Per-service deep audit.** For controller patterns, layered architecture review, security, data-layer depth, and similar concerns, defer to `../analyzing-spring-boot-codebases/`.
- **Inferring downstream services for opaque hops.** A hostname or URL is never auto-mapped to a service. Resolution requires a user-supplied host mapping.

## Inputs

- **Repository paths** — a list of one or more local Service_Repo directory paths (absolute or workspace-relative). Required.
- **Host mappings** — optional path to a `host-mappings.yaml` that maps hostnames or URL prefixes to service names. See `reference/target-resolution.md` for the format.
- **Output directory** — optional path for the produced artifacts. Defaults to `./tracing-output/` relative to the current working directory.

## Outputs

Always produce both of the following in the output directory:

- `tracing-report.md` — Mermaid sequence diagrams (one per entry point), one Mermaid component diagram, run metadata, and a hop-by-hop evidence table.
- `tracing-graph.json` — a machine-readable graph of services, endpoints, calls, opaque hosts, Kafka topics, host mappings, and stitched traces.

If the output directory does not exist, create it. If either file already exists, overwrite it. See `reference/output-format.md` and `reference/tracing-graph-schema.md`.

## Workflow

Copy this checklist into the working notes and keep it updated:

```text
Tracing Progress
- [ ] Step 1: Accept inputs (repo paths, host mappings, output dir)
- [ ] Step 2: Per-repo
    - [ ] Resolve service name
    - [ ] Detect Spring Boot (or mark non-spring-boot)
    - [ ] Detect inbound endpoints
    - [ ] Detect outbound calls
    - [ ] Resolve URL strings from @Value / properties / constants
- [ ] Step 3: Merge into Tracing_Graph; disambiguate name conflicts
- [ ] Step 4: Apply user host mappings
- [ ] Step 5: Stitch cross-service traces (one per entry point)
- [ ] Step 6: Render tracing-report.md
- [ ] Step 7: Write tracing-graph.json
```

## Conditional loading guidance

The workflow is short, so most reference files are effectively always-on. Load them in this order as you progress through the steps.

| Reference file | When to load | Purpose |
|---|---|---|
| `reference/multi-repo-input.md` | Step 1 and Step 3 | Accepting repo lists, resolving service names, conflict and skip handling |
| `reference/detection-patterns.md` | Step 2 (inbound and outbound) | Inbound endpoint detection (defers to peer skill) and the four outbound call kinds |
| `reference/target-resolution.md` | Step 2 (URL resolution) and Step 4 | `@Value` / properties / constant resolution, opaque hops, host-mapping rules |
| `reference/trace-construction.md` | Step 5 | Stitching algorithm, cycle handling, depth limit, path matching |
| `reference/output-format.md` | Step 6 | `tracing-report.md` structure, Mermaid templates, marker rendering, evidence table |
| `reference/tracing-graph-schema.md` | Step 7 | `tracing-graph.json` shape, required keys, closed enums, encoding rules |
| `reference/examples.md` | Any step (illustrative) | One worked two-service example with a Kafka topic, an opaque hop, and a host mapping |

## Cross-skill references

This skill reuses the per-service detection vocabulary of the peer skill by reference. Use these documents for the controller/mapping pattern catalog and the layered-architecture vocabulary; do not duplicate them here.

- [`../analyzing-spring-boot-codebases/reference/api-and-contracts.md`](../analyzing-spring-boot-codebases/reference/api-and-contracts.md) — full controller and mapping annotation catalog (read-only reference).
- [`../analyzing-spring-boot-codebases/reference/architecture.md`](../analyzing-spring-boot-codebases/reference/architecture.md) — layered-architecture vocabulary used by this skill (read-only reference).

This skill never edits, renames, moves, or deletes any file inside `../analyzing-spring-boot-codebases/`.
