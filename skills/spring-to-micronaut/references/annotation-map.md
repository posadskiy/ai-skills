# Spring Boot → Micronaut Annotation Map

## Core DI

| Spring Boot | Micronaut | Notes |
|-------------|-----------|-------|
| `@SpringBootApplication` | *(remove)* — call `Micronaut.run(App.class, args)` | |
| `@Component` | `@Singleton` (`jakarta.inject`) | Default scope is Singleton in both |
| `@Service` | `@Singleton` | No semantic difference in Micronaut |
| `@Repository` | `@Singleton` or `@JdbcRepository` (Data JDBC) | |
| `@Configuration` | `@Factory` (`io.micronaut.context.annotation`) | Bean factory methods |
| `@Bean` (on method) | `@Bean` (inside `@Factory` class) | |
| `@Autowired` | Constructor injection (no annotation needed) or `@Inject` | Prefer constructor injection |
| `@Qualifier("name")` | `@Named("name")` (`jakarta.inject`) | |
| `@Primary` | `@Primary` (`io.micronaut.context.annotation`) | |
| `@Lazy` | `@Requires` / `@Prototype` | Micronaut creates singletons at startup; use `@Prototype` for per-request scope |
| `@Scope` | `@Prototype`, `@RequestScope`, `@ThreadLocal` | |
| `@PostConstruct` | `@PostConstruct` (`jakarta.annotation`) | Same |
| `@PreDestroy` | `@PreDestroy` (`jakarta.annotation`) | Same |
| `@Value("${prop}")` | `@Value("${prop}")` or `@Property(name="prop")` | |
| `@ConfigurationProperties` | `@ConfigurationProperties` (`io.micronaut.context.annotation`) | |

## Web / HTTP

| Spring Boot | Micronaut | Notes |
|-------------|-----------|-------|
| `@RestController` | `@Controller` (`io.micronaut.http.annotation`) | |
| `@RequestMapping("/path")` | Combine `@Controller("/path")` with method annotations | |
| `@GetMapping("/path")` | `@Get("/path")` | |
| `@PostMapping` | `@Post` | |
| `@PutMapping` | `@Put` | |
| `@DeleteMapping` | `@Delete` | |
| `@PatchMapping` | `@Patch` | |
| `@RequestBody` | `@Body` | |
| `@PathVariable` | `@PathVariable` | Requires `-parameters` compiler flag |
| `@RequestParam` | `@QueryValue` | |
| `@RequestHeader` | `@Header` | |
| `@ResponseStatus` | Return `HttpResponse.status(...)` | |
| `ResponseEntity<T>` | `HttpResponse<T>` | |
| `@ExceptionHandler` | Implement `ExceptionHandler<E, R>` with `@Singleton` | |
| `@ControllerAdvice` | `@Singleton` + `ExceptionHandler<E, R>` + `@Produces` | |

## Validation

| Spring Boot | Micronaut | Notes |
|-------------|-----------|-------|
| `@Validated` (on class) | `@Validated` (`io.micronaut.validation`) | |
| `@Valid` (on parameter) | `@Valid` (`jakarta.validation`) | Requires `@Introspected` on the validated type |
| All `javax.validation.*` | `jakarta.validation.*` | Micronaut uses Jakarta EE namespace |
| No annotation on DTO | `@Introspected` | Required for DTOs used with `@Valid`; generates compile-time introspection |

## Persistence (Spring Data JPA → Micronaut Data JDBC)

| Spring Boot / JPA | Micronaut Data JDBC | Notes |
|-------------------|---------------------|-------|
| `@Entity` | `@MappedEntity("table_name")` | |
| `@Table(name="t")` | `@MappedEntity("t")` | Combined into one annotation |
| `@Id` | `@Id` (`io.micronaut.data.annotation`) | |
| `@GeneratedValue` | `@GeneratedValue(strategy=IDENTITY)` | |
| `@Column(name="col")` | `@MappedProperty("col")` | |
| `@OneToMany` / `@ManyToOne` | *(not supported)* — load relations explicitly | JDBC Data has no ORM-style joins |
| `@Transactional` | `@Transactional` (`io.micronaut.transaction.annotation`) | |
| `JpaRepository<T, ID>` | `CrudRepository<T, ID>` (`io.micronaut.data.repository`) | |
| `@Repository` interface | `@JdbcRepository(dialect = POSTGRES)` on interface | |
| `@Query("JPQL …")` | `@Query("SQL …")` | Native SQL only (JDBC Data) |
| Derived query methods (`findByName`) | Same naming, validated at **compile time** | Invalid names fail build |

## HTTP Client (RestTemplate / FeignClient → @Client)

| Spring | Micronaut | Notes |
|--------|-----------|-------|
| `@FeignClient(url="${url}")` | `@Client("${url}")` on interface | |
| `@GetMapping` on Feign method | `@Get` on `@Client` method | |
| `RestTemplate` / `RestClient` | `@Client` interface or `HttpClient` (injected) | Prefer declarative `@Client` |
| Separate service wrapper class | `default` methods on the `@Client` interface | No extra wrapper needed |

## Security (Spring Security → Micronaut Security)

| Spring Security | Micronaut Security | Notes |
|-----------------|-------------------|-------|
| `WebSecurityConfigurerAdapter` | `SecurityFilterChain` equivalent → `micronaut.security.intercept-url-map` in YAML | |
| `@PreAuthorize("isAuthenticated()")` | `intercept-url-map: access: isAuthenticated()` | YAML-based or `@Secured` |
| `@PreAuthorize("isAnonymous()")` | `intercept-url-map: access: isAnonymous()` | |
| `@PreAuthorize("hasRole('ADMIN')")` | `@Secured("ADMIN")` or `access: hasRole('ADMIN')` | |
| `SecurityContextHolder` | Inject `Authentication` parameter in controller | |
| `UserDetailsService` | Implement `AuthenticationProvider` | |
| `UsernamePasswordAuthenticationFilter` | `LoginHandler` or built-in `LoginController` | |
| Custom token filter | Implement `AuthenticationFetcher<HttpRequest<?>>` | For delegated external token validation |

## Testing

| Spring Boot | Micronaut | Notes |
|-------------|-----------|-------|
| `@SpringBootTest` | `@MicronautTest` | |
| `@Autowired` in test | `@Inject` | |
| `MockMvc` | `HttpClient` (injected) or `@Client` in test | |
| `@MockBean` | `@MockBean` (`io.micronaut.test.extensions.junit5`) or `@Replaces` | |
| `TestRestTemplate` | Inject `@Client("/")` in test | |

## Logging / Config files

| Spring Boot | Micronaut |
|-------------|-----------|
| `logback-spring.xml` | `logback.xml` (Spring-specific naming NOT picked up) |
| `application-dev.properties` | `application-dev.yml` (same suffix convention) |
| `SPRING_PROFILES_ACTIVE=dev` | `MICRONAUT_ENVIRONMENTS=dev` |
| `spring.application.name` | `micronaut.application.name` |
| `server.port` | `micronaut.server.port` |
| `spring.datasource.*` | `datasources.default.*` |
