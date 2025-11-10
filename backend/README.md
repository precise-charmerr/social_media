# Points learnt while doing backend (Spring Boot)

## details of pom.xml

- `<modelVersion>` — specifies which POM model is used.
- `<parent>` — inherits configuration from `spring-boot-starter-parent`. The parent provides sensible defaults (dependency versions and plugin configs).
  - You can inspect the effective POM with:
```bash
mvn help:effective-pom                # produce effective POM to stdout
mvn help:effective-pom -Doutput=effective-pom.xml  # write it to a file
```
- `<properties>` — used to set values like the Java version (e.g., Java 17).
- `<dependency>` — declares libraries (JARs) your project needs. Examples commonly used here:
  1. `spring-boot-starter-web` — REST APIs / web support
  2. `spring-boot-starter-data-jpa` — JPA / database support
  3. `mysql-connector-java` — MySQL JDBC driver
  4. `spring-boot-starter-security` — authentication & authorization
  5. `lombok` — generates boilerplate (getters/setters/constructors) at compile time
  6. `spring-boot-starter-test` — testing libraries
- `<build>` — configures the build process and plugins.
  - A common plugin is `spring-boot-maven-plugin` which enables `mvn spring-boot:run` and creating executable JARs.
  - You may exclude compile-time-only libs (like Lombok) from the final JAR so they don't increase artifact size — Lombok works by generating code at compile time, so it isn't needed at runtime.

## Transitive property of dependencies (short)

Transitive dependencies = "friends of your libraries" that Maven pulls in automatically. If you add library A and A depends on B, Maven adds B for you.

Example:
```xml
<!-- you declare A in your POM -->
<dependency>
  <groupId>com.example</groupId>
  <artifactId>A</artifactId>
  <version>1.0</version>
</dependency>
```

If A depends on B, and B depends on C, then your project will get A, B and C (unless one of them uses a scope like `test` which prevents it from being transitive into your final build).

How scope affects transitivity (brief):
- `compile` (default): available at compile/test/runtime — transitive.
- `provided`: available at compile/test but NOT bundled (container provides it) — still transitive to modules that expect it at compile time, but not packaged.
- `runtime`: not needed to compile, but included at runtime — transitive for runtime.
- `test`: only for tests — NOT transitive to consumers/runtime.

## How to check the dependency tree

Run this in the module directory (e.g., `backend`):
```bash
mvn dependency:tree               # show the whole direct + transitive dependency tree
mvn dependency:tree -Dincludes=groupId:artifactId  # filter for a specific dependency
```

This shows which libraries are pulled in directly and which are transitive (and the scope for each).

---
You can copy this into your repository README or keep it under `backend/` for quick reference.
