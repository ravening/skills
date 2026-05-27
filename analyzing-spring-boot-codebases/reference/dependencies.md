# Dependency Review

## Scope

Review platform and library risk across:

- Spring Boot
- Spring Framework modules
- Spring Security
- Jackson
- SnakeYAML
- Tomcat / Jetty / Undertow
- Netty
- Log4j / Logback
- Database drivers
- HTTP clients
- Cloud SDKs
- Build plugins

## Maven checks

Use commands such as:

```bash
mvn -q dependency:tree
mvn -q dependency:list
```

Look for:

- direct dependencies
- risky transitive dependencies
- duplicate versions
- version conflicts
- old Spring Boot parent versions

## Gradle checks

Use commands such as:

```bash
./gradlew dependencies
./gradlew dependencyInsight --dependency spring-boot
```

## Findings format

For each notable dependency, capture:

- package coordinates
- version found
- why it matters
- risk level
- recommended upgrade path

## Risk classification

- P0: confirmed critical exposure or severe unsupported platform risk
- P1: high-risk outdated or vulnerable component
- P2: moderate upgrade or maintainability risk
- P3: low-priority cleanup or alignment issue

## Review notes

Do not claim a CVE applies unless:

1. the dependency version is identified
2. the vulnerable range plausibly matches
3. the project actually includes or uses the affected component
