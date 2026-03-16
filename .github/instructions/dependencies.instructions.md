---
applyTo: "**/pom.xml,**/build.gradle,**/build.gradle.kts,**/gradle.properties"
---

# Dependency Analysis & Migration Instructions

When working on build files (pom.xml, build.gradle, build.gradle.kts), analyze all dependencies
and suggest changes based on the rules below. Always check every dependency against these rules
before making or suggesting changes.

## Step 1: Check the Vert.x BOM Version

The project should use the Vert.x BOM to manage dependency versions consistently.

### Maven — Use `vertx-stack-depchain` or `vertx-dependencies` BOM

```xml
<!-- Target: Vert.x 5.0.8 (latest stable as of March 2026) -->
<properties>
  <vertx.version>5.0.8</vertx.version>
</properties>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.vertx</groupId>
      <artifactId>vertx-stack-depchain</artifactId>
      <version>${vertx.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

### Gradle — Use platform BOM

```kotlin
// build.gradle.kts
val vertxVersion = "5.0.8"

dependencies {
    implementation(platform("io.vertx:vertx-stack-depchain:$vertxVersion"))
}
```

If the project is on Vert.x 4.x, suggest upgrading first to 4.5.25 (latest 4.x), resolving
deprecations, then to 5.0.8. Never suggest jumping directly from 4.0.x to 5.x without the
intermediate step.

## Step 2: Identify Removed Artifacts

The following artifacts have been removed in Vert.x 5. If any of these are found, flag them
and suggest the replacement:

| Removed Artifact | Replacement | Notes |
|---|---|---|
| `vertx-sync` | Virtual threads (built-in Java 21) | Remove dependency entirely. Use `vertx.executeBlocking(() -> ...)` or virtual thread verticles |
| `vertx-service-factory` | None | Remove. Deploy verticles directly |
| `vertx-maven-service-factory` | None | Remove. Use fat JAR packaging instead |
| `vertx-http-service-factory` | None | Remove. Use standard HTTP client |
| `vertx-web-api-contract` | `vertx-web-openapi-router` | Complete API change — see OpenAPI migration section |
| `vertx-rx-java` (RxJava 1) | `vertx-rx-java3` or Mutiny | RxJava 1 is EOL |

## Step 3: Identify Sunsetted Artifacts (Still Work but Discouraged)

These still work in 5.x but will be removed in 6.x. Flag with a warning and suggest the
replacement:

| Sunsetted Artifact | Replacement | Priority |
|---|---|---|
| `vertx-grpc` (Netty-based) | `vertx-grpc-client` + `vertx-grpc-server` | HIGH — rewrite uses native Vert.x HTTP/2 |
| `vertx-jdbc-client` | `vertx-pg-client`, `vertx-mysql-client`, or `vertx-sql-client` with JDBC pool | MEDIUM |
| `vertx-service-discovery` | `vertx-service-resolver` | LOW — works fine for now |
| `vertx-rx-java2` | `vertx-rx-java3` or Mutiny | MEDIUM — RxJava 2 is EOL upstream |
| `vertx-opentracing` | `vertx-opentelemetry` | HIGH — OpenTracing is archived |
| `vertx-unit` | `vertx-junit5` | MEDIUM — JUnit 5 is the standard |

## Step 4: Check for Renamed / Restructured Artifacts

| Old Artifact (4.x) | New Artifact (5.x) | Notes |
|---|---|---|
| `vertx-web-openapi` | `vertx-web-openapi-router` | Complete rewrite in 5.x |
| `vertx-core` (includes Launcher) | `vertx-core` + `vertx-launcher-application` | Launcher is now a separate module |
| No equivalent | `vertx-launcher-legacy-cli` | Only if you need backward-compatible CLI |
| No equivalent | `vertx-service-resolver` | New in 5.x, replaces service discovery |
| No equivalent | `vertx-uri-template` | New utility module in 5.x |

## Step 5: Check for New Required Dependencies

When migrating to 5.x, these dependencies may need to be added:

### Launcher (required if using main class deployment)

```xml
<!-- Replace io.vertx.core.Launcher with VertxApplication -->
<dependency>
  <groupId>io.vertx</groupId>
  <artifactId>vertx-launcher-application</artifactId>
</dependency>
```

Update the main class in the build plugin:

```xml
<!-- Maven shade/exec plugin -->
<mainClass>io.vertx.launcher.application.VertxApplication</mainClass>

<!-- Or Vert.x Maven Plugin -->
<plugin>
  <groupId>io.reactiverse</groupId>
  <artifactId>vertx-maven-plugin</artifactId>
  <version>2.0.1</version>
</plugin>
```

```kotlin
// Gradle
application {
    mainClass.set("io.vertx.launcher.application.VertxApplication")
}
```

### Virtual Threads (if replacing vertx-sync)

No dependency needed — Java 21+ has built-in virtual thread support. Just ensure:

```xml
<properties>
  <maven.compiler.source>21</maven.compiler.source>
  <maven.compiler.target>21</maven.compiler.target>
</properties>
```

## Step 6: Check Java Version Compatibility

| Vert.x Version | Minimum Java | Recommended Java |
|---|---|---|
| 4.0.x - 4.5.x | Java 8 | Java 11+ |
| 5.0.x | Java 11 | Java 21 (LTS, enables virtual threads) |

If the project targets Java 8, flag this as a blocker for Vert.x 5 migration.
Recommend upgrading to Java 21 for full feature support including virtual threads
and pattern matching.

## Step 7: Check Plugin Versions

### Maven — Vert.x Maven Plugin

```xml
<!-- For Vert.x 5.x projects -->
<plugin>
  <groupId>io.reactiverse</groupId>
  <artifactId>vertx-maven-plugin</artifactId>
  <version>2.0.1</version>
  <executions>
    <execution>
      <id>vmp</id>
      <goals>
        <goal>initialize</goal>
        <goal>package</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <verticle>com.example.MainVerticle</verticle>
  </configuration>
</plugin>
```

### Gradle — Vert.x Gradle Plugin

```kotlin
plugins {
    id("io.vertx.vertx-plugin") version "4.0.0"
}

vertx {
    mainVerticle = "com.example.MainVerticle"
}
```

## Step 8: Check for Common Companion Dependency Updates

When migrating Vert.x, also check these commonly paired dependencies:

| Dependency | Check For |
|---|---|
| Netty (`io.netty`) | Do NOT declare explicitly — let Vert.x BOM manage it. Vert.x 5 uses Netty 4.1.x and prepares for Netty 5 |
| Jackson (`com.fasterxml.jackson`) | Ensure compatible version — Vert.x 5.0.8 uses Jackson 2.17+ |
| SLF4J / Logback | `vertx-core-logging` is now a separate module. Add if using SLF4J |
| Micrometer | If using `vertx-micrometer-metrics`, ensure Micrometer 1.12+ |
| JUnit 5 | If migrating from `vertx-unit`, add `vertx-junit5` |
| Testcontainers | Update to 1.19+ for Java 21 compatibility |

## Step 9: Validate the Dependency Tree

After making changes, always suggest running:

```bash
# Maven — check for conflicts
mvn dependency:tree -Dverbose | grep conflict

# Maven — check for convergence issues
mvn enforcer:enforce

# Gradle — check for conflicts
./gradlew dependencies --configuration runtimeClasspath
```

## Output Format

When analyzing a build file, produce a summary like:

```
## Dependency Analysis Summary

### Blockers (must fix before migration)
- [ ] Java version must be upgraded to 11+ (currently 8)
- [ ] `vertx-sync` is removed in 5.x — replace with virtual threads

### Required Changes
- [ ] Update Vert.x BOM from 4.0.0 → 4.5.25 (phase 1), then → 5.0.8 (phase 2)
- [ ] Add `vertx-launcher-application` dependency
- [ ] Replace `io.vertx.core.Launcher` main class with `io.vertx.launcher.application.VertxApplication`

### Recommended Changes
- [ ] Replace `vertx-unit` with `vertx-junit5`
- [ ] Replace `vertx-opentracing` with `vertx-opentelemetry`

### No Action Needed
- `vertx-web` ✓ (available in 5.0.8 BOM)
- `vertx-pg-client` ✓ (available in 5.0.8 BOM)
```
