# BuyZen — Production-Grade E-Commerce Microservices

**Package architecture & repository blueprint**
Base package: `com.sm.buyzen` · Build: Maven multi-module monorepo · Stack: Spring Boot 3.x / Spring Cloud, Kafka, Keycloak (OAuth2/OIDC), AWS, PostgreSQL, Redis, OpenSearch, AI (embeddings + LLM).

This document is the complete, "nothing-missing" package and module layout for the platform. It is organized so you can read it top-down: repo root → shared libraries → platform (infra) services → a reusable per-service blueprint → every business service in full → infrastructure-as-code → catalogs (Kafka topics, databases, ports) → cross-cutting patterns.

---

## 1. Technology stack (what lives where)

| Concern | Technology | Where it appears in the tree |
|---|---|---|
| Runtime / framework | Java 21, Spring Boot 3.x | every service |
| Service mesh basics | Spring Cloud Gateway, Netflix Eureka, Spring Cloud Config | `buyzen-platform/*` |
| Inter-service (sync) | OpenFeign / Spring `RestClient` + Resilience4j | `client/` package per service |
| Inter-service (async) | Apache Kafka + Schema Registry (Avro) | `messaging/` package + `buyzen-contracts` |
| Identity & access | Keycloak (OIDC), OAuth2 Resource Server, JWT | `buyzen-common-security`, gateway, `identity-service` |
| Persistence | PostgreSQL (per service), Flyway migrations | `domain/` + `resources/db/migration` |
| Cache / session | Redis (ElastiCache) | `cart-service`, gateway rate-limit |
| Search | OpenSearch / Elasticsearch | `search-service` |
| AI | AWS Bedrock (embeddings + LLM), Comprehend, pgvector | `recommendation`, `review`, `search` services |
| Cloud | AWS (ECS/EKS, RDS, MSK, ElastiCache, S3, CloudFront, SES/SNS, Secrets Manager) | `infra/` |
| Observability | Micrometer + Prometheus, Grafana, OpenTelemetry/Tempo, Loki | `buyzen-common-observability` |
| Resilience | Resilience4j (circuit breaker, retry, bulkhead, rate limiter) | `buyzen-common-web`, per-service `config/` |
| Build / CI | Maven, Docker, GitHub Actions, Testcontainers | root, `.github/`, `src/test` |
| IaC | Terraform + Helm/Kustomize | `infra/` |

## 2. Legend & conventions

**Reading the trees.** `dir/` is a folder/package. Files show their extension. `…` means "and siblings of the same kind." Where a business service reuses the canonical blueprint (Section 8) verbatim, it is marked *(template)* rather than repeated line-by-line.

**Naming conventions**

- **Maven groupId:** `com.sm.buyzen` for every module.
- **Maven artifactId / module dir:** `buyzen-<area>-<name>` (e.g. `buyzen-order-service`, `buyzen-common-kafka`).
- **Java root package:** `com.sm.buyzen.<serviceShortName>` (e.g. `com.sm.buyzen.order`). Shared libs use `com.sm.buyzen.common.<lib>`.
- **REST paths:** `/api/v{n}/<resource>` — version in the path, plural nouns.
- **Kafka topics:** `buyzen.<domain>.<event>.v{n}` (e.g. `buyzen.order.created.v1`); DLQ = `<topic>.dlt`.
- **Database:** one schema/DB per service, named `buyzen_<service>` (e.g. `buyzen_order`). No cross-service DB access — ever.
- **Classes:** `*Controller`, `*Service`/`*ServiceImpl`, `*Repository`, `*Entity` (or plain domain names), `*Request`/`*Response` DTOs, `*Mapper` (MapStruct), `*Producer`/`*Consumer`, `*Client`, `*Properties`, `*Exception`.

**Per-service parent.** Every module declares the root `buyzen` `pom.xml` as its `<parent>` (dependency & plugin management). The intermediate `buyzen-common/`, `buyzen-platform/`, `buyzen-services/` folders are Maven **aggregator** POMs (packaging `pom`) that only list `<modules>` — they group the tree, they do not manage dependencies.

---

## 3. Repository root

```text
buyzen/
├── pom.xml                          # root parent + aggregator (dependencyManagement, pluginManagement, <modules>)
├── mvnw  /  mvnw.cmd                # Maven wrapper
├── .mvn/
│   └── wrapper/maven-wrapper.properties
├── README.md
├── CONTRIBUTING.md
├── CHANGELOG.md
├── LICENSE
├── .gitignore
├── .gitattributes
├── .editorconfig
├── .dockerignore
├── .env.example                     # local env vars for docker-compose
├── Makefile                         # make up / down / build / test / lint shortcuts
├── docker-compose.yml               # local: kafka, zookeeper/kraft, schema-registry, keycloak, postgres, redis, opensearch, prometheus, grafana, tempo, loki, localstack
├── docker-compose.infra.yml         # split file: just the backing services
├── lombok.config
├── checkstyle.xml  /  spotbugs-exclude.xml   # static analysis config
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                   # build + test + static analysis on PR
│   │   ├── release.yml              # tag → build+push images → deploy
│   │   ├── security-scan.yml        # dependency + container CVE scan (Trivy/OWASP)
│   │   └── codeql.yml
│   ├── CODEOWNERS
│   ├── dependabot.yml
│   └── pull_request_template.md
│
├── buyzen-common/                   # shared libraries (aggregator pom)  → Section 5
│   ├── pom.xml
│   ├── buyzen-common-core/
│   ├── buyzen-common-web/
│   ├── buyzen-common-security/
│   ├── buyzen-common-kafka/
│   ├── buyzen-common-data/
│   ├── buyzen-common-observability/
│   ├── buyzen-common-aws/
│   └── buyzen-common-test/
│
├── buyzen-contracts/                # versioned event/API contracts (Avro/OpenAPI) → Section 6
│   ├── pom.xml
│   └── src/main/...
│
├── buyzen-platform/                 # infrastructure services (aggregator pom)  → Section 7
│   ├── pom.xml
│   ├── buyzen-config-server/
│   ├── buyzen-discovery-server/
│   └── buyzen-api-gateway/
│
├── buyzen-services/                 # business microservices (aggregator pom)  → Sections 8–9
│   ├── pom.xml
│   ├── buyzen-identity-service/
│   ├── buyzen-product-service/
│   ├── buyzen-inventory-service/
│   ├── buyzen-cart-service/
│   ├── buyzen-order-service/
│   ├── buyzen-payment-service/
│   ├── buyzen-shipping-service/
│   ├── buyzen-recommendation-service/
│   ├── buyzen-review-service/
│   ├── buyzen-search-service/
│   ├── buyzen-notification-service/
│   └── buyzen-media-service/
│
├── infra/                           # NOT Maven modules — infrastructure as code → Section 10
│   ├── terraform/
│   ├── k8s/
│   ├── helm/
│   ├── keycloak/
│   ├── monitoring/
│   └── scripts/
│
└── docs/
    ├── architecture/
    │   ├── c4-context.md
    │   ├── c4-container.md
    │   ├── adr/                      # Architecture Decision Records: 0001-record-architecture-decisions.md …
    │   └── diagrams/
    ├── api/                          # aggregated OpenAPI specs
    ├── runbooks/                     # on-call runbooks per service
    └── event-catalog.md             # source of truth for Kafka topics & schemas
```

### 3.1 Root `pom.xml` responsibilities

The root POM is both parent and aggregator. It pins Spring Boot / Spring Cloud BOMs, declares shared dependency versions in `<dependencyManagement>`, configures shared plugins in `<pluginManagement>` (compiler, surefire/failsafe, jacoco, spotless/checkstyle, spring-boot, jib/buildpacks for images, flyway, mapstruct/lombok processors), and lists every module:

```xml
<modules>
  <module>buyzen-common</module>
  <module>buyzen-contracts</module>
  <module>buyzen-platform</module>
  <module>buyzen-services</module>
</modules>
```

---

## 4. How the layers depend on each other

```text
buyzen-contracts  ─────────────┐        (event schemas; no Spring deps)
                               ▼
buyzen-common-core ─► buyzen-common-web ─► buyzen-common-security
        │                    │                     │
        ├──► buyzen-common-data                     │
        ├──► buyzen-common-kafka ◄──────────────────┘  (consumes contracts)
        ├──► buyzen-common-observability
        └──► buyzen-common-aws
                               ▼
        buyzen-platform/*  and  buyzen-services/*      (depend on the commons they need)
                               ▲
        buyzen-common-test  (test scope only, everywhere)
```

Rule: **business services depend on commons; commons never depend on services; no service depends on another service's code** (only on `buyzen-contracts` for event shapes and on other services' HTTP APIs at runtime).

---

## 5. Shared libraries — `buyzen-common/`

Each is a normal jar (not a Spring Boot app). Auto-configuration is exposed via `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` so that simply adding the dependency wires the behavior in.

### 5.1 `buyzen-common-core` — DTO/error/utility primitives

```text
buyzen-common-core/
├── pom.xml
└── src/main/java/com/sm/buyzen/common/core/
    ├── api/
    │   ├── ApiResponse.java                 # envelope: data + meta
    │   ├── ApiError.java                     # RFC-7807 ProblemDetail extension
    │   ├── ErrorCode.java                    # enum of stable machine codes
    │   ├── PageResponse.java                 # paging envelope
    │   └── ValidationError.java
    ├── exception/
    │   ├── BuyzenException.java              # base runtime exception
    │   ├── BusinessException.java
    │   ├── ResourceNotFoundException.java
    │   ├── ConflictException.java
    │   ├── UnauthorizedException.java
    │   ├── ForbiddenException.java
    │   └── ExternalServiceException.java
    ├── domain/
    │   ├── Money.java                        # value object (amount + currency)
    │   ├── Currency.java
    │   ├── Quantity.java
    │   └── Identifier.java
    ├── constant/
    │   ├── HeaderNames.java                  # X-Correlation-Id, X-User-Id, X-Tenant-Id…
    │   ├── AppConstants.java
    │   └── DateTimeFormats.java
    ├── util/
    │   ├── IdGenerator.java                  # ULID/UUIDv7
    │   ├── JsonUtils.java
    │   ├── StringUtils.java
    │   ├── DateTimeUtils.java
    │   └── MoneyUtils.java
    └── i18n/
        └── MessageKeys.java
```

### 5.2 `buyzen-common-web` — REST cross-cutting concerns

```text
buyzen-common-web/
└── src/main/java/com/sm/buyzen/common/web/
    ├── advice/
    │   └── GlobalExceptionHandler.java       # @RestControllerAdvice → ProblemDetail
    ├── filter/
    │   ├── CorrelationIdFilter.java          # generate/propagate X-Correlation-Id
    │   ├── RequestLoggingFilter.java
    │   └── MdcContextFilter.java             # push user/correlation into SLF4J MDC
    ├── interceptor/
    │   └── ApiVersionInterceptor.java
    ├── config/
    │   ├── WebMvcCommonConfig.java
    │   ├── JacksonConfig.java                # ISO dates, Money serializer, snake/camel
    │   ├── CorsProperties.java
    │   └── OpenApiConfig.java                # springdoc base config + security scheme
    ├── resilience/
    │   └── Resilience4jDefaults.java
    ├── pagination/
    │   └── PageableArgumentConfig.java
    └── autoconfigure/
        └── WebAutoConfiguration.java
    └── src/main/resources/META-INF/spring/…AutoConfiguration.imports
```

### 5.3 `buyzen-common-security` — OAuth2 / Keycloak

```text
buyzen-common-security/
└── src/main/java/com/sm/buyzen/common/security/
    ├── config/
    │   ├── ResourceServerSecurityConfig.java # JWT resource-server defaults
    │   └── MethodSecurityConfig.java         # @EnableMethodSecurity
    ├── jwt/
    │   ├── KeycloakJwtAuthenticationConverter.java   # realm/client roles → authorities
    │   ├── JwtRoleConverter.java
    │   └── AudienceValidator.java
    ├── context/
    │   ├── SecurityContextUtils.java         # current userId, roles, token
    │   └── AuthenticatedUser.java
    ├── annotation/
    │   ├── CurrentUser.java                  # @CurrentUser arg resolver
    │   └── RequiresRole.java
    ├── properties/
    │   └── KeycloakProperties.java           # issuer-uri, jwk-set-uri, audience
    └── autoconfigure/
        └── SecurityAutoConfiguration.java
```

### 5.4 `buyzen-common-kafka` — messaging plumbing

```text
buyzen-common-kafka/
└── src/main/java/com/sm/buyzen/common/kafka/
    ├── config/
    │   ├── KafkaProducerConfig.java
    │   ├── KafkaConsumerConfig.java
    │   ├── KafkaErrorHandlerConfig.java      # DefaultErrorHandler + backoff
    │   ├── SchemaRegistryConfig.java         # Avro serde
    │   └── KafkaTopicsProperties.java
    ├── event/
    │   ├── DomainEvent.java                  # base: eventId, occurredAt, correlationId, type, version
    │   ├── EventEnvelope.java
    │   └── EventType.java
    ├── producer/
    │   ├── EventPublisher.java               # interface
    │   └── KafkaEventPublisher.java
    ├── consumer/
    │   ├── AbstractEventConsumer.java
    │   └── IdempotentConsumerSupport.java    # dedupe via processed-event table
    ├── outbox/
    │   ├── OutboxEvent.java                  # transactional-outbox entity
    │   ├── OutboxRepository.java
    │   ├── OutboxPublisher.java              # polling/Debezium-friendly relay
    │   └── OutboxProperties.java
    ├── dlt/
    │   └── DeadLetterPublisher.java
    ├── tracing/
    │   └── KafkaTracingConfig.java           # W3C traceparent propagation
    └── autoconfigure/
        └── KafkaAutoConfiguration.java
```

### 5.5 `buyzen-common-data` — persistence defaults

```text
buyzen-common-data/
└── src/main/java/com/sm/buyzen/common/data/
    ├── entity/
    │   ├── BaseEntity.java                   # @Id (ULID), @Version
    │   └── AuditableEntity.java              # createdAt/By, updatedAt/By, deleted (soft delete)
    ├── audit/
    │   ├── JpaAuditingConfig.java
    │   └── SecurityAuditorAware.java         # current user from JWT
    ├── converter/
    │   ├── MoneyConverter.java
    │   └── JsonbConverter.java               # Postgres jsonb <-> object
    ├── config/
    │   ├── JpaCommonConfig.java
    │   ├── FlywayConfig.java
    │   └── DataSourceProperties.java
    ├── repository/
    │   └── BaseRepository.java               # common query fragments, soft-delete
    └── autoconfigure/
        └── DataAutoConfiguration.java
```

### 5.6 `buyzen-common-observability` — logs / metrics / traces

```text
buyzen-common-observability/
└── src/main/java/com/sm/buyzen/common/observability/
    ├── logging/
    │   ├── LoggingConfig.java
    │   └── StructuredLogEncoderConfig.java   # JSON logs → Loki
    ├── metrics/
    │   ├── MetricsConfig.java                # Micrometer + Prometheus registry
    │   └── CommonTags.java                   # service, env, version tags
    ├── tracing/
    │   └── OpenTelemetryConfig.java          # OTLP exporter → Tempo
    ├── health/
    │   └── CustomHealthIndicators.java
    └── autoconfigure/
        └── ObservabilityAutoConfiguration.java
    └── src/main/resources/logback-spring.xml
```

### 5.7 `buyzen-common-aws` — cloud SDK wrappers

```text
buyzen-common-aws/
└── src/main/java/com/sm/buyzen/common/aws/
    ├── config/
    │   ├── AwsCredentialsConfig.java         # default provider chain / IRSA
    │   ├── AwsRegionProperties.java
    │   └── LocalstackConfig.java             # @Profile("local")
    ├── s3/
    │   ├── S3StorageService.java
    │   ├── PresignedUrlService.java
    │   └── S3Properties.java
    ├── ses/
    │   └── SesEmailSender.java
    ├── sns/
    │   └── SnsPublisher.java
    ├── secrets/
    │   └── SecretsManagerPropertySource.java
    ├── bedrock/
    │   ├── BedrockClientConfig.java
    │   ├── EmbeddingClient.java              # Titan/Cohere embeddings
    │   └── ChatModelClient.java              # LLM invoke (Claude on Bedrock)
    ├── comprehend/
    │   └── ComprehendClient.java             # sentiment / PII detection
    └── autoconfigure/
        └── AwsAutoConfiguration.java
```

### 5.8 `buyzen-common-test` — test scaffolding

```text
buyzen-common-test/
└── src/main/java/com/sm/buyzen/common/test/
    ├── containers/
    │   ├── PostgresTestContainer.java
    │   ├── KafkaTestContainer.java
    │   ├── RedisTestContainer.java
    │   ├── KeycloakTestContainer.java
    │   ├── OpenSearchTestContainer.java
    │   └── LocalstackTestContainer.java
    ├── AbstractIntegrationTest.java          # @SpringBootTest + containers
    ├── fixtures/
    │   └── TestDataFactory.java
    ├── security/
    │   └── WithMockJwt.java                  # test annotation for authenticated calls
    └── kafka/
        └── KafkaTestConsumer.java
```

---

## 6. Event & API contracts — `buyzen-contracts`

Single source of truth for what crosses the wire. Avro schemas compile to Java at build time; OpenAPI specs are published for consumers. This module has **no Spring dependency** so any service can use it.

```text
buyzen-contracts/
├── pom.xml                                   # avro-maven-plugin + openapi-generator
└── src/main/
    ├── avro/                                 # Kafka event schemas (→ generated Java)
    │   ├── common/
    │   │   ├── EventMetadata.avsc            # eventId, occurredAt, correlationId, source, version
    │   │   └── MoneyRecord.avsc
    │   ├── order/
    │   │   ├── OrderCreated.avsc
    │   │   ├── OrderConfirmed.avsc
    │   │   ├── OrderCancelled.avsc
    │   │   └── OrderCompleted.avsc
    │   ├── payment/
    │   │   ├── PaymentRequested.avsc
    │   │   ├── PaymentCompleted.avsc
    │   │   ├── PaymentFailed.avsc
    │   │   └── RefundProcessed.avsc
    │   ├── inventory/
    │   │   ├── StockReserved.avsc
    │   │   ├── StockReservationFailed.avsc
    │   │   └── StockReleased.avsc
    │   ├── shipping/
    │   │   ├── ShipmentCreated.avsc
    │   │   ├── ShipmentDispatched.avsc
    │   │   └── ShipmentDelivered.avsc
    │   ├── product/
    │   │   ├── ProductCreated.avsc
    │   │   ├── ProductUpdated.avsc
    │   │   ├── PriceChanged.avsc
    │   │   └── ProductDeleted.avsc
    │   ├── review/
    │   │   └── ReviewCreated.avsc
    │   ├── media/
    │   │   └── MediaUploaded.avsc
    │   └── user/
    │       ├── UserRegistered.avsc
    │       └── UserUpdated.avsc
    └── resources/openapi/                    # published REST contracts
        ├── product-api.yaml
        ├── order-api.yaml
        └── … (one per service, kept in sync via CI)
```

---

## 7. Platform (infrastructure) services — `buyzen-platform/`

### 7.1 `buyzen-config-server` — centralized configuration

```text
buyzen-config-server/
├── pom.xml
├── Dockerfile
└── src/main/
    ├── java/com/sm/buyzen/config/
    │   ├── ConfigServerApplication.java      # @EnableConfigServer
    │   ├── config/
    │   │   ├── ConfigSecurityConfig.java     # secured; only services can pull
    │   │   └── EncryptionConfig.java         # {cipher} value support
    │   └── health/
    │       └── ConfigRepoHealthIndicator.java
    └── resources/
        ├── application.yml                   # git/native backend, refresh
        └── bootstrap.yml
# Config files themselves live in a separate git repo (config-repo/): application.yml, <service>-<profile>.yml
```

### 7.2 `buyzen-discovery-server` — service registry

```text
buyzen-discovery-server/
├── pom.xml
├── Dockerfile
└── src/main/
    ├── java/com/sm/buyzen/discovery/
    │   ├── DiscoveryServerApplication.java   # @EnableEurekaServer
    │   └── config/
    │       └── EurekaSecurityConfig.java
    └── resources/
        └── application.yml                   # peer-aware, self-preservation tuned
```

### 7.3 `buyzen-api-gateway` — edge / BFF

The single public entry point: routing, JWT validation, rate limiting, request aggregation.

```text
buyzen-api-gateway/
├── pom.xml
├── Dockerfile
└── src/main/
    ├── java/com/sm/buyzen/gateway/
    │   ├── ApiGatewayApplication.java
    │   ├── config/
    │   │   ├── GatewaySecurityConfig.java    # OAuth2 resource server at the edge
    │   │   ├── RouteConfig.java              # programmatic routes (or in yml)
    │   │   ├── CorsConfig.java
    │   │   ├── RateLimiterConfig.java        # Redis-backed request rate limiter
    │   │   ├── ResilienceConfig.java         # per-route circuit breakers
    │   │   └── SwaggerAggregatorConfig.java  # aggregate service OpenAPI docs
    │   ├── filter/
    │   │   ├── AuthenticationRelayFilter.java# forward JWT / user headers downstream
    │   │   ├── CorrelationIdGlobalFilter.java
    │   │   ├── RequestLoggingGlobalFilter.java
    │   │   └── FallbackHeaderFilter.java
    │   ├── ratelimit/
    │   │   └── UserKeyResolver.java          # rate-limit key = userId or IP
    │   ├── fallback/
    │   │   └── FallbackController.java        # circuit-breaker fallbacks
    │   └── exception/
    │       └── GatewayExceptionHandler.java
    └── resources/
        ├── application.yml                   # spring.cloud.gateway.routes[...] per service
        ├── application-local.yml
        └── application-prod.yml
```

---

## 8. Canonical microservice blueprint (the reusable template)

Every business service in Section 9 follows this exact internal shape. It is a pragmatic layered architecture (API → service → domain → infrastructure) with all cross-cutting packages present. Below is the **complete** blueprint using a placeholder root `com.sm.buyzen.<svc>`; Section 9 shows each service's concrete classes and only calls out what differs.

```text
buyzen-<name>-service/
├── pom.xml                                   # parent = root buyzen pom; deps = the commons it needs
├── Dockerfile                                # multi-stage; or Jib/buildpacks via pom
├── .dockerignore
├── README.md                                 # service purpose, API, events, run instructions
└── src/
    ├── main/
    │   ├── java/com/sm/buyzen/<svc>/
    │   │   ├── <Svc>ServiceApplication.java   # @SpringBootApplication entry point
    │   │   │
    │   │   ├── config/                        # ── Spring configuration & @ConfigurationProperties
    │   │   │   ├── SecurityConfig.java        # imports common-security, adds route rules
    │   │   │   ├── OpenApiConfig.java
    │   │   │   ├── PersistenceConfig.java     # JPA/tx, auditing (from common-data)
    │   │   │   ├── KafkaConfig.java           # bindings specific to this service
    │   │   │   ├── KafkaTopicConfig.java      # topic beans (dev auto-create)
    │   │   │   ├── CacheConfig.java           # Redis/Caffeine cache manager
    │   │   │   ├── WebClientConfig.java       # RestClient/Feign + Resilience4j
    │   │   │   ├── ResilienceConfig.java      # circuit breaker/retry/bulkhead defaults
    │   │   │   ├── AsyncConfig.java           # @EnableAsync executors
    │   │   │   ├── SchedulingConfig.java      # @EnableScheduling (outbox relay, jobs)
    │   │   │   ├── AwsConfig.java             # only if the service uses AWS SDK directly
    │   │   │   └── properties/
    │   │   │       ├── <Svc>Properties.java
    │   │   │       └── TopicProperties.java
    │   │   │
    │   │   ├── api/                           # ── Inbound adapter: REST
    │   │   │   ├── rest/
    │   │   │   │   ├── <Res>Controller.java           # public API (/api/v1/…)
    │   │   │   │   ├── <Res>AdminController.java       # admin API (/api/v1/admin/…)
    │   │   │   │   └── internal/<Res>InternalController.java  # service-to-service only
    │   │   │   ├── dto/
    │   │   │   │   ├── request/               # *Request records + Bean Validation
    │   │   │   │   ├── response/              # *Response records
    │   │   │   │   └── command/               # internal command objects (optional)
    │   │   │   ├── mapper/
    │   │   │   │   └── <Res>DtoMapper.java    # MapStruct: entity ⇄ dto
    │   │   │   └── advice/
    │   │   │       └── <Svc>ExceptionHandler.java   # service-specific @RestControllerAdvice
    │   │   │
    │   │   ├── service/                       # ── Application/business layer
    │   │   │   ├── <Res>Service.java          # interface
    │   │   │   ├── impl/<Res>ServiceImpl.java # @Transactional orchestration
    │   │   │   ├── command/                   # command handlers (write side)
    │   │   │   ├── query/                     # query services (read side)
    │   │   │   └── policy/                    # business rules / validators
    │   │   │
    │   │   ├── domain/                        # ── Core domain
    │   │   │   ├── model/                     # entities + embeddables + enums
    │   │   │   ├── repository/                # Spring Data repository interfaces
    │   │   │   ├── event/                     # in-process domain events (ApplicationEvent)
    │   │   │   ├── vo/                        # value objects
    │   │   │   └── specification/             # JPA Specifications for dynamic queries
    │   │   │
    │   │   ├── messaging/                     # ── Async adapter: Kafka
    │   │   │   ├── producer/
    │   │   │   │   └── <Svc>EventProducer.java
    │   │   │   ├── consumer/
    │   │   │   │   └── <X>EventConsumer.java  # @KafkaListener, idempotent
    │   │   │   ├── mapper/
    │   │   │   │   └── <Svc>EventMapper.java  # domain ⇄ Avro contract
    │   │   │   └── handler/
    │   │   │       └── <X>EventHandler.java   # what to do on each consumed event
    │   │   │
    │   │   ├── client/                        # ── Outbound adapter: other services (sync)
    │   │   │   ├── <Other>Client.java         # Feign / RestClient interface
    │   │   │   ├── dto/                        # client-facing DTOs
    │   │   │   └── fallback/<Other>ClientFallback.java
    │   │   │
    │   │   ├── integration/                   # ── Outbound adapter: external systems
    │   │   │   └── …                          # payment gateway, carrier, AWS, LLM, etc.
    │   │   │
    │   │   ├── exception/                     # ── Service-specific exceptions
    │   │   │   └── <Res>NotFoundException.java …
    │   │   │
    │   │   └── support/                       # ── Misc helpers, constants, mappers
    │   │       ├── constant/
    │   │       └── util/
    │   │
    │   └── resources/
    │       ├── application.yml                # base config (imports config-server)
    │       ├── application-local.yml
    │       ├── application-dev.yml
    │       ├── application-prod.yml
    │       ├── bootstrap.yml                  # spring.config.import → config-server
    │       ├── logback-spring.xml             # (or inherit from common-observability)
    │       └── db/migration/                  # Flyway
    │           ├── V1__init.sql
    │           ├── V2__add_indexes.sql
    │           └── R__seed_reference_data.sql
    │
    └── test/
        ├── java/com/sm/buyzen/<svc>/
        │   ├── api/                           # @WebMvcTest controller slice tests
        │   ├── service/                       # unit tests (Mockito)
        │   ├── domain/                        # entity/VO tests
        │   ├── messaging/                     # consumer/producer tests
        │   ├── integration/                   # @SpringBootTest + Testcontainers
        │   ├── contract/                      # Spring Cloud Contract / Pact
        │   └── architecture/                  # ArchUnit rules (layer boundaries)
        └── resources/
            ├── application-test.yml
            └── wiremock/                      # stubbed downstream responses
```

**Why each package earns its place**

- `config/` isolates all wiring so business code stays framework-light; `properties/` gives typed, validated config.
- `api/` is the only place HTTP types live — controllers stay thin, DTOs never leak into `domain/`.
- `service/` holds transactions and orchestration; splitting `command/` vs `query/` keeps a light CQRS seam.
- `domain/` is persistence-aware but service-agnostic; `specification/` avoids query sprawl in repositories.
- `messaging/` and `client/` and `integration/` are all *adapters* — swap Kafka/Feign/vendor without touching `service/`.
- `exception/` + the common `GlobalExceptionHandler` guarantee consistent RFC-7807 error bodies.
- `test/architecture/` (ArchUnit) mechanically enforces these boundaries so the structure can't rot.

---

## 9. Business services (full set)

Each service below follows the Section 8 blueprint. Only concrete, domain-specific classes are listed; packages marked *(template)* contain the standard cross-cutting files shown in Section 8.

### 9.1 `buyzen-identity-service` — profiles & Keycloak integration

Keycloak is the IdP (it owns credentials/tokens). This service owns the **extended user profile**, addresses, preferences, and provisioning against the Keycloak Admin API.

```text
buyzen-identity-service/  → com.sm.buyzen.identity
├── IdentityServiceApplication.java
├── config/
│   ├── SecurityConfig.java
│   ├── KeycloakAdminConfig.java              # admin client bean
│   ├── properties/KeycloakAdminProperties.java
│   └── (template: OpenApiConfig, KafkaConfig, PersistenceConfig…)
├── api/rest/
│   ├── AccountController.java                # POST /api/v1/accounts (register → Keycloak)
│   ├── UserProfileController.java            # /api/v1/users/me, profiles
│   ├── AddressController.java                # /api/v1/users/{id}/addresses
│   └── admin/UserAdminController.java        # /api/v1/admin/users (roles, enable/disable)
│   └── dto/{request,response}/               # RegisterUserRequest, UpdateProfileRequest, AddressRequest, UserProfileResponse…
├── service/
│   ├── AccountService.java / impl/
│   ├── UserProfileService.java / impl/
│   ├── AddressService.java / impl/
│   └── KeycloakUserProvisioningService.java  # create/update/disable users in Keycloak
├── domain/
│   ├── model/ UserProfile, Address, UserPreference, Consent (enums: AddressType, Gender)
│   └── repository/ UserProfileRepository, AddressRepository
├── messaging/
│   ├── producer/UserEventProducer.java       # → buyzen.user.registered.v1, buyzen.user.updated.v1
│   └── consumer/ (none critical)
├── integration/keycloak/
│   ├── KeycloakAdminClient.java
│   └── KeycloakRoleMapper.java
├── exception/ UserNotFoundException, DuplicateEmailException, KeycloakIntegrationException
└── resources/db/migration/ V1__user_profile.sql …
```

DB: `buyzen_identity`. Publishes `UserRegistered`/`UserUpdated`. Consumers: notification (welcome email).

### 9.2 `buyzen-product-service` — catalog

```text
buyzen-product-service/  → com.sm.buyzen.product
├── ProductServiceApplication.java
├── config/ (template) + CacheConfig (hot catalog reads)
├── api/rest/
│   ├── ProductController.java                # /api/v1/products (list, get, filter, facet)
│   ├── CategoryController.java               # /api/v1/categories (tree)
│   ├── BrandController.java
│   └── admin/ ProductAdminController, CategoryAdminController, InventoryLinkController
│   └── dto/{request,response}/               # CreateProductRequest, UpdateProductRequest, ProductResponse, ProductSummaryResponse, CategoryResponse…
├── service/
│   ├── ProductService / impl
│   ├── CategoryService / impl
│   ├── BrandService / impl
│   ├── PricingService / impl                 # base price, list price, tax class
│   └── query/ProductQueryService.java        # read-optimized fetches
├── domain/
│   ├── model/ Product, ProductVariant, ProductAttribute, AttributeValue, Category, Brand,
│   │          ProductImageRef, PriceInfo, Dimension (enums: ProductStatus, AttributeType)
│   ├── repository/ ProductRepository, VariantRepository, CategoryRepository, BrandRepository
│   └── specification/ ProductSpecifications.java
├── messaging/
│   ├── producer/ProductEventProducer.java    # → product.created/updated/deleted, price.changed
│   └── consumer/ MediaEventConsumer (attach uploaded images), ReviewEventConsumer (rating rollup)
├── client/ InventoryClient (availability badge), fallback/
├── exception/ ProductNotFoundException, CategoryNotFoundException, DuplicateSkuException
└── resources/db/migration/ V1__catalog.sql, V2__attributes.sql, R__seed_categories.sql
```

DB: `buyzen_product`. Publishes catalog events consumed by search & recommendation.

### 9.3 `buyzen-inventory-service` — stock & reservations

Participates in the order saga: reserves stock on `OrderCreated`, releases on cancel.

```text
buyzen-inventory-service/  → com.sm.buyzen.inventory
├── InventoryServiceApplication.java
├── config/ (template) + SchedulingConfig (reservation expiry sweeper)
├── api/rest/
│   ├── InventoryController.java              # /api/v1/inventory/{sku} availability
│   ├── WarehouseController.java
│   └── admin/StockAdminController.java       # adjust stock, receive shipments
│   └── dto/{request,response}/               # StockAdjustmentRequest, AvailabilityResponse…
├── service/
│   ├── InventoryService / impl
│   ├── StockReservationService / impl        # reserve / confirm / release (idempotent)
│   └── WarehouseService / impl
├── domain/
│   ├── model/ InventoryItem, StockReservation, ReservationLine, Warehouse, StockMovement
│   │          (enums: ReservationStatus, MovementType)
│   └── repository/ InventoryItemRepository, StockReservationRepository, StockMovementRepository
├── messaging/
│   ├── consumer/OrderEventConsumer.java      # on OrderCreated → try reserve
│   │            + on OrderCancelled → release
│   ├── producer/InventoryEventProducer.java  # → stock.reserved / stock.reservation-failed / stock.released
│   └── handler/StockReservationHandler.java
├── exception/ InsufficientStockException, ReservationNotFoundException
└── resources/db/migration/ V1__inventory.sql, V2__reservation.sql
```

DB: `buyzen_inventory`. Uses optimistic locking (`@Version`) on `InventoryItem` to prevent oversell.

### 9.4 `buyzen-cart-service` — shopping cart (Redis)

```text
buyzen-cart-service/  → com.sm.buyzen.cart
├── CartServiceApplication.java
├── config/
│   ├── RedisConfig.java                      # RedisTemplate, TTL, serializers
│   ├── CartProperties.java                   # cart TTL, max items
│   └── (template: SecurityConfig, WebClientConfig…)
├── api/rest/
│   ├── CartController.java                   # /api/v1/cart (get, add, update qty, remove, clear)
│   └── CheckoutController.java               # /api/v1/cart/checkout → hands off to order-service
│   └── dto/{request,response}/               # AddItemRequest, UpdateItemRequest, CartResponse, CartItemResponse
├── service/
│   ├── CartService / impl                    # Redis-backed operations
│   ├── CartPricingService / impl             # subtotal, discounts (calls product)
│   └── CheckoutService / impl
├── domain/
│   ├── model/ Cart, CartItem (Redis hashes; @RedisHash), enums CartStatus
│   └── repository/ CartRepository (Spring Data Redis)
├── client/
│   ├── ProductClient.java                    # price/availability at add-to-cart
│   ├── InventoryClient.java
│   └── fallback/
├── messaging/
│   └── producer/CartEventProducer.java       # cart.checked-out (optional analytics)
├── exception/ CartNotFoundException, ItemNotInCartException, PriceMismatchException
└── (no Flyway — Redis only; optional Postgres for persisted/abandoned carts)
```

Store: Redis (ElastiCache). Optional abandoned-cart persistence to Postgres consumed by notification.

### 9.5 `buyzen-order-service` — orders + **saga orchestrator**

The heart of the platform. Owns the order lifecycle and orchestrates payment → inventory → shipping via Kafka, using the **transactional outbox** for reliable publishing and a **saga** for distributed consistency with compensations.

```text
buyzen-order-service/  → com.sm.buyzen.order
├── OrderServiceApplication.java
├── config/
│   ├── SecurityConfig, KafkaConfig, KafkaTopicConfig, PersistenceConfig
│   ├── SchedulingConfig.java                 # outbox relay + saga timeout sweeper
│   └── properties/ OrderProperties, SagaProperties
├── api/rest/
│   ├── OrderController.java                  # /api/v1/orders (place, get, list mine, cancel)
│   ├── admin/OrderAdminController.java       # /api/v1/admin/orders (search, refund, force-cancel)
│   └── internal/OrderInternalController.java # status lookups for other services
│   └── dto/{request,response}/               # PlaceOrderRequest, OrderResponse, OrderLineResponse, OrderStatusResponse
├── service/
│   ├── OrderService / impl                   # create order (PENDING) + write outbox in same tx
│   ├── OrderQueryService / impl
│   └── OrderCancellationService / impl
├── saga/                                     # ── orchestration lives here
│   ├── OrderSagaOrchestrator.java            # drives state transitions
│   ├── OrderSagaState.java                   # persisted saga instance
│   ├── OrderSagaStep.java                    # enum: RESERVE_STOCK, TAKE_PAYMENT, CREATE_SHIPMENT…
│   ├── SagaStateRepository.java
│   └── compensation/
│       ├── ReleaseStockCompensation.java
│       └── RefundPaymentCompensation.java
├── domain/
│   ├── model/ Order, OrderLine, OrderStatusHistory, ShippingAddress, BillingAddress,
│   │          OrderTotals (enums: OrderStatus, PaymentStatus, FulfillmentStatus)
│   └── repository/ OrderRepository, OrderLineRepository, OrderStatusHistoryRepository
├── messaging/
│   ├── producer/OrderEventProducer.java      # via outbox → order.created/confirmed/cancelled/completed
│   ├── consumer/
│   │   ├── PaymentEventConsumer.java         # payment.completed / payment.failed
│   │   ├── InventoryEventConsumer.java       # stock.reserved / stock.reservation-failed
│   │   └── ShippingEventConsumer.java        # shipment.created / shipment.delivered
│   ├── handler/ (one per consumed event → advances the saga)
│   └── mapper/OrderEventMapper.java
├── client/
│   ├── ProductClient.java                    # snapshot price/name at order time
│   ├── CartClient.java                       # pull cart on checkout
│   └── fallback/
├── outbox/                                   # (uses common-kafka outbox; local relay config)
├── exception/ OrderNotFoundException, InvalidOrderStateException, OrderCreationException
└── resources/db/migration/ V1__orders.sql, V2__outbox.sql, V3__saga_state.sql, V4__processed_events.sql
```

DB: `buyzen_order`. Patterns: transactional outbox, orchestration saga with compensations, idempotent consumers (processed-events table).

### 9.6 `buyzen-payment-service` — payments & refunds

```text
buyzen-payment-service/  → com.sm.buyzen.payment
├── PaymentServiceApplication.java
├── config/ (template) + payment gateway (Stripe) config, idempotency
├── api/rest/
│   ├── PaymentController.java                # /api/v1/payments (status, methods)
│   ├── RefundController.java                 # /api/v1/refunds
│   ├── webhook/PaymentWebhookController.java # /webhooks/stripe (signature-verified)
│   └── admin/PaymentAdminController.java
│   └── dto/{request,response}/               # payment method, refund request, payment response…
├── service/
│   ├── PaymentService / impl                 # authorize/capture, idempotent by orderId
│   ├── RefundService / impl
│   └── PaymentMethodService / impl
├── domain/
│   ├── model/ Payment, PaymentTransaction, Refund, PaymentMethod, PaymentAttempt
│   │          (enums: PaymentStatus, TransactionType, Provider)
│   └── repository/ PaymentRepository, TransactionRepository, RefundRepository
├── messaging/
│   ├── consumer/OrderEventConsumer.java      # on order.created → charge
│   ├── producer/PaymentEventProducer.java    # → payment.completed / payment.failed / refund.processed
│   └── handler/PaymentRequestedHandler.java
├── integration/gateway/                      # ── external PSP adapter
│   ├── PaymentGateway.java                   # port
│   ├── StripePaymentGateway.java             # adapter
│   ├── dto/ …
│   └── WebhookSignatureVerifier.java
├── exception/ PaymentDeclinedException, PaymentGatewayException, RefundNotAllowedException
└── resources/db/migration/ V1__payment.sql, V2__refund.sql, V3__idempotency.sql
```

DB: `buyzen_payment`. Secrets (PSP keys) via AWS Secrets Manager. Webhooks verified before processing.

### 9.7 `buyzen-shipping-service` — fulfillment & tracking

```text
buyzen-shipping-service/  → com.sm.buyzen.shipping
├── ShippingServiceApplication.java
├── config/ (template) + carrier integration config
├── api/rest/
│   ├── ShipmentController.java               # /api/v1/shipments (get, track)
│   ├── TrackingController.java               # /api/v1/shipments/{id}/tracking
│   └── admin/ShipmentAdminController.java
│   └── dto/{request,response}/               # ShipmentResponse, TrackingResponse…
├── service/
│   ├── ShipmentService / impl
│   ├── TrackingService / impl
│   └── RateService / impl                    # shipping-rate quotes
├── domain/
│   ├── model/ Shipment, ShipmentItem, Carrier, TrackingEvent, ShippingLabel
│   │          (enums: ShipmentStatus, CarrierType)
│   └── repository/ ShipmentRepository, TrackingEventRepository, CarrierRepository
├── messaging/
│   ├── consumer/OrderEventConsumer.java      # on order.confirmed → create shipment
│   ├── producer/ShippingEventProducer.java   # → shipment.created / dispatched / delivered
│   └── handler/CreateShipmentHandler.java
├── integration/carrier/                      # ── external carrier adapters
│   ├── CarrierClient.java                    # port
│   ├── FedExCarrierClient.java / UpsCarrierClient.java
│   └── webhook/CarrierWebhookController.java  # tracking status callbacks
├── exception/ ShipmentNotFoundException, CarrierIntegrationException
└── resources/db/migration/ V1__shipping.sql, V2__tracking.sql, R__seed_carriers.sql
```

DB: `buyzen_shipping`. Feeds `shipment.delivered` back to the order saga to complete orders.

### 9.8 `buyzen-recommendation-service` — **AI** personalization

Generates personalized recommendations, "similar products," and trending items using vector embeddings (AWS Bedrock) stored in **pgvector**, fed by clickstream + order events.

```text
buyzen-recommendation-service/  → com.sm.buyzen.recommendation
├── RecommendationServiceApplication.java
├── config/
│   ├── PgVectorConfig.java                   # vector datasource / extension
│   ├── BedrockConfig.java                    # embedding + LLM clients (from common-aws)
│   ├── RecommendationProperties.java         # top-K, model ids, weights
│   └── (template: SecurityConfig, KafkaConfig, SchedulingConfig for batch rebuilds)
├── api/rest/
│   ├── RecommendationController.java         # /api/v1/recommendations/{userId}
│   ├── SimilarProductController.java         # /api/v1/products/{id}/similar
│   ├── TrendingController.java               # /api/v1/recommendations/trending
│   └── AssistantController.java              # /api/v1/assistant/chat  (LLM shopping assistant, RAG over catalog)
│   └── dto/{request,response}/               # RecommendationResponse, AssistantChatRequest/Response…
├── service/
│   ├── RecommendationService / impl          # blends collaborative + content-based signals
│   ├── SimilarityService / impl              # vector KNN search
│   ├── TrendingService / impl                # window aggregation of interactions
│   └── AssistantService / impl               # RAG: retrieve → prompt → LLM answer
├── ai/                                       # ── AI/ML layer
│   ├── embedding/
│   │   ├── EmbeddingService.java             # text/product → vector (Bedrock Titan)
│   │   └── ProductEmbeddingIndexer.java
│   ├── vector/
│   │   ├── VectorStore.java                  # port
│   │   └── PgVectorStore.java                # adapter (KNN over embeddings)
│   ├── rag/
│   │   ├── Retriever.java
│   │   ├── PromptBuilder.java
│   │   └── ChatModelService.java             # Bedrock LLM invoke
│   └── feature/
│       └── UserFeatureBuilder.java           # builds user vector from interactions
├── domain/
│   ├── model/ UserInteraction, ProductEmbedding (vector column), Recommendation,
│   │          TrendingScore (enums: InteractionType)
│   └── repository/ InteractionRepository, ProductEmbeddingRepository, RecommendationRepository
├── messaging/
│   └── consumer/
│       ├── ProductEventConsumer.java         # product.created/updated → (re)embed & index
│       ├── OrderEventConsumer.java           # order.completed → purchase signal
│       └── InteractionEventConsumer.java     # clickstream: viewed / added-to-cart
├── client/ ProductClient (enrich responses), fallback/
├── exception/ EmbeddingException, VectorSearchException, ModelInvocationException
└── resources/db/migration/ V1__interactions.sql, V2__product_embeddings.sql (CREATE EXTENSION vector)
```

DB: `buyzen_recommendation` (PostgreSQL + `pgvector`). AI: Bedrock embeddings + LLM; RAG for the assistant.

### 9.9 `buyzen-review-service` — reviews + **AI** sentiment/moderation

```text
buyzen-review-service/  → com.sm.buyzen.review
├── ReviewServiceApplication.java
├── config/ (template) + Comprehend/Bedrock config
├── api/rest/
│   ├── ReviewController.java                 # /api/v1/products/{id}/reviews (create, list)
│   ├── RatingController.java                 # aggregate rating for a product
│   └── admin/ReviewModerationController.java
│   └── dto/{request,response}/               # CreateReviewRequest, ReviewResponse, RatingSummaryResponse
├── service/
│   ├── ReviewService / impl
│   ├── RatingAggregationService / impl
│   └── ModerationService / impl
├── ai/
│   ├── SentimentAnalyzer.java                # AWS Comprehend / Bedrock → POSITIVE/NEGATIVE/…
│   ├── ContentModerator.java                 # toxicity / policy check before publish
│   └── ReviewSummarizer.java                 # LLM "customers say…" summary per product
├── domain/
│   ├── model/ Review, Rating, ReviewSentiment, ReviewSummary, ModerationResult
│   │          (enums: ReviewStatus, Sentiment)
│   └── repository/ ReviewRepository, RatingRepository, ReviewSummaryRepository
├── messaging/
│   ├── producer/ReviewEventProducer.java     # → review.created (product & search consume)
│   └── consumer/OrderEventConsumer.java      # gate reviews to verified purchasers
├── client/ ProductClient, OrderClient (verify purchase), fallback/
├── exception/ ReviewNotAllowedException, ModerationRejectedException
└── resources/db/migration/ V1__reviews.sql, V2__sentiment.sql
```

DB: `buyzen_review`. AI: sentiment, moderation, and LLM review summaries.

### 9.10 `buyzen-search-service` — search (keyword + **AI** semantic)

CQRS read-model: consumes catalog/review events and indexes into OpenSearch; offers hybrid (BM25 + vector) search.

```text
buyzen-search-service/  → com.sm.buyzen.search
├── SearchServiceApplication.java
├── config/
│   ├── OpenSearchConfig.java                 # client, index templates
│   ├── EmbeddingConfig.java                  # Bedrock embeddings for semantic search
│   ├── SearchProperties.java                 # index names, boosts, k-NN params
│   └── (template: SecurityConfig, KafkaConfig)
├── api/rest/
│   ├── SearchController.java                 # /api/v1/search?q=… (hybrid, facets, paging)
│   ├── AutocompleteController.java           # /api/v1/search/suggest
│   └── admin/ReindexController.java          # /api/v1/admin/search/reindex
│   └── dto/{request,response}/               # SearchRequest, SearchResponse, FacetResponse, SuggestionResponse
├── service/
│   ├── SearchService / impl                  # builds hybrid query
│   ├── IndexingService / impl                # upsert/delete docs
│   ├── SuggestionService / impl
│   └── SemanticSearchService / impl          # vector k-NN branch
├── ai/
│   └── QueryEmbeddingService.java            # embed the query for semantic ranking
├── domain/
│   ├── document/ ProductDocument, SuggestionDocument   # OpenSearch mappings
│   └── repository/ ProductSearchRepository (Spring Data OpenSearch)
├── messaging/
│   └── consumer/
│       ├── ProductEventConsumer.java         # created/updated/priceChanged/deleted → index
│       └── ReviewEventConsumer.java          # review.created → update rating field
├── exception/ SearchException, IndexingException
└── resources/opensearch/ product-index-template.json, suggest-settings.json
```

Store: OpenSearch (no relational DB). AI: query embeddings for semantic/hybrid ranking.

### 9.11 `buyzen-notification-service` — email / SMS / push

Pure event consumer: turns domain events into user notifications via AWS SES/SNS.

```text
buyzen-notification-service/  → com.sm.buyzen.notification
├── NotificationServiceApplication.java
├── config/ (template) + SES/SNS config, template engine (Thymeleaf)
├── api/rest/
│   ├── NotificationController.java           # /api/v1/notifications (history for a user)
│   ├── PreferenceController.java             # /api/v1/notifications/preferences (opt-in/out)
│   └── admin/TemplateAdminController.java     # manage templates
│   └── dto/{request,response}/
├── service/
│   ├── NotificationService / impl            # orchestrates channel selection + preferences
│   ├── TemplateService / impl                # render templates with event data
│   ├── PreferenceService / impl
│   └── channel/
│       ├── NotificationChannel.java          # port
│       ├── EmailChannel.java                 # SES
│       ├── SmsChannel.java                   # SNS
│       └── PushChannel.java
├── domain/
│   ├── model/ Notification, NotificationTemplate, NotificationPreference, DeliveryAttempt
│   │          (enums: Channel, NotificationType, DeliveryStatus)
│   └── repository/ NotificationRepository, TemplateRepository, PreferenceRepository
├── messaging/
│   └── consumer/
│       ├── OrderEventConsumer.java           # order.created/confirmed/shipped/delivered
│       ├── PaymentEventConsumer.java         # payment.completed/failed, refund.processed
│       ├── ShippingEventConsumer.java
│       └── UserEventConsumer.java            # user.registered → welcome email
├── exception/ NotificationDeliveryException, TemplateRenderException
└── resources/
    ├── templates/email/ order-confirmation.html, shipment-dispatched.html, welcome.html …
    └── db/migration/ V1__notification.sql, V2__templates.sql, R__seed_templates.sql
```

DB: `buyzen_notification`. Channels: SES (email), SNS (SMS/push). Respects per-user preferences.

### 9.12 `buyzen-media-service` — images & assets (S3/CloudFront)

```text
buyzen-media-service/  → com.sm.buyzen.media
├── MediaServiceApplication.java
├── config/ (template) + S3/CloudFront config, multipart limits
├── api/rest/
│   ├── MediaController.java                  # /api/v1/media (upload, get metadata)
│   ├── PresignController.java                # /api/v1/media/presign (direct-to-S3 upload URLs)
│   └── admin/MediaAdminController.java
│   └── dto/{request,response}/               # PresignRequest, MediaResponse, UploadResponse
├── service/
│   ├── MediaService / impl                   # store, retrieve, delete
│   ├── PresignService / impl                 # presigned PUT/GET URLs
│   └── ImageProcessingService / impl         # thumbnails, variants (async)
├── domain/
│   ├── model/ MediaAsset, MediaVariant, MediaMetadata (enums: MediaType, ProcessingStatus)
│   └── repository/ MediaAssetRepository, MediaVariantRepository
├── integration/aws/
│   ├── S3MediaStore.java                     # uses common-aws S3StorageService
│   └── CloudFrontUrlSigner.java
├── messaging/
│   ├── producer/MediaEventProducer.java      # → media.uploaded (product consumes)
│   └── consumer/ (optional: cleanup on product.deleted)
├── exception/ MediaNotFoundException, UnsupportedMediaTypeException, StorageException
└── resources/db/migration/ V1__media.sql
```

DB: `buyzen_media` (metadata only; bytes live in S3, served via CloudFront).

---

## 10. Infrastructure as code — `infra/`

Not Maven modules; deployed by the CI/CD pipeline.

```text
infra/
├── terraform/                                # AWS provisioning
│   ├── environments/
│   │   ├── dev/    (main.tf, backend.tf, terraform.tfvars)
│   │   ├── staging/
│   │   └── prod/
│   ├── modules/
│   │   ├── network/          # VPC, subnets, NAT, security groups
│   │   ├── eks/              # or ecs/ — cluster, node groups, IRSA
│   │   ├── rds/              # PostgreSQL per service (or shared instance, separate DBs)
│   │   ├── msk/              # managed Kafka + schema registry
│   │   ├── elasticache/      # Redis
│   │   ├── opensearch/
│   │   ├── s3-cloudfront/    # media bucket + CDN
│   │   ├── ses-sns/
│   │   ├── secrets/          # Secrets Manager entries
│   │   ├── ecr/              # container registries
│   │   ├── iam/              # roles/policies, IRSA bindings
│   │   └── observability/    # managed Prometheus/Grafana or self-hosted
│   └── global/               # Route53, ACM certificates, WAF
│
├── k8s/                                      # Kustomize base + overlays (if EKS)
│   ├── base/
│   │   ├── <service>/ deployment.yaml, service.yaml, hpa.yaml, configmap.yaml, servicemonitor.yaml
│   │   └── namespace.yaml
│   └── overlays/ dev/ staging/ prod/
│
├── helm/                                     # alternative to raw manifests
│   ├── buyzen-service/                       # one reusable chart for all services
│   │   ├── Chart.yaml, values.yaml
│   │   └── templates/ deployment.yaml, service.yaml, ingress.yaml, hpa.yaml, secrets.yaml
│   └── umbrella/ Chart.yaml (deps on subcharts + kafka/keycloak/redis)
│
├── keycloak/
│   ├── realm-export/buyzen-realm.json        # realm, clients, roles, scopes
│   ├── themes/buyzen/                        # login theme
│   └── README.md                             # bootstrap instructions
│
├── monitoring/
│   ├── prometheus/ prometheus.yml, alert-rules.yaml
│   ├── grafana/dashboards/ *.json            # per-service + platform dashboards
│   ├── tempo/  loki/  config
│   └── alertmanager/ alertmanager.yml
│
└── scripts/
    ├── bootstrap-local.sh                    # spin up docker-compose + seed data
    ├── create-topics.sh
    ├── import-keycloak-realm.sh
    └── smoke-test.sh
```

---

## 11. Kafka topic catalog

All topics use the `buyzen.<domain>.<event>.v1` convention with a matching `.dlt` dead-letter topic. Avro schemas live in `buyzen-contracts`.

| Topic | Producer | Consumers | Purpose |
|---|---|---|---|
| `buyzen.user.registered.v1` | identity | notification | Welcome email, downstream profile cache |
| `buyzen.user.updated.v1` | identity | (various) | Profile change propagation |
| `buyzen.product.created.v1` | product | search, recommendation | Index & embed new products |
| `buyzen.product.updated.v1` | product | search, recommendation | Re-index / re-embed |
| `buyzen.product.price-changed.v1` | product | search, cart | Keep price fresh |
| `buyzen.product.deleted.v1` | product | search, recommendation, media | Cleanup |
| `buyzen.order.created.v1` | order | payment, inventory, notification | Kick off saga |
| `buyzen.order.confirmed.v1` | order | shipping, notification | Trigger fulfillment |
| `buyzen.order.cancelled.v1` | order | inventory, payment, notification | Compensations |
| `buyzen.order.completed.v1` | order | recommendation, notification | Purchase signal |
| `buyzen.payment.completed.v1` | payment | order, notification | Advance saga |
| `buyzen.payment.failed.v1` | payment | order, notification | Compensate saga |
| `buyzen.payment.refund-processed.v1` | payment | order, notification | Refund closure |
| `buyzen.inventory.stock-reserved.v1` | inventory | order | Advance saga |
| `buyzen.inventory.stock-reservation-failed.v1` | inventory | order | Compensate saga |
| `buyzen.inventory.stock-released.v1` | inventory | order | Compensation ack |
| `buyzen.shipping.shipment-created.v1` | shipping | order, notification | Fulfillment started |
| `buyzen.shipping.shipment-dispatched.v1` | shipping | notification | Tracking update |
| `buyzen.shipping.shipment-delivered.v1` | shipping | order, notification | Complete order |
| `buyzen.review.created.v1` | review | product, search | Rating rollup, index |
| `buyzen.media.uploaded.v1` | media | product | Attach image to product |
| `buyzen.interaction.recorded.v1` | gateway/product | recommendation | Clickstream signal |

---

## 12. Data stores per service (database-per-service)

| Service | Store | Name |
|---|---|---|
| identity | PostgreSQL | `buyzen_identity` |
| product | PostgreSQL | `buyzen_product` |
| inventory | PostgreSQL | `buyzen_inventory` |
| cart | Redis (+ optional PostgreSQL) | `buyzen_cart` |
| order | PostgreSQL | `buyzen_order` |
| payment | PostgreSQL | `buyzen_payment` |
| shipping | PostgreSQL | `buyzen_shipping` |
| recommendation | PostgreSQL + pgvector | `buyzen_recommendation` |
| review | PostgreSQL | `buyzen_review` |
| search | OpenSearch | index `buyzen-products` |
| notification | PostgreSQL | `buyzen_notification` |
| media | PostgreSQL (metadata) + S3 | `buyzen_media` |

Identity/tokens themselves are owned by **Keycloak** (its own database).

## 13. Local port map

| Component | Port |
|---|---|
| API Gateway | 8080 |
| Config Server | 8888 |
| Discovery (Eureka) | 8761 |
| identity / product / inventory / cart | 8081 / 8082 / 8083 / 8084 |
| order / payment / shipping | 8085 / 8086 / 8087 |
| recommendation / review / search | 8088 / 8089 / 8090 |
| notification / media | 8091 / 8092 |
| Keycloak | 8443 (or 9080 http) |
| Kafka / Schema Registry | 9092 / 8081-registry |
| PostgreSQL / Redis / OpenSearch | 5432 / 6379 / 9200 |
| Prometheus / Grafana | 9090 / 3000 |

---

## 14. Cross-cutting patterns (baked into the structure)

- **API Gateway as single ingress** — all external traffic authenticated at the edge; JWTs and user headers relayed downstream.
- **OAuth2 / OIDC everywhere** — Keycloak issues tokens; every service is a resource server (via `buyzen-common-security`); method-level `@PreAuthorize`.
- **Database per service** — no shared schemas; integration only via API or events.
- **Saga (orchestration)** — `order-service` coordinates payment/inventory/shipping with explicit compensations.
- **Transactional outbox** — business write + event enqueue in one DB transaction; a relay publishes to Kafka (Debezium-friendly). Lives in `buyzen-common-kafka` + each service's `outbox` table.
- **Idempotent consumers** — `processed_events` table + `IdempotentConsumerSupport` prevent double-processing on redelivery.
- **Dead-letter topics + retry/backoff** — non-recoverable messages parked on `<topic>.dlt`.
- **CQRS-lite** — `search-service` is a read model built from events; write services stay authoritative.
- **Resilience4j** — circuit breakers, retries, bulkheads, and rate limiters on every outbound `client/` call, with fallbacks.
- **Observability by default** — structured JSON logs (Loki), Micrometer metrics (Prometheus), OpenTelemetry traces (Tempo), correlation IDs propagated over HTTP *and* Kafka headers.
- **Config & secrets** — externalized via Config Server; secrets via AWS Secrets Manager (never in Git).
- **Schema governance** — Avro in `buyzen-contracts` + Schema Registry compatibility checks in CI.
- **ArchUnit** — `test/architecture/` enforces layering (e.g. `domain` must not depend on `api`).

## 15. Build & run

```bash
# one-time: start backing services + Keycloak realm + Kafka topics
make up                      # docker compose up (infra) + import realm + create topics

# build everything (parent aggregator builds all modules in dependency order)
./mvnw clean install

# run a single service against local infra
./mvnw -pl buyzen-services/buyzen-order-service spring-boot:run -Dspring-boot.run.profiles=local

# run the full stack as containers
docker compose up --build

# tests: unit + slice always; integration via Testcontainers
./mvnw verify                # runs failsafe + Testcontainers integration tests
```

Startup order at runtime: **config-server → discovery-server → platform/business services → api-gateway**. Backing infra (Kafka, Keycloak, Postgres, Redis, OpenSearch) must be healthy first.

## 16. Suggested build order & extension points

Build the backbone first (config, discovery, gateway, common libs, identity, product), then the commerce core (cart, order, inventory, payment), then fulfillment (shipping, notification, media), then the AI/search tier (search, recommendation, review). Natural next services when you outgrow this set: `promotion-service` (coupons/pricing rules), `analytics-service` (BI events), `wishlist-service`, and a `bff-web` / `bff-mobile` for channel-specific aggregation at the edge.

---

*End of blueprint. Every module here declares the root `buyzen` POM as its parent, lives under the aggregator it belongs to, and follows the Section 8 internal layout — so the structure stays uniform as the platform grows.*
