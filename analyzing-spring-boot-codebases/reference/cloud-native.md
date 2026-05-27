# Cloud-Native Review

## Scope

This file covers containerization and orchestration review for Spring Boot services: Dockerfile hardening, Kubernetes manifest review, secrets management, and service mesh basics. The guidance is explicitly cloud-agnostic — it applies the same way on AWS, GCP, Azure, on-premises Kubernetes, or any other target. It does NOT cover application-level security controls, generic resilience patterns, or deployment topology diagrams.

Cross-references:

- [reference/security.md](security.md) for secrets handling depth, including how application-level auth and credential review interact with the platform-level secrets posture described here.
- [reference/architecture.md](architecture.md) for the deployment diagram subsection, including when to generate a deployment diagram from `Dockerfile`, `docker-compose.yml`, and Kubernetes manifests.
- [reference/azure-resilience.md](azure-resilience.md) for Azure-specific deployment, identity, and service-binding specifics. This file does NOT absorb Azure content; readers go to `azure-resilience.md` for Azure depth.

## Review areas

Assess these categories:

- Dockerfile hardening (non-root execution, multi-stage builds, base image choice, no secrets in image layers)
- Kubernetes manifest review (resource limits, probes, `securityContext`, network policies)
- secrets management (delivery mechanism, rotation, blast radius)
- service mesh basics (only when a service mesh is present in the target codebase)

## Dockerfile hardening

Inspect every `Dockerfile` in the target codebase for the following posture.

**Non-root execution.** The container should run as a dedicated, unprivileged user. Look for an explicit `USER` instruction near the end of the Dockerfile that points at a non-root account created with a fixed UID and GID. The default `root` user is the bad case — it gives a compromised process the ability to write anywhere in the container filesystem, install packages, and exploit kernel surface that user namespaces would otherwise block. Spring Boot images that inherit from generic `eclipse-temurin` or `openjdk` bases without an added `USER` line are a common offender. Record the file path and the relevant line numbers, explain the risk, and recommend adding a `USER` directive backed by a `RUN` line that creates the account.

**Multi-stage builds.** The build stage and the runtime stage should be separate. The build stage owns Maven or Gradle, the JDK, source code, and any build-time secrets such as private artifact registry credentials. The runtime stage owns only the final fat JAR (or layered JAR output) and a JRE. The bad case is a single-stage Dockerfile that ships the full JDK, the Maven/Gradle wrapper, the local repository cache, and the entire source tree into production. That bloats the image, expands the attack surface, and risks leaking build-time files. Flag any Dockerfile without a `FROM ... AS build` stage followed by a separate `FROM` for runtime, with a `COPY --from=build` of just the artifact.

**Distroless or minimal base images.** Prefer a base image that contains only what is needed to run a Java process. Good choices conceptually are distroless Java images, minimal JRE images, or hardened slim variants. The bad case is a full-OS base image with shells, package managers, debugging tools, and unrelated language runtimes baked in — every one of those is a tool a successful attacker can use without bringing their own. Note the chosen base image and recommend a smaller alternative when the current image is general-purpose.

**No secrets in image layers.** Image layers are immutable and shareable. A secret introduced in any intermediate layer is recoverable from the image even if a later layer overwrites or deletes it. Watch for `ENV` lines containing tokens, passwords, or connection strings; `ARG` values used as long-lived secrets rather than build-only inputs; `COPY` of `.env`, `application-prod.yml` with embedded credentials, private keys, or `kubeconfig`; and `RUN` lines that `echo` a secret into a file. Recommend BuildKit secret mounts for build-time-only credentials and platform-level secret delivery (Kubernetes Secrets, External Secrets Operator, cloud secret managers) for runtime credentials.

## Kubernetes manifest review

Inspect every Kubernetes manifest in the target codebase — typically under `k8s/`, `manifests/`, `helm/` (chart templates), or `kustomize/` overlays — for the following posture.

**Resource limits.** Every container in every Pod should declare both `requests` and `limits` for CPU and memory. The Spring Boot JVM is particularly sensitive to memory misconfiguration: without a memory limit the JVM may grow until the node OOM-kills the Pod, and without a CPU request the scheduler cannot place the Pod on a node that can actually run it. The bad case is a `Deployment`, `StatefulSet`, or `Job` whose `containers[].resources` block is empty or missing. Flag every workload with absent or one-sided (only requests, only limits) resource declarations and recommend setting both, sized to the Spring Boot service's measured footprint.

**Liveness, readiness, and startup probes.** A Spring Boot service should expose all three probes against the actuator health endpoints and have them wired into the Pod spec. Readiness gates traffic — if it is missing, the Service routes requests to the Pod before the application context finishes loading, producing 5xx during rollouts. Liveness restarts a wedged process — if it is missing, a deadlocked Pod stays in the load balancer indefinitely. Startup is necessary for slow-starting Spring Boot applications (large schema validation, large dependency graphs, long classpath scanning) so liveness does not kill the Pod before it finishes booting. Flag any Pod spec missing readiness, missing liveness, or relying on a single probe to do the work of three. Record the manifest path, the missing probe(s), and recommend pointing at `/actuator/health/readiness`, `/actuator/health/liveness`, and an appropriate startup probe.

**`securityContext`.** Both the Pod-level and container-level `securityContext` should be set. At minimum: `runAsNonRoot: true`, an explicit `runAsUser` matching the Dockerfile's `USER`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, and dropped Linux capabilities (`drop: ["ALL"]`, adding back only what is genuinely needed). The bad case is no `securityContext` at all, which lets the container run as root, write the root filesystem, escalate privileges, and inherit every default capability. Flag missing or weak `securityContext` blocks and recommend the hardened defaults.

**Network policies.** A `NetworkPolicy` should constrain which Pods can talk to the Spring Boot service and which destinations the service can reach. The bad case is a namespace with no `NetworkPolicy` resources, which means every Pod can reach every other Pod and every external endpoint by default — the exact posture an attacker needs after compromising any single workload. Flag namespaces without ingress and egress policies and recommend a default-deny policy plus explicit allow rules for the dependencies discovered during architecture mapping (database, cache, message broker, downstream services, identity provider).

## Secrets management

Inspect how the Spring Boot service receives its credentials and other sensitive configuration.

**The environment-variable anti-pattern.** Plain environment variables — whether set via Kubernetes `env:` literals, `envFrom: configMapRef`, or a `.env` file mounted into the container — are not a secrets mechanism. They are visible to any process inside the container, appear in `ps` output and `/proc/<pid>/environ`, leak into crash dumps and error pages, and frequently end up in observability pipelines (logs, traces, metrics labels) when the application or a sidecar prints its environment for diagnostics. The bad case is a `Deployment` that injects a database password, JWT signing key, or API token directly under `env:` as a literal value or from a `ConfigMap`. Flag every such occurrence, record the manifest path and the offending environment variable name (not its value), and recommend moving the value to a real secret store.

**HashiCorp Vault.** When Vault is in use, look for sidecar injection annotations or the Vault Agent pattern, a defined path layout, lease and renewal handling on the application side, and rotation behavior on credential expiry. The good case is short-lived dynamic credentials (database, cloud provider) issued per-Pod with a sensible TTL. The bad case is a long-lived static secret stored at a Vault path and treated as if it never expires — that is the environment-variable anti-pattern with extra steps. Record the Vault path, the lease behavior, and recommend dynamic credentials where the underlying system supports them.

**External Secrets Operator.** When ESO is in use, look for `ExternalSecret` resources mapped to a `SecretStore` or `ClusterSecretStore`, the refresh interval, and the target `Secret` shape. The good case is a short refresh interval that picks up rotations promptly without a Pod restart, and a `SecretStore` scoped to a single namespace or backend. The bad case is a multi-hour refresh interval that masks rotation failures, or a cluster-wide `ClusterSecretStore` that grants every namespace access to every secret. Record the resource path, the refresh interval, and the backend reference.

**AWS, GCP, and Azure secret managers.** When the service pulls secrets directly from a cloud secret manager (AWS Secrets Manager or Parameter Store, GCP Secret Manager, Azure Key Vault), check that the workload identity binding is per-service rather than a shared node-level role, that the IAM policy is scoped to the specific secret resources rather than wildcarded, that the application caches the secret with a bounded TTL rather than refetching on every request, and that the application handles rotation without requiring a manual restart. For Azure Key Vault depth — Managed Identity token failures, Key Vault throttling, secret-rotation handling against a Spring Boot `@RefreshScope` bean — defer to [reference/azure-resilience.md](azure-resilience.md). For application-level review of how the fetched credentials are then used (auth headers, signing keys, DB passwords) defer to [reference/security.md](security.md).

## Service mesh basics

This guidance applies only when a service mesh is present in the target codebase. Detect it by looking for Istio CRDs (`VirtualService`, `DestinationRule`, `Gateway`, `PeerAuthentication`, `AuthorizationPolicy`), Linkerd annotations (`linkerd.io/inject: enabled`), Consul Connect sidecar annotations, or AWS App Mesh / Azure Service Mesh resources. If no mesh artifacts are present, skip this section entirely.

**mTLS posture.** This guidance applies only when a service mesh is present in the target codebase. Confirm that mesh-wide mutual TLS is set to `STRICT` for the namespace the Spring Boot service runs in, not `PERMISSIVE`. Permissive mode accepts plaintext alongside mTLS and is intended only as a migration state — leaving it permissive in production means an attacker who lands a Pod inside the mesh can call any service in cleartext. Record the `PeerAuthentication` (or equivalent) resource path and the current mode.

**Authorization policies.** This guidance applies only when a service mesh is present in the target codebase. The mesh should enforce service-to-service authorization rather than relying solely on application-layer checks. Look for `AuthorizationPolicy` resources (Istio) or equivalents that express which workloads may call the Spring Boot service, on which paths, and with which methods. The bad case is a mesh-enabled namespace with no authorization policies — the mesh provides identity and encryption but no access control, and any Pod in the mesh can call any endpoint. Record the policy paths, the principals allowed, and any overly broad rules (wildcarded principals, wildcarded paths).

**Traffic management and retries.** This guidance applies only when a service mesh is present in the target codebase. Mesh-level retries, timeouts, and circuit breakers can interact poorly with application-level resilience configured in Resilience4j or in HTTP client libraries — the same call may be retried at both layers, multiplying load on a degraded dependency. Confirm there is one owner per concern: mesh-level for cross-cutting defaults, application-level for business-aware behavior. Cross-reference [reference/observability-and-resilience.md](observability-and-resilience.md) for application-level resilience pattern depth.

**Sidecar resource sizing.** This guidance applies only when a service mesh is present in the target codebase. The injected proxy sidecar (Envoy for Istio, the Linkerd proxy, etc.) needs its own CPU and memory requests and limits. The bad case is a Pod whose application container is sized correctly but whose sidecar inherits cluster-wide defaults that are too small under load — the proxy throttles, latency rises, and the symptom looks like an application-layer slowdown. Record the sidecar resource block (or its absence) and recommend explicit values matched to the service's traffic profile.

## Severity guidance

### P0

- Production Dockerfile that runs as `root` and ships a JAR exposed on the public internet
- Secrets baked into image layers (passwords, signing keys, private certificates, cloud provider tokens)
- Kubernetes workload running with no `securityContext`, `runAsNonRoot: false` (or unset), and `allowPrivilegeEscalation: true`
- Database password, JWT signing key, or cloud provider token delivered as a plain environment variable from a `ConfigMap` or literal `env:` value in production
- Service mesh in `PERMISSIVE` mTLS mode in production with no compensating network policy

### P1

- Single-stage Dockerfile shipping the full JDK and source tree to production
- Pod spec missing readiness or liveness probes, causing 5xx during rollouts or wedged Pods
- Containers without resource `requests` and `limits`, risking OOM kills and noisy-neighbor effects
- Namespace with no `NetworkPolicy` resources (default-allow posture)
- Long-lived static secret stored in Vault or a cloud secret manager and treated as non-rotating
- Mesh-enabled namespace with no `AuthorizationPolicy` (or equivalent) resources

### P2

- General-purpose base image (full OS) where a distroless or minimal JRE base would work
- Missing startup probe on a Spring Boot service known to start slowly, causing premature liveness restarts
- `securityContext` present but missing `readOnlyRootFilesystem`, capability drops, or an explicit `runAsUser`
- External Secrets Operator refresh interval long enough to mask rotation failures
- Mesh sidecar relying on cluster-default resource sizing under non-trivial traffic

### P3

- Hardening polish: tighter capability drops, smaller base image variants, more granular network policies
- Documentation gaps describing secret rotation procedures or mesh policy intent
- Observability of platform-level posture (e.g., emitting probe-failure metrics)

## Evidence rules

Always include:

- file path (Dockerfile, manifest, Helm template, Kustomize overlay, or mesh CRD)
- line number when possible
- why the issue matters (impact at the platform layer, blast radius if exploited or triggered)
- exact remediation (the specific directive, field, or resource to add or change)
