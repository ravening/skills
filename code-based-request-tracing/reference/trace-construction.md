# Trace Construction

This document tells the Agent how to stitch the per-service detections (services, endpoints, calls, host mappings, Kafka topics) into end-to-end cross-service traces. Run this step **after** the global Tracing_Graph has been assembled (i.e., after every accepted Service_Repo has been scanned and any user-supplied host mappings have been applied).

The output of this step is a `traces[]` array on the Tracing_Graph plus the data needed to render one Mermaid `sequenceDiagram` per entry point in `tracing-report.md`.

## Inputs

The Agent has, at this point:

- A `services[]` list (with `springBoot: true | false`).
- An `endpoints[]` list of every Inbound_Endpoint detected across all Spring Boot services.
- A `calls[]` list of every Outbound_Call (`rest-template`, `web-client`, `feign-client`, `kafka-listener`, `kafka-template`) with its `status` already assigned (`resolved-service`, `opaque-host`, `kafka-topic`, or `unresolved`) and any `mappingApplied` provenance.
- An `opaqueHosts[]`, `kafkaTopics[]`, and `hostMappings[]` list.
- A configurable `maxDepth` (default `10`).
- An optional list of entry-point selectors (see "Default entry-point selection" below).

The Agent does not re-scan source code during trace construction; this step operates purely over the in-memory graph.

## Algorithm — `buildTraces` and `walk`

The trace builder is a depth-bounded, cycle-safe walk rooted at every selected Inbound_Endpoint. The Agent must implement the algorithm as written; this is the canonical specification.

```text
function buildTraces(graph, maxDepth = 10):
    traces = []
    for each endpoint E in selectedEntryPoints(graph):
        seen = set()
        trace = walk(E, graph, maxDepth, seen, depth = 0)
        traces.append(trace)
    return traces

function walk(node, graph, maxDepth, seen, depth):
    if depth >= maxDepth:
        return Truncated(node, marker = "max-depth-reached")
    if node in seen:
        return Truncated(node, marker = "cycle-detected")
    seen.add(node)

    children = []
    for each call C originating from node.service that is reachable from node:
        target = resolveCallTarget(C, graph)
        switch target.status:
            case "resolved-service":
                # match path against target service's endpoints (after stripping host)
                matchedEndpoints = findEndpointMatch(target, graph)
                if matchedEndpoints.length == 0:
                    children.append(Leaf(target, kind = "resolved-service-no-endpoint-match"))
                else if matchedEndpoints.length == 1:
                    children.append(walk(matchedEndpoints[0], graph, maxDepth, copyOf(seen), depth + 1))
                else:
                    # ambiguous: fan out one child per candidate, mark each
                    for each M in matchedEndpoints:
                        child = walk(M, graph, maxDepth, copyOf(seen), depth + 1)
                        child.markers.append("match-ambiguous")
                        children.append(child)
            case "opaque-host":
                children.append(Leaf(target, kind = "opaque-host"))
            case "kafka-topic":
                children.append(Leaf(target, kind = "kafka-topic"))
            case "unresolved":
                children.append(Leaf(target, kind = "unresolved"))
    return Branch(node, children)
```

Notes on the pseudocode:

- `seen` is duplicated with `copyOf(seen)` per child so sibling branches do not poison each other's cycle detection. Cycle detection is per-path, not graph-wide.
- `depth` increments only when the walk steps into another endpoint (a true hop). Leaves (`opaque-host`, `kafka-topic`, `unresolved`, `resolved-service-no-endpoint-match`) do not consume depth; they terminate the branch.
- `Branch` and `Leaf` are the two `TraceNode` shapes serialized into `traces[].nodes`. `Truncated` is a `Leaf` carrying the truncation marker on the parent trace's `markers` array (also rendered as a Mermaid `Note over` on the diagram).
- Markers accumulate on the trace, not on individual nodes, except `match-ambiguous` which is attached to each ambiguous child branch so the renderer can label every candidate edge.

## Coarse intra-service reachability (v0.1 scope)

The clause "every outbound call originating from `node.service` that is reachable from `node`" is implemented in v0.1 as a deliberate over-approximation:

> Every Outbound_Call whose `callerServiceName` equals the endpoint's `serviceName` is treated as reachable from any Inbound_Endpoint of that service.

In other words, the Agent does **not** trace `controller → service → repository → client` call chains within a single service. It assumes that if a service exposes endpoint `E` and contains outbound call `C`, then `E` may invoke `C`.

This is intentionally coarse because:

- Precise intra-service call-graph analysis (controller-method → injected-service-method → outbound-client-call) is the territory of the peer skill `analyzing-spring-boot-codebases` and would require AST-level dataflow that is out of scope for v0.1.
- The over-approximation never hides hops; at worst, it shows extra hops on entry points that do not actually reach a given outbound call. Reviewers can cross-check by reading the evidence-table source files.
- Under-approximation (missing real hops) would be far more dangerous for a tracing tool than over-approximation.

A future revision may refine reachability by walking method-level dataflow within a service. The v0.1 scope decision is to keep the algorithm whole-service-coarse and document it explicitly so reviewers know the limitation.

## `findEndpointMatch` — path normalization and `{var}` wildcards

`findEndpointMatch` decides whether a `resolved-service` call lands on a concrete Inbound_Endpoint of the target service.

### Normalization

Before comparison, both the call path (extracted from `targetResolved` after stripping scheme + host) and every candidate endpoint path are normalized:

```text
function normalizePath(s):
    if s is null or empty:
        return "/"
    s = "/" + s
    s = collapseRepeatedSlashes(s)         # "//a///b" → "/a/b"
    if length(s) > 1 and s endsWith "/":
        s = s[:-1]                         # drop trailing slash, except for "/"
    return s
```

Examples:

| Raw | Normalized |
|---|---|
| `/orders/` | `/orders` |
| `orders` | `/orders` |
| `//api//v1/orders/` | `/api/v1/orders` |
| `""` | `/` |

### Segment-by-segment matching with `{var}` wildcards

After normalization, both paths are split on `/`. The match succeeds when:

1. Both paths have the same segment count.
2. For each position `i`, either:
   - `callSegments[i] == endpointSegments[i]` (literal equality), or
   - `endpointSegments[i]` is a `{var}` placeholder (e.g. `{id}`, `{orderId}`), which acts as a single-segment wildcard, or
   - `callSegments[i]` is a `{var}` placeholder preserved during target resolution (e.g. `${baseUrl}/users/{id}`) and `endpointSegments[i]` is also a placeholder.
3. The HTTP methods match (`call.httpMethod == endpoint.httpMethod`).

A `{var}` placeholder matches **exactly one** segment; multi-segment wildcards (e.g. `**`) are not supported in v0.1. Spring's `*` and `**` PathPattern wildcards on the endpoint side are treated as literal characters (i.e., they will not match), which is acceptable because the Agent reports false negatives as `resolved-service-no-endpoint-match` rather than fabricating matches.

Query strings and fragments on the call URL are stripped before matching.

### Ambiguity — `match-ambiguous` marker

When `findEndpointMatch` returns more than one endpoint (for example, a call to `GET /users/{id}` matches two endpoints from different controllers in the same service that both declare `GET /users/{id}` after normalization, or a call with multiple `{var}` segments matches several similarly-shaped endpoints), the Agent must:

1. Emit one child branch per candidate endpoint by walking each independently.
2. Append the marker `match-ambiguous` to **each** of those child branches' `markers` arrays.
3. Render every candidate edge in the corresponding sequence diagram with the `(ambiguous)` label.
4. Add an `Ambiguity` row to the hop-by-hop evidence table for the originating call so reviewers can see why the trace branched.

The Agent does not pick a "best" candidate. Showing all candidates lets reviewers resolve the ambiguity by reading the source.

## Default entry-point selection

The default selection is **every Inbound_Endpoint of every Service** that has `springBoot: true`. Concretely:

```text
function selectedEntryPoints(graph):
    if userEntryPointSelectors is empty:
        return graph.endpoints
    else:
        return filterByUserSelectors(graph.endpoints, userEntryPointSelectors)
```

Non-Spring-Boot services have no endpoints (Requirement 3.3), so they contribute no entry points by construction.

### Optional `service:method:path` selectors

The user may narrow the entry-point set by passing a list of selectors in the form:

```text
<service-name>:<HTTP-METHOD>:<path>
```

Examples:

```text
orders-service:POST:/orders
payments-service:GET:/charges/{id}
```

Selector matching rules:

- `<service-name>` matches `endpoints[].serviceName` exactly (after any name-conflict disambiguation, e.g. `orders-service@orders-repo`).
- `<HTTP-METHOD>` matches `endpoints[].httpMethod` exactly (case-insensitive on input, normalized to upper case for comparison).
- `<path>` is normalized via `normalizePath` before comparison and matched against `endpoints[].path` using the same `{var}` semantics as `findEndpointMatch`. This lets the user pass a literal path like `/orders/42` to select an endpoint declared as `/orders/{id}`.

If a selector matches no endpoint, the Agent records a warning under run metadata (`Selector "<selector>" matched no endpoint`) and continues. If the resulting selected set is empty, the Agent still emits the report and graph but with zero traces and a top-level note.

## Cycle handling — `cycle-detected` marker

The `seen` set on each path prevents infinite recursion through cyclic call graphs (e.g. `service-a → service-b → service-a`).

When `walk` is invoked with `node in seen`, the Agent:

1. Returns a `Truncated` leaf for the current call's child position.
2. Appends the marker `cycle-detected` to the trace's `markers` array.
3. Records the cycle in the rendered sequence diagram as `Note over <node>: cycle-detected` placed over the repeated node.

The branch terminates at the first repeated node; siblings on other branches continue normally because each branch carries its own copy of `seen`.

A trace can carry the `cycle-detected` marker more than once if multiple branches hit different cycles; the marker is appended once per occurrence.

## Configurable max depth — `max-depth-reached` marker

Trace depth is bounded by `maxDepth`, defaulting to `10`. The user may override the default by passing a `--max-depth=<n>` style configuration; the Agent accepts any positive integer.

`depth` increments each time `walk` recurses into another endpoint via a `resolved-service` call. Leaf-producing branches (`opaque-host`, `kafka-topic`, `unresolved`, `resolved-service-no-endpoint-match`) do not increment depth — they terminate the branch.

When `depth >= maxDepth`, the Agent:

1. Returns a `Truncated` leaf at the would-be-next node.
2. Appends the marker `max-depth-reached` to the trace's `markers` array.
3. Records the truncation in the rendered sequence diagram as `Note over <last-node>: max-depth-reached` placed over the deepest visited node.

The depth limit is a guardrail, not a property of the system under test. Reviewers seeing `max-depth-reached` on many traces should consider raising the limit or narrowing the entry-point selection.

## Termination guarantee

The `walk` function terminates for any input because every recursive call falls into one of three terminating cases:

- The depth bound (`depth >= maxDepth`) is hit.
- A previously visited node is encountered (`node in seen`).
- A leaf-producing call status (`opaque-host`, `kafka-topic`, `unresolved`, or `resolved-service-no-endpoint-match`) ends the branch.

There is no path on which the recursion can run unbounded. This is the basis of Property 9 ("Termination — cycles and depth limit") in the design.

## Marker summary

| Marker | Where emitted | Where rendered |
|---|---|---|
| `cycle-detected` | `walk` when `node in seen` | Trace `markers[]`; sequence-diagram `Note over <node>` |
| `max-depth-reached` | `walk` when `depth >= maxDepth` | Trace `markers[]`; sequence-diagram `Note over <last-node>` |
| `match-ambiguous` | `findEndpointMatch` returns >1 endpoint | Each ambiguous child branch's `markers[]`; `(ambiguous)` label on every candidate edge; evidence-table ambiguity row |

Marker rendering details (Mermaid syntax, edge labels, line styles) are owned by `output-format.md`; this document only defines when each marker is emitted.

## Illustrative cases

The two scenarios below are minimal worked examples that demonstrate the termination guarantee in practice. They are not exhaustive test fixtures — `examples.md` deliberately omits these markers and defers to this section.

### Cycle: `service-a → service-b → service-a`

Inputs (abbreviated):

- `service-a` exposes endpoint `E_A = POST /a` and contains an outbound `resolved-service` call to `service-b` `POST /b`.
- `service-b` exposes endpoint `E_B = POST /b` and contains an outbound `resolved-service` call back to `service-a` `POST /a`.
- Entry point selected: `E_A`. `maxDepth = 10`.

Walk:

1. `walk(E_A, seen = {}, depth = 0)` — adds `E_A` to `seen`; matches the call to `E_B`.
2. `walk(E_B, seen = {E_A}, depth = 1)` — adds `E_B` to `seen`; matches the call back to `E_A`.
3. `walk(E_A, seen = {E_A, E_B}, depth = 2)` — `E_A in seen`, returns `Truncated(E_A, marker = "cycle-detected")`.

Result: the trace rooted at `E_A` carries one node for `E_A`, a child for `E_B`, and a truncated leaf for `E_A`. The trace's `markers[]` includes `cycle-detected`. Depth was bounded by the `seen` check, not by `maxDepth`.

### Max-depth: linear chain longer than `maxDepth`

Inputs (abbreviated):

- Eleven services `s0, s1, …, s10`, each exposing endpoint `E_i = GET /step` and (for `i < 10`) containing one `resolved-service` call to `s{i+1}` `GET /step`.
- Entry point selected: `E_0`. `maxDepth = 10`.

Walk: the recursion descends `E_0 → E_1 → … → E_10`, incrementing `depth` on each `resolved-service` hop. When `walk` is entered with `node = E_10` and `depth = 10`, the guard `depth >= maxDepth` fires before the call to `s11` would be expanded.

Result: the trace contains `E_0` through `E_10` as a linear chain; the deepest node carries a `Truncated` leaf with marker `max-depth-reached`. The trace's `markers[]` includes `max-depth-reached`. The `seen` set was never hit because the chain has no repeated nodes.

Together these two cases exercise both terminating branches of the algorithm and confirm Property 9 holds for cyclic and unbounded-depth inputs alike.

## Cross-references

- `target-resolution.md` — owns `resolveCallTarget` and the four `status` outcomes consumed by `walk`.
- `detection-patterns.md` — owns the per-service detection that populates `endpoints[]` and `calls[]`.
- `output-format.md` — owns the rendering of traces and markers in `tracing-report.md`.
- `tracing-graph-schema.md` — owns the `traces[]` and `markers[]` serialization shape in `tracing-graph.json`.
