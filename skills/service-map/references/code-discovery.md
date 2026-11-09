# Where to look in the code

A checklist for JVM / Spring services, the common case. For another stack, ask the same
questions of its equivalents (route definitions, HTTP clients, message consumers, build file).
Search with Grep first and open only the files that match; the doc needs facts, not a tour.

Resolve `${...}` placeholders from `application*.yml` / `.properties` (and profile files) so
names and paths in the doc are real values. If a value only comes from the environment and
the repo shows no default, say so and open a GAP if it hides a service or topic name.

## Identification

- `pom.xml`: `groupId`, `artifactId`, `<parent>`, BOMs under `<dependencyManagement>`
  (`<scope>import</scope>`). Gradle: `group`, `rootProject.name` in `settings.gradle(.kts)`,
  `platform(...)` / `enforcedPlatform(...)`, version catalogs.
- Base package: the package of the `@SpringBootApplication` class.
- Service name: `spring.application.name`. Aliases: k8s `Service` / helm release name,
  compose service name, Eureka/Consul id if different.
- Several `@SpringBootApplication` classes or several deployable modules: ask which one.

## Set evidence

Parent POM / BOM coordinates, internal libraries shared with other repos (same groupId
prefix), compose / helm / k8s files that list other services, config properties that name
other services, CI files naming a shared registry or namespace.

## Inbound

- REST: `@RestController`, `@Controller`, `@RequestMapping`, `@GetMapping` and siblings.
  Full path = `server.servlet.context-path` + `spring.mvc.servlet.path` + class mapping +
  method mapping. WebFlux `RouterFunction` beans. JAX-RS `@Path`. Generated interfaces from an
  OpenAPI spec (`src/main/resources/**/*.yaml`, openapi-generator config): paths are in the
  spec.
- Messaging: `@KafkaListener` (topics, topicPattern, groupId), `@RabbitListener`,
  `@JmsListener`, `@SqsListener`, Spring Cloud Stream bindings
  (`spring.cloud.stream.bindings.<fn>-in-0.destination` and `group`).
- gRPC: `.proto` services implemented here (classes extending `*ImplBase`, `@GrpcService`).
- Scheduled work: `@Scheduled`, Quartz jobs, ShedLock.
- Security on the way in: `SecurityFilterChain`, resource-server / JWT config, method
  security. Tells you who is allowed to call, which often hints at who does.

## Outbound

- `@FeignClient(name, url, path)`: `name` is usually the target service name. Combine `path`
  with method mappings for the full path.
- `WebClient`, `RestTemplate`, `RestClient`, `@HttpExchange` interfaces: find the base URL
  (bean config or property) and map its host to a service name. Host `billing-service`,
  `http://billing-service:8080`, `lb://billing-service` all mean billing-service. A property
  like `pricing.base-url` with no resolvable value is a GAP ("which service is behind
  property pricing.base-url?").
- gRPC stubs (`*BlockingStub`, `*Stub`, `ManagedChannelBuilder.forTarget`, `grpc.client.*`
  config).
- Producers: `KafkaTemplate.send`, `StreamBridge.send`, `-out-0` bindings, `RabbitTemplate`,
  `JmsTemplate`, outbox tables relayed by a publisher. Note event types where cheap.
- Data: `spring.datasource.*`, JPA entities and repositories, Mongo, Redis, Elasticsearch;
  migrations (Flyway `db/migration`, Liquibase changelogs).
- External systems: SDK clients (S3, Stripe, SMTP), URLs pointing outside the set.

## Cross-cutting

Tenant handling (filters, `TenantContext`, Hibernate filters, `X-Tenant-Id` headers),
`@Transactional` boundaries that span a write plus a publish, outbox, idempotency keys,
`@Retryable` / Resilience4j, `@Cacheable` and eviction, locks, ordering assumptions on
topics. Record only what a change could break, with the class that enforces it.

## Code conventions

Package layering, DTO / entity mapping (MapStruct, hand-written), error handling
(`@ControllerAdvice`, `ProblemDetail`, error codes), naming patterns, validation, logging or
tracing setup that every class follows.

## Tests

`src/test` and any `*-IT` module: JUnit 5 / Spock / Cucumber, Mockito, AssertJ,
Testcontainers, WireMock, embedded Kafka, shared base classes and fixtures, test profiles.
