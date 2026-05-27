# Multi-Repo Input Handling

## Scope

This file tells the Agent how to accept and normalize the set of Service repositories it will analyze: what input shape to expect, how to derive a stable Service name for each repo, how to disambiguate name collisions, and how to record repos that must be skipped. It is the first thing the Agent does in the workflow, before any inbound or outbound detection.

It does not cover detection patterns (see [detection-patterns.md](detection-patterns.md)) or target resolution (see [target-resolution.md](target-resolution.md)).

## Accepted input shape

The Agent accepts **an array of one or more local directory paths**, where each path points at the root of a single Spring Boot Service repository. Both forms are valid:

- **Absolute paths** — e.g. `/Users/alice/work/orders-service`, `C:\repos\orders-service`.
- **Workspace-relative paths** — e.g. `./repos/orders-service`, `../external/payments-service`. Resolve these against the current working directory before any further processing.

Rules:

- Each path must point at a **directory**, not a file or an archive. A path that resolves to a file is treated as `not-readable` (see "Skip handling" below).
- Each path is treated as the root of one Service. The Agent does not recurse into subdirectories looking for additional Spring Boot projects; if a monorepo contains multiple services, the user must supply each service-root path explicitly.
- Duplicate paths in the input list (after normalization to absolute form) are deduplicated silently.
- Symlinks are resolved before path comparison so that two inputs pointing at the same physical directory via different links collapse to one.

The input list is the only way to enumerate Services. The Agent does not autodiscover repos on disk.

## Service name resolution

The Agent must assign one canonical Service name per repo before any merging happens. The name is used as a stable identifier across `tracing-graph.json`, the component diagram, sequence diagrams, and the evidence table.

### Priority order

Consult the following sources **in order**, first non-empty value wins:

1. **`spring.application.name`** from one of the following Spring Boot configuration files, in this order:
   - `src/main/resources/application.yml`
   - `src/main/resources/application.yaml`
   - `src/main/resources/application.properties`

   Profile-specific files (e.g. `application-prod.yml`, `application-dev.properties`) are **not** consulted for naming. The first matching file that contains a non-empty `spring.application.name` value wins.

2. **Build-tool artifact name**:
   - Maven `artifactId` from `pom.xml` (`/project/artifactId`), or
   - Gradle `rootProject.name` from `settings.gradle` or `settings.gradle.kts`.

   If both Maven and Gradle build files are present, prefer Maven (`pom.xml`) since Spring Boot multi-module projects are most often Maven-rooted; record this choice in evidence so a reviewer can audit it.

3. **Repository directory base name** — the final path segment of the resolved absolute repo path.

### Deviation note from requirements

Requirements 2.3 lists the priority as "directory name → spring.application.name → artifactId". The design (and this reference) instead uses **config → build → directory name**, because `spring.application.name` is the canonical service identity in Spring Cloud and service discovery, and is the most stable across renames of either the directory or the build artifact.

This is the **only deviation** from the requirements wording. The skill author or operator should confirm this priority before relying on it for a real run. If the user prefers the requirements wording, fall back to **directory name → `spring.application.name` → artifactId** without further changes; everything else in this skill is unaffected.

Record the resolved name and which source produced it in the run-metadata block of `tracing-report.md` so the choice is auditable.

## Name conflicts

Two or more repos can resolve to the same Service name (for example, two repos that both set `spring.application.name: payments` for different deployment targets, or two directories both named `auth`). When this happens:

- Append `@<repoDirName>` to **every colliding name**, not only the second one. So if `repos/payments-prod` and `repos/payments-staging` both resolve to `payments`, the final names become `payments@payments-prod` and `payments@payments-staging`. This keeps every name visibly disambiguated; readers do not have to remember which repo "won" the original name.
- The disambiguated name is what appears everywhere downstream: `services[].name`, `endpoints[].serviceName`, `calls[].callerServiceName`, diagram node labels, and evidence-table rows.
- Record every conflict under a **"Name conflicts"** subsection in the run-metadata block of `tracing-report.md`, listing the original colliding name and the repos involved. The same data is recorded in `tracing-graph.json` under `nameConflicts[]`.

If a third repo would collide with an already-disambiguated name (e.g. another repo also named `payments-prod` somewhere on disk), apply the same rule again using its own directory base name; the disambiguated form is always `<originalName>@<repoDirName>`.

## Skip handling

A repo that cannot be analyzed must not abort the run. The Agent records the repo as skipped, notes the reason, and continues with the rest. Three reason codes are defined:

| Reason code | When to use |
|---|---|
| `not-found` | The path does not exist on disk after normalization (e.g. typo in the input list, repo not cloned). |
| `not-readable` | The path exists but cannot be read (e.g. permission denied, path resolves to a file rather than a directory, broken symlink). |
| `not-spring-boot` | The path is a readable directory but no Spring Boot dependency or plugin is detected in `pom.xml`, `build.gradle`, or `build.gradle.kts`. |

Behavior for skipped repos:

- The repo is **not** added to `services[]`, **not** scanned for endpoints or calls, and contributes no edges to any diagram.
- One entry per skipped repo is added to `tracing-graph.json` under `skippedRepositories[]` with shape `{ "path": "<input path as given>", "reason": "<reason-code>" }`.
- The same list is rendered under a **"Skipped repositories"** subsection in the run-metadata block of `tracing-report.md`, in the order the repos appeared in the input.
- If a repo is skipped as `not-spring-boot`, also surface it in the report's "Repositories analyzed" list with a `(non-spring-boot, skipped)` suffix so the reader sees that the path was acknowledged, not silently ignored. The corresponding `services[]` entry is **not** created in this case (the entry only exists for repos the Agent actually accepts as Services).

Skip handling is non-fatal by design: a run with five repos where two are `not-found` still produces a valid, partial trace over the remaining three.

## Cross-references

- [SKILL.md](../SKILL.md) — workflow Step 1 invokes this file.
- [detection-patterns.md](detection-patterns.md) — what the Agent does next, per accepted repo.
- [output-format.md](output-format.md) — exact shape of the run-metadata block, including the "Name conflicts" and "Skipped repositories" subsections.
- [tracing-graph-schema.md](tracing-graph-schema.md) — schema for `services[]`, `nameConflicts[]`, and `skippedRepositories[]`.
