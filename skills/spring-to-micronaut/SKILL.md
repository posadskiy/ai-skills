---
name: spring-to-micronaut
description: >-
  Migrates Java/Maven services from Spring Boot to Micronaut 4. Covers
  annotation replacement, compile-time DI wiring, persistence migration
  from Spring Data JPA to Micronaut Data JDBC, HTTP client migration to
  @Client interfaces, security with intercept-url-map and
  BearerAuthenticationFetcher, serialization strategy choice
  (jackson-databind vs serde), multi-module Maven processor setup, and
  Dockerfile / docker-compose / Kubernetes env var alignment. Use when
  migrating a Spring Boot service to Micronaut, fixing 404 routes, fixing
  Introspection/serialization errors, or setting up Micronaut annotation
  processors in a multi-module Maven project.
compatibility: Micronaut 4.x, Maven 3.8+, Java 17+, Jakarta EE (jakarta.* namespace)
metadata:
  author: posadskiy
  stack: Micronaut 4, Micronaut Data JDBC, Micronaut Security JWT, Micronaut HTTP Client, Maven
---

# Spring Boot → Micronaut 4 Migration

Micronaut generates DI metadata and HTTP routes **at compile time** via annotation processors.
Every Spring pattern that works at runtime must be wired at build time instead.
This is the single mental model shift that prevents most migration bugs.

**See [references/annotation-map.md](references/annotation-map.md) for the full Spring → Micronaut translation table.**  
**See [references/pom-patterns.md](references/pom-patterns.md) for Maven POM templates.**

---

## Step 1 — POM: switch parent and serialization library

### Single-module service
Replace the Spring Boot parent with Micronaut BOM + explicit serialization choice.

```xml
<!-- Remove Spring Boot parent -->
<!-- Add Micronaut BOM in dependencyManagement -->
<properties>
  <micronaut.version>4.10.11</micronaut.version>
</properties>
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.micronaut.platform</groupId>
      <artifactId>micronaut-platform</artifactId>
      <version>${micronaut.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

**Serialization: choose one**

| Scenario | Dependency | Notes |
|----------|-----------|-------|
| Migrating existing Spring codebase, shared DTOs, no time to annotate every class | `io.micronaut:micronaut-jackson-databind` | All classes serializable by default, like classic Jackson. Prefer when you cannot modify DTO source (shared module). |
| New code or full migration, GraalVM native compatibility desired | `io.micronaut.serde:micronaut-serde-jackson` | Opt-in only: must add `@Serdeable` to every serialized type. More secure, smaller footprint. |

**If your DTOs are in a shared module you don't fully control, use `micronaut-jackson-databind`.**
Mixing both on the classpath can cause serialization surprises — pick one per service.

### Multi-module: `micronaut-inject-java` must run on EVERY module that uses Micronaut annotations

The #1 silent failure: if `micronaut-inject-java` is missing from the annotation processor path of even one module, its `@Controller`/`@Singleton`/`@Client` beans generate no route metadata → the server starts but every route returns **404**.

Put it in parent `pluginManagement` with `combine.children="append"` so each module inherits it and can add its own processors:

```xml
<!-- Parent pom.xml — pluginManagement -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.14.0</version>
  <configuration>
    <compilerArgs>
      <arg>-parameters</arg>       <!-- required for @QueryValue / @PathVariable name resolution -->
    </compilerArgs>
    <annotationProcessorPaths combine.children="append">
      <path>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
      </path>
      <path>
        <groupId>io.micronaut</groupId>
        <artifactId>micronaut-inject-java</artifactId>
        <version>${micronaut.version}</version>
      </path>
    </annotationProcessorPaths>
  </configuration>
</plugin>
```

Individual modules (e.g. `service`) add data/serde/validation processors with `combine.children="append"`:

```xml
<!-- service/pom.xml -->
<annotationProcessorPaths combine.children="append">
  <path>
    <groupId>io.micronaut.data</groupId>
    <artifactId>micronaut-data-processor</artifactId>
  </path>
  <path>
    <groupId>io.micronaut.validation</groupId>
    <artifactId>micronaut-validation-processor</artifactId>
  </path>
</annotationProcessorPaths>
```

See [references/pom-patterns.md](references/pom-patterns.md) for full templates.

---

## Step 2 — Entry point

```java
// Spring
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// Micronaut
public class Application {
    public static void main(String[] args) {   // MUST be public static void
        Micronaut.run(Application.class, args);
    }
}
```

`main` must be `public static void` — package-private causes the JVM to silently skip it as the entry point.

---

## Step 3 — Annotation replacement

See the full table in [references/annotation-map.md](references/annotation-map.md).

Key rules:
- `@Service` / `@Component` → `@Singleton` (from `jakarta.inject`)
- `@RestController` → `@Controller` (from `io.micronaut.http.annotation`)
- `@Autowired` → constructor injection (no annotation needed; Micronaut injects via constructor automatically)
- `@GetMapping("/path")` → `@Get("/path")`; same for `@Post`, `@Put`, `@Delete`
- `@RequestBody` → `@Body`; `@PathVariable` → `@PathVariable`; `@RequestParam` → `@QueryValue`
- `@Value("${key}")` → `@Value("${key}")` (same) or `@Property(name = "key")`

---

## Step 4 — Persistence: Spring Data JPA → Micronaut Data JDBC

Micronaut Data JDBC avoids runtime reflection (no Hibernate session, no lazy loading).

```java
// Entity: @MappedEntity instead of @Entity
@MappedEntity("lesson")
public class LessonEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    // ...
}

// Repository: @JdbcRepository, extend CrudRepository
@JdbcRepository(dialect = Dialect.POSTGRES)
public interface LessonRepository extends CrudRepository<LessonEntity, Long> {
    List<LessonEntity> findByCategoryId(Long categoryId);
}
```

**No lazy/eager fetch joins** — JDBC Data does not support them.
Load related collections explicitly in the service layer:

```java
// Service layer explicit loading instead of @OneToMany fetch
List<TaskEntity> tasks = taskRepository.findByLessonId(lesson.getId());
```

**Micronaut Data query method naming** is validated at **compile time**.
Invalid names (e.g. `findByTaskIds` when the field is `taskId`) fail the build — fix the method name or add `@Query`.

---

## Step 5 — HTTP clients: `@Client` interfaces

Replace Spring `RestTemplate`/`RestClient`/`WebClient`/`FeignClient` with Micronaut `@Client`.

```java
@Client("${service.urls.auth}")      // URL from application.yml
public interface AuthServiceClient {

    @Post("/login")
    AuthTokenResponse loginRequest(@Body AuthRequest request);

    // Business logic goes in default methods — no separate wrapper class needed
    default Optional<AuthResponse> login(AuthRequest request) {
        try {
            AuthTokenResponse r = loginRequest(request);
            return Optional.of(new AuthResponse(r.accessToken(), r.refreshToken(),
                Instant.now().plusMillis(r.expiresIn())));
        } catch (HttpClientResponseException e) {
            return Optional.empty();
        }
    }
}
```

- Micronaut generates the HTTP implementation at compile time.
- No separate `@Singleton` wrapper class needed — put business defaults on the interface.
- Inject `AuthServiceClient` directly where needed.

---

## Step 6 — Security

### Public vs authenticated routes via `intercept-url-map`

```yaml
micronaut:
  security:
    enabled: true
    authentication: bearer
    intercept-url-map:
      - pattern: /api/auth/**
        access: isAnonymous()
      - pattern: /api/users/register
        access: isAnonymous()
      - pattern: /api/**
        access: isAuthenticated()
```

### Delegated token validation (no local JWT signing)

When the service trusts an external auth-service to validate tokens:

```java
@Singleton
public class BearerAuthenticationFetcher implements AuthenticationFetcher<HttpRequest<?>> {
    private final AuthServiceClient authServiceClient;

    @Override
    public Publisher<Authentication> fetchAuthentication(HttpRequest<?> request) {
        return Mono.fromCallable(() -> {
            String token = extractBearerToken(request).orElse(null);
            if (token == null) return null;
            return authServiceClient.getUserIdFromToken(token)
                .map(id -> Authentication.build(id.toString()))
                .orElse(null);
        }).filter(Objects::nonNull);
    }

    @Override
    public int getOrder() { return 0; }
}
```

---

## Step 7 — DTOs in shared modules (multi-module validation)

When a DTO is used with `@Valid @Body` in a controller, Micronaut's validator needs a compile-time **bean introspection** for it. DTOs in a shared `domain` module are compiled by a separate processor pass — they need `micronaut-inject-java` in their own module.

**Shared `domain/pom.xml`:**

```xml
<properties>
  <micronaut.version>4.10.11</micronaut.version>
</properties>
<dependencyManagement>
  <!-- import micronaut-platform BOM same version as service -->
</dependencyManagement>
<dependencies>
  <!-- optional: service already has these; optional prevents transitive leak -->
  <dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-core</artifactId>
    <optional>true</optional>
  </dependency>
  <dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-inject</artifactId>
    <optional>true</optional>
  </dependency>
</dependencies>
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <configuration>
        <annotationProcessorPaths combine.children="append">
          <path>
            <groupId>io.micronaut</groupId>
            <artifactId>micronaut-inject-java</artifactId>
            <version>${micronaut.version}</version>
          </path>
        </annotationProcessorPaths>
      </configuration>
    </plugin>
  </plugins>
</build>
```

**Annotate validated DTOs:**

```java
@Introspected
public record AuthRequest(
    @NotBlank String username,
    @NotBlank String password
) {}
```

`@Introspected` (from `io.micronaut.core.annotation`) is enough for validation.
If you are using `micronaut-serde-jackson`, also add `@Serdeable`.
If you are using `micronaut-jackson-databind`, `@Introspected` alone suffices.

---

## Step 8 — Configuration file alignment

| Spring Boot | Micronaut |
|-------------|-----------|
| `spring.application.name` | `micronaut.application.name` |
| `server.port` | `micronaut.server.port` |
| `spring.datasource.*` | `datasources.default.*` |
| `spring.profiles.active=dev` | `MICRONAUT_ENVIRONMENTS=dev` env var |
| `application-dev.properties` | `application-dev.yml` (same naming convention) |
| `logging.level.root` | `logging.level.root` (same) |

```yaml
# application.yml
micronaut:
  application:
    name: my-service
  server:
    port: ${MY_SERVICE_PORT:8080}

datasources:
  default:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/mydb}
    username: ${DB_USER:user}
    password: ${DB_PASS:pass}
    driverClassName: org.postgresql.Driver
    schema-generate: NONE
    dialect: POSTGRES
```

---

## Step 9 — Logging

Micronaut uses Logback; `logback-spring.xml` is a Spring Boot convention and is NOT picked up.
Rename to `logback.xml`.

```xml
<!-- src/main/resources/logback.xml -->
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="STDOUT"/>
  </root>
</configuration>
```

Add `logback-classic` as a `runtime` dependency — `micronaut-runtime` does not pull it automatically in Micronaut 4.

---

## Step 10 — Dockerfile and docker-compose

```dockerfile
# Use wildcard to avoid hardcoding version
COPY target/service-*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```yaml
# docker-compose.yml
environment:
  MICRONAUT_ENVIRONMENTS: docker     # replaces SPRING_PROFILES_ACTIVE
  JAVA_SWING_TUTOR_DATABASE_URL: jdbc:postgresql://postgres:5432/mydb
```

---

## Common error → fix reference

| Error | Root cause | Fix |
|-------|-----------|-----|
| Every route returns `404` | `micronaut-inject-java` not in `annotationProcessorPaths`, so no bean definitions generated | Add processor; verify `META-INF/micronaut/` exists in compiled output |
| `500 No bean introspection … add @Introspected` | DTO in shared module was never processed by `micronaut-inject-java` | Add `@Introspected` to DTO + add processor to that module's POM |
| `500 No serializable introspection present … add @Serdeable` | Using `micronaut-serde-jackson` but DTO missing `@Serdeable` | Either add `@Serdeable` to DTO, or switch to `micronaut-jackson-databind` (simpler for migrations) |
| `SLF4J: No providers found` | `logback-classic` missing at runtime | Add `<artifactId>logback-classic</artifactId><scope>runtime</scope>` |
| Server starts on wrong port (8080) | Config not loading / `main` not `public static void` | Make `main` public; verify env var `MICRONAUT_ENVIRONMENTS` matches yml filename suffix |
| `@QueryValue`/`@PathVariable` not bound | `-parameters` compiler flag missing | Add `<arg>-parameters</arg>` to `maven-compiler-plugin` |
| Shade JAR starts but services not found | `ServicesResourceTransformer` missing from shade config | Add transformer (see [references/pom-patterns.md](references/pom-patterns.md)) |

---

## Industry notes (2025–2026)

- **GraalVM native images**: Micronaut's compile-time DI makes it far easier than Spring to compile to native.
  No reflection config files needed for your own beans. Add `micronaut-graalvm` and run `mn package -Dpackaging=native-image`.
- **Startup time**: 1–2 s for Micronaut vs 10–30 s for Spring Boot on equivalent services.
  Kubernetes readiness probes and horizontal scaling benefit directly.
- **Memory**: ~50–100 MB heap vs ~200–500 MB for Spring Boot.
- **`micronaut-spring` bridge** (`micronaut-projects/micronaut-spring`): lets you keep Spring annotations
  (`@Component`, `@Service`, `@RestController`, `@GetMapping`) and compile them to Micronaut.
  Useful for incremental migration without rewriting all annotations at once.
  Not all Spring features are supported; see the [compatibility matrix](https://micronaut-projects.github.io/micronaut-spring/4.0.0/guide/index.html#supportedAnnotations).
- **Compile-time errors** from annotation processors can be cryptic; always run `mvn clean compile`
  (not just incremental) after structural changes to get accurate diagnostics.
- **Breaking changes between major Micronaut versions are more frequent** than in Spring Boot.
  Pin `micronaut.version` explicitly rather than relying on transitive version resolution.
