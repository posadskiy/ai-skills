# Maven POM Patterns for Spring Boot → Micronaut Migration

## Single-module service POM (minimal)

```xml
<properties>
  <micronaut.version>4.10.11</micronaut.version>
  <exec.mainClass>com.example.Application</exec.mainClass>
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

<dependencies>
  <!-- HTTP server -->
  <dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-http-server-netty</artifactId>
  </dependency>
  <!-- HTTP client (for @Client interfaces) -->
  <dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-http-client</artifactId>
  </dependency>
  <!-- Jackson databind (use for legacy/shared DTOs; swap for micronaut-serde-jackson for new greenfield) -->
  <dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-jackson-databind</artifactId>
  </dependency>
  <!-- Validation -->
  <dependency>
    <groupId>io.micronaut.validation</groupId>
    <artifactId>micronaut-validation</artifactId>
  </dependency>
  <!-- JDBC persistence -->
  <dependency>
    <groupId>io.micronaut.data</groupId>
    <artifactId>micronaut-data-jdbc</artifactId>
  </dependency>
  <dependency>
    <groupId>io.micronaut.sql</groupId>
    <artifactId>micronaut-jdbc-hikari</artifactId>
  </dependency>
  <!-- Security (JWT) -->
  <dependency>
    <groupId>io.micronaut.security</groupId>
    <artifactId>micronaut-security-jwt</artifactId>
  </dependency>
  <!-- Logging -->
  <dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <scope>runtime</scope>
  </dependency>
  <!-- YAML config support -->
  <dependency>
    <groupId>org.yaml</groupId>
    <artifactId>snakeyaml</artifactId>
    <scope>runtime</scope>
  </dependency>
</dependencies>

<build>
  <plugins>
    <!-- Shade plugin: produces executable fat JAR -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-shade-plugin</artifactId>
      <version>3.6.0</version>
      <executions>
        <execution>
          <phase>package</phase>
          <goals><goal>shade</goal></goals>
          <configuration>
            <transformers>
              <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                <mainClass>${exec.mainClass}</mainClass>
              </transformer>
              <!-- REQUIRED: merges META-INF/services files so Micronaut service discovery works -->
              <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
            </transformers>
            <filters>
              <filter>
                <artifact>*:*</artifact>
                <excludes>
                  <exclude>META-INF/*.SF</exclude>
                  <exclude>META-INF/*.DSA</exclude>
                  <exclude>META-INF/*.RSA</exclude>
                </excludes>
              </filter>
            </filters>
            <createDependencyReducedPom>false</createDependencyReducedPom>
          </configuration>
        </execution>
      </executions>
    </plugin>

    <!-- Compiler: annotation processors for Micronaut -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.14.0</version>
      <configuration>
        <compilerArgs>
          <arg>-parameters</arg>
        </compilerArgs>
        <annotationProcessorPaths combine.children="append">
          <path>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
          </path>
          <path>
            <!-- Core bean/route/client generation — MUST be present or routes return 404 -->
            <groupId>io.micronaut</groupId>
            <artifactId>micronaut-inject-java</artifactId>
            <version>${micronaut.version}</version>
          </path>
          <path>
            <groupId>io.micronaut.data</groupId>
            <artifactId>micronaut-data-processor</artifactId>
          </path>
          <path>
            <groupId>io.micronaut.validation</groupId>
            <artifactId>micronaut-validation-processor</artifactId>
          </path>
        </annotationProcessorPaths>
      </configuration>
    </plugin>
  </plugins>
</build>
```

---

## Multi-module: parent POM (root)

The parent POM is Spring Boot-based (other modules) but the service module is Micronaut.
Use `combine.children="append"` so each module can add its own processors without
overriding the parent's Lombok entry.

```xml
<!-- Root pom.xml -->
<build>
  <pluginManagement>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.14.0</version>
        <configuration>
          <release>${java.version}</release>
          <annotationProcessorPaths>
            <!-- Lombok only at parent level — Micronaut modules append their own processors -->
            <path>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok</artifactId>
              <version>${lombok.version}</version>
            </path>
          </annotationProcessorPaths>
        </configuration>
      </plugin>
    </plugins>
  </pluginManagement>
</build>
```

## Multi-module: Micronaut service module

```xml
<!-- service/pom.xml -->
<properties>
  <micronaut.version>4.10.11</micronaut.version>
  <exec.mainClass>com.example.ServiceApplication</exec.mainClass>
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

<!-- ... dependencies as above ... -->

<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <configuration>
        <compilerArgs>
          <arg>-parameters</arg>
        </compilerArgs>
        <!-- combine.children="append" adds to parent's Lombok entry -->
        <annotationProcessorPaths combine.children="append">
          <path>
            <groupId>io.micronaut</groupId>
            <artifactId>micronaut-inject-java</artifactId>
            <version>${micronaut.version}</version>
          </path>
          <path>
            <groupId>io.micronaut.data</groupId>
            <artifactId>micronaut-data-processor</artifactId>
          </path>
          <path>
            <groupId>io.micronaut.validation</groupId>
            <artifactId>micronaut-validation-processor</artifactId>
          </path>
        </annotationProcessorPaths>
      </configuration>
    </plugin>
    <!-- shade plugin as above -->
  </plugins>
</build>
```

## Multi-module: shared domain module (DTOs with @Introspected)

When DTOs in this module are used with `@Valid @Body` in the Micronaut service module,
they must be introspected at compile time by the domain module's own processor pass.

```xml
<!-- domain/pom.xml -->
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

<dependencies>
  <!-- optional: annotation types only; consumer (service) provides the runtime -->
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
  <!-- validation API only - no runtime validator here -->
  <dependency>
    <groupId>jakarta.validation</groupId>
    <artifactId>jakarta.validation-api</artifactId>
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

**DTO source:**
```java
@Introspected   // io.micronaut.core.annotation.Introspected
public record AuthRequest(
    @NotBlank String username,
    @NotBlank String password
) {}
```

---

## Verify processor ran correctly

After `mvn clean compile`, check that Micronaut bean definitions exist:

```bash
find target/classes/META-INF/micronaut -type f | head -20
# Should show files like:
# ...io.micronaut.inject.BeanDefinitionReference/com.example.web.$AuthController$Definition
# ...io.micronaut.inject.BeanDefinitionReference/com.example.web.$AuthController$Definition$Intercepted$Definition
```

If `META-INF/micronaut/` is **empty or missing** → `micronaut-inject-java` processor did not run.
Check that `combine.children="append"` is set and the version matches the BOM.
