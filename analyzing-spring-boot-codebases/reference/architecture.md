# Architecture Mapping

Generate every diagram below only when its **When to generate** rule matches the target codebase. Small applications skip diagrams whose "when to generate" rule does not match — do not emit empty placeholders or "N/A" diagrams.

## What to collect

Capture real names for:

- application entry point
- controllers
- services
- repositories
- entities and DTOs
- security filters and config
- caches
- databases
- message brokers
- HTTP or Feign clients
- scheduled jobs
- async executors
- exception handlers

## Component diagram

Use a Mermaid graph with actual class names.

```mermaid
graph TB
    Client[Client]
    Security[Security Filter Chain]
    Controller[ExampleController]
    Service[ExampleService]
    Repository[ExampleRepository]
    DB[(Database)]
    Cache[(Cache)]
    External[External API]

    Client --> Security
    Security --> Controller
    Controller --> Service
    Service --> Repository
    Repository --> DB
    Service --> Cache
    Service --> External
```

Replace placeholders with discovered components.

## Sequence diagram

Choose one representative endpoint and map:

1. incoming request
2. auth and authorization
3. validation
4. controller entry
5. service orchestration
6. cache lookup if present
7. repository/database call
8. outbound integration if present
9. exception handling
10. final response

```mermaid
sequenceDiagram
    actor Client
    participant Filter as Security Filter
    participant Controller
    participant Service
    participant Repository
    participant DB

    Client->>Filter: HTTP request
    Filter->>Controller: Authenticated request
    Controller->>Service: Validated DTO
    Service->>Repository: Query or command
    Repository->>DB: SQL / persistence call
    DB-->>Repository: Result
    Repository-->>Service: Domain data
    Service-->>Controller: Response DTO
    Controller-->>Client: HTTP response
```

## Package dependency diagram

- **When to generate:** always. Every Spring Boot codebase has packages worth checking for layering.
- **What to capture:** the top-level packages under the application's base package (for example `com.acme.orders.web`, `com.acme.orders.service`, `com.acme.orders.repository`, `com.acme.orders.domain`, `com.acme.orders.config`) and the dependency arrows between them. Surface layering violations such as a controller package depending directly on a repository package, a domain package depending on a web package, or any cycle between packages.
- **Mermaid template:**

```mermaid
graph TB
    Web[com.acme.orders.web]
    Service[com.acme.orders.service]
    Repository[com.acme.orders.repository]
    Domain[com.acme.orders.domain]
    Config[com.acme.orders.config]

    Web --> Service
    Service --> Repository
    Service --> Domain
    Repository --> Domain
    Config --> Service
```

- **Naming rule:** use the actual fully-qualified package names discovered in the target codebase. Do not abbreviate to `web` / `service` / `repo`. If a violating edge exists (for example `web --> repository`), keep it in the diagram so the violation is visible.

## Per-flow sequence diagrams

- **When to generate:** for the top three to five business flows discovered in Step 2 of the workflow. These are in addition to the one representative sequence diagram already required by `SKILL.md`, not a replacement for it.
- **What to capture:** for each selected flow, the same ten-element shape used by the representative sequence diagram (request, auth, validation, controller, service, cache lookup if present, repository or database call, outbound integration if present, exception handling, response). Name the flow after the business action it performs (for example "Place order", "Cancel subscription", "Issue refund"), not after the HTTP route.
- **Mermaid template:**

```mermaid
sequenceDiagram
    actor Client
    participant Filter as Security Filter
    participant Controller as OrderController
    participant Service as OrderService
    participant Repository as OrderRepository
    participant DB as OrdersDB
    participant External as PaymentClient

    Client->>Filter: POST /orders
    Filter->>Controller: Authenticated request
    Controller->>Service: placeOrder(OrderRequest)
    Service->>Repository: save(Order)
    Repository->>DB: INSERT order
    DB-->>Repository: order id
    Service->>External: charge(paymentRef)
    External-->>Service: charge result
    Service-->>Controller: OrderResponse
    Controller-->>Client: 201 Created
```

- **Naming rule:** populate every participant with the real discovered class name (`OrderController`, `OrderService`, `PaymentClient`, etc.) and every message with the real method name or HTTP route. One diagram per flow, each in its own fenced Mermaid block.

## ER / domain model diagram

- **When to generate:** when JPA `@Entity` classes are present in the target codebase.
- **What to capture:** every `@Entity` class as a node, its primary key and notable fields, and the relationships derived from JPA annotations (`@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`). Reflect cardinality and ownership (which side declares `mappedBy`) accurately.
- **Cross-reference:** see [reference/data-layer.md](data-layer.md) for fetch-strategy, cascade, and `open-in-view` review depth that informs how these relationships behave at runtime.
- **Mermaid template:**

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--|{ ORDER_ITEM : contains
    PRODUCT ||--o{ ORDER_ITEM : "is referenced by"

    CUSTOMER {
        UUID id PK
        string email
        string status
    }
    ORDER {
        UUID id PK
        UUID customer_id FK
        string status
        timestamp created_at
    }
    ORDER_ITEM {
        UUID id PK
        UUID order_id FK
        UUID product_id FK
        int quantity
    }
    PRODUCT {
        UUID id PK
        string sku
        string name
    }
```

- **Naming rule:** use the actual discovered entity class names (or table names if `@Table(name = ...)` overrides them) and the actual field names. Do not invent fields that are not present on the entity.

## Async / event flow diagram

- **When to generate:** when `@KafkaListener`, `@RabbitListener`, or `@EventListener` are present in the target codebase.
- **What to capture:** producers, the broker or in-process event bus, the topic / queue / exchange / event-type names, and every consumer. For Spring application events, capture the publisher (`ApplicationEventPublisher`) and each `@EventListener`-annotated method. For Kafka and Rabbit, capture the topic / queue name on the arrow.
- **Cross-reference:** see [reference/observability-and-resilience.md](observability-and-resilience.md) for how trace context, MDC, and correlation IDs propagate across these asynchronous boundaries.
- **Mermaid template:**

```mermaid
graph TB
    Producer[OrderService]
    Broker[(Kafka: orders.events)]
    Consumer1[InvoiceListener]
    Consumer2[NotificationListener]
    Publisher[OrderPublisher]
    EventBus[(Spring ApplicationEventPublisher)]
    EventConsumer[AuditListener]

    Producer -->|OrderPlacedEvent| Broker
    Broker --> Consumer1
    Broker --> Consumer2
    Publisher -->|OrderShippedEvent| EventBus
    EventBus --> EventConsumer
```

- **Naming rule:** use the actual discovered topic / queue / exchange names, the actual event class names, and the actual listener class and method names. Keep arrow labels equal to the message or event type so flows are traceable to source.

## Deployment diagram

- **When to generate:** when `Dockerfile`, `docker-compose.yml`, or Kubernetes manifests (`*.yaml` / `*.yml` under `k8s/`, `deploy/`, `manifests/`, or similar) are present.
- **What to capture:** the runnable units (containers, pods, deployments), how they are exposed (services, ingress, gateway), the data stores and brokers they depend on, and any sidecars or init containers. For Kubernetes, capture namespace, deployment name, service name, and ingress host. For Compose, capture the service name and the published port.
- **Cross-reference:** see [reference/cloud-native.md](cloud-native.md) for Dockerfile hardening, Kubernetes resource limits and probes, and secrets handling depth.
- **Mermaid template:**

```mermaid
graph TB
    Ingress[Ingress: api.example.com]
    Service[Service: order-service]
    Deployment[Deployment: order-service]
    Pod1[Pod: order-service-xxxx]
    Pod2[Pod: order-service-yyyy]
    DB[(Postgres: orders-db)]
    Cache[(Redis: orders-cache)]
    Broker[(Kafka)]

    Ingress --> Service
    Service --> Deployment
    Deployment --> Pod1
    Deployment --> Pod2
    Pod1 --> DB
    Pod1 --> Cache
    Pod1 --> Broker
    Pod2 --> DB
    Pod2 --> Cache
    Pod2 --> Broker
```

- **Naming rule:** use the actual deployment, service, ingress, namespace, and host names from the manifests. For Compose, use the actual service names from `docker-compose.yml`. Do not invent infrastructure that is not declared.

## State diagram per entity

- **When to generate:** when an entity has an explicit lifecycle or status enum (for example `OrderStatus`, `PaymentState`, `SubscriptionStatus`). Generate one diagram per such entity.
- **What to capture:** every enum value as a state, every legal transition as an arrow labelled with the action or event that causes it, and the initial and terminal states. Derive transitions from the service / domain code that mutates the status field, not from speculation.
- **Mermaid template:**

```mermaid
stateDiagram-v2
    [*] --> CREATED
    CREATED --> CONFIRMED: confirm()
    CREATED --> CANCELLED: cancel()
    CONFIRMED --> SHIPPED: ship()
    CONFIRMED --> CANCELLED: cancel()
    SHIPPED --> DELIVERED: deliver()
    DELIVERED --> [*]
    CANCELLED --> [*]
```

- **Naming rule:** use the actual enum value names (matching case) and the actual method or event names that drive the transition. One diagram per entity, with the entity class name as the section heading.

## Auth / security flow diagram

- **When to generate:** when OAuth2, OIDC, or JWT are in use (for example `spring-boot-starter-oauth2-resource-server`, `spring-boot-starter-oauth2-client`, or a custom JWT filter is present).
- **What to capture:** the client, the identity provider or authorization server, the resource server (the Spring Boot app), the security filter chain, and the token lifecycle (issue, validate, refresh). Show where tokens are validated (signature, `iss`, `aud`, `exp`) and where authorization decisions are made.
- **Cross-reference:** see [reference/security.md](security.md) for JWT pitfalls, OAuth2 / OIDC flow review, and session management depth.
- **Mermaid template:**

```mermaid
sequenceDiagram
    actor User
    participant Client
    participant IdP as Authorization Server
    participant App as Spring Resource Server
    participant Filter as JwtAuthenticationFilter
    participant Controller

    User->>Client: Initiate login
    Client->>IdP: Authorization code request (PKCE)
    IdP-->>Client: Authorization code
    Client->>IdP: Exchange code for tokens
    IdP-->>Client: id_token + access_token + refresh_token
    Client->>App: Request with Bearer access_token
    App->>Filter: Extract and validate JWT
    Filter->>Filter: Verify signature, iss, aud, exp
    Filter->>Controller: Authenticated principal
    Controller-->>Client: Protected resource
```

- **Naming rule:** use the actual filter class name, the actual authorization server hostname or issuer URL, and the actual scopes or audiences declared in configuration. If the codebase uses opaque tokens with introspection rather than JWTs, replace the local validation step with the introspection call to the authorization server.

## Diagram rules

- Use actual discovered class names
- Keep arrows directional and readable
- Show only important components
- Prefer one clear diagram over one huge unreadable diagram
