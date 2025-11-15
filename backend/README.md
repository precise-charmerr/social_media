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

How scope affects transitivity (with examples):

1. **compile** (default scope)
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <scope>compile</scope>
</dependency>
```
Why: Core framework classes needed throughout the application lifecycle. If your module depends on another module that uses spring-core, you'll get it transitively because it's needed for compilation and runtime.

2. **provided**
```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <scope>provided</scope>
</dependency>
```
Why: Servlet containers like Tomcat already include this. It's transitive for compilation (so other modules can compile against servlet APIs) but won't be packaged to avoid conflicts with the container's version.

3. **runtime**
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
```
Why: JDBC driver isn't needed for compilation (you code against JDBC interfaces), but required at runtime. Transitive because any module using your database layer needs the driver at runtime.

4. **test**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```
Why: Testing frameworks are only needed for testing. NOT transitive because your module's tests are internal - other modules using your code don't need your test dependencies.

### Concrete Example: Two-Module Scenario with runtime Scope

**Module A (data-access library)** — `pom.xml`:
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.27</version>
    <scope>runtime</scope>  <!-- Note: runtime scope -->
</dependency>
```

Module A exposes a public API for DB operations (e.g., `UserRepository.findById()`) that uses JDBC interfaces internally.

**Module B (app)** — `pom.xml`:
```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>data-access</artifactId>
    <version>1.0.0</version>
</dependency>
```

Module B depends on A but does not explicitly declare the mysql driver.

**What happens:**

1. **At compile time (B compiles):**
   - B's compiler sees types from A (UserRepository, etc.) which only use JDBC interfaces (java.sql.Connection, etc.).
   - The mysql-connector-java JAR is NOT on B's compile classpath (because it's scope=runtime in A).
   - B compiles successfully.

2. **At runtime (B runs):**
   - Maven puts mysql-connector-java on B's runtime classpath (transitive propagation of runtime scope).
   - When B calls A's methods, the JDBC driver is available, and database connections work.

**Dependency tree output** (from B's perspective):
```bash
$ mvn dependency:tree
...
com.example:app:jar:1.0.0
└── com.example:data-access:jar:1.0.0
    └── mysql:mysql-connector-java:jar:8.0.27:runtime
```

Notice `runtime` next to the mysql driver — it's pulled in transitively for runtime but is absent from B's compile classpath.

## Understanding Classpath

**What is a classpath?**

A classpath is a list of directories and JAR files that tells the Java compiler (javac) and Java runtime (java) where to find classes, interfaces, and resources. Think of it as a "search path" — when Java needs a class (e.g., `java.sql.Connection`), it searches through each location in the classpath in order until it finds the .class file or .class inside a JAR.

**Format of a classpath:**

On Linux/Mac, paths are separated by `:`. On Windows, they're separated by `;`.

Example (Linux):
```
/home/user/.m2/repository/org/springframework/spring-core/5.3.0/spring-core-5.3.0.jar:/home/user/.m2/repository/mysql/mysql-connector-java/8.0.27/mysql-connector-java-8.0.27.jar:/home/user/myapp/target/classes
```

This means: search in spring-core JAR, then mysql-connector JAR, then the local `target/classes` directory.

**Compile classpath vs Runtime classpath:**

- **Compile classpath**: used by `javac` when compiling `.java` files into `.class` files. The compiler needs to see type definitions (interfaces, classes) so it can check that your code is type-correct.
- **Runtime classpath**: used by the JVM (`java` command) when running your application. The JVM needs actual class implementations (.class bytecode) to execute them.

### Example Scenario

**Your project structure:**
```
myapp/
├── src/main/java/
│   └── com/example/App.java
├── target/
│   ├── classes/           # compiled .class files go here
│   └── dependency/        # Maven downloads JARs here
└── pom.xml
```

**Your pom.xml declares:**
```xml
<dependencies>
  <!-- Compile scope (default) -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>5.3.0</version>
    <scope>compile</scope>
  </dependency>
  
  <!-- Runtime scope -->
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.27</version>
    <scope>runtime</scope>
  </dependency>
  
  <!-- Test scope -->
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13.2</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

**Your code** (`src/main/java/com/example/App.java`):
```java
import org.springframework.core.io.Resource;  // from spring-core (compile scope)
import java.sql.Connection;                   // from JDK (always available)
import com.mysql.cj.jdbc.Driver;              // from mysql-connector (runtime scope)

public class App {
  public static void main(String[] args) {
    // Can use spring-core types at compile time
    Resource resource = ...;
    
    // Cannot use mysql-specific types here — they won't be available at compile time
    // Driver driver = new Driver();  // COMPILE ERROR if this is at the top level
  }
}
```

**Step 1: Compile (mvn compile)**

Maven runs `javac` with the compile classpath:

```
Compile Classpath = 
  /path/to/.m2/repository/org/springframework/spring-core/5.3.0/spring-core-5.3.0.jar
  /path/to/.m2/repository/org/springframework/spring-jcl/5.3.0/spring-jcl-5.3.0.jar
  (other transitive compile deps)
  /path/to/myapp/src/main/java/com/example/
```

Notice:
- ✅ spring-core is included (compile scope)
- ❌ mysql-connector-java is NOT included (runtime scope)
- ❌ junit is NOT included (test scope)

**Command** (simplified):
```bash
javac -cp /path/spring-core.jar:... src/main/java/com/example/App.java -d target/classes
```

**Result:**
- `target/classes/com/example/App.class` is created (compilation succeeds).
- If your code tried to import `com.mysql.cj.jdbc.Driver` at compile time, you'd get a compile error because the JAR isn't on the compile classpath.

**Step 2: Run (mvn exec:java or java -jar)**

Maven/Java runs the app with the runtime classpath:

```
Runtime Classpath = 
  /path/to/.m2/repository/org/springframework/spring-core/5.3.0/spring-core-5.3.0.jar
  /path/to/.m2/repository/mysql/mysql-connector-java/8.0.27/mysql-connector-java-8.0.27.jar
  /path/to/myapp/target/classes
```

Notice:
- ✅ spring-core is included (compile scope always included at runtime)
- ✅ mysql-connector-java is included (runtime scope is now available)
- ❌ junit is NOT included (test scope)

**Command** (simplified):
```bash
java -cp /path/spring-core.jar:/path/mysql-connector.jar:target/classes com.example.App
```

**Result:**
- The JVM loads and runs `com.example.App.class` from `target/classes`.
- When the app opens a JDBC connection, the mysql-connector JAR is on the classpath, so the driver can be loaded and used.

### Real Maven Commands to See Classpaths

**See the compile classpath:**
```bash
mvn dependency:build-classpath -Dmdep.outputFile=compile-cp.txt
cat compile-cp.txt
```

**See the runtime classpath:**
```bash
mvn dependency:build-classpath -Dmdep.classpathScope=runtime -Dmdep.outputFile=runtime-cp.txt
cat runtime-cp.txt
```

**See the test classpath (includes test scope):**
```bash
mvn dependency:build-classpath -Dmdep.classpathScope=test -Dmdep.outputFile=test-cp.txt
cat test-cp.txt
```

### Summary Table

| Scope | Compile Classpath | Runtime Classpath | Test Classpath | Transitive? |
|-------|-------------------|-------------------|----------------|------------|
| compile | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| provided | ✅ Yes | ❌ No | ✅ Yes | ✅ (for compile) |
| runtime | ❌ No | ✅ Yes | ✅ Yes | ✅ (for runtime) |
| test | ❌ No | ❌ No | ✅ Yes | ❌ No |

## How to check the dependency tree

Run this in the module directory (e.g., `backend`):
```bash
mvn dependency:tree               # show the whole direct + transitive dependency tree
mvn dependency:tree -Dincludes=groupId:artifactId  # filter for a specific dependency
```

This shows which libraries are pulled in directly and which are transitive (and the scope for each).

---

## Maven Lifecycle: clean and install

Understanding what happens when you run Maven commands is essential for effective project management.

### 1. `mvn clean`

**What it does:**
- Deletes the entire `target/` directory (and everything in it)
- Removes all previously compiled `.class` files, packaged JARs, and build artifacts
- Gives you a "clean slate" to start a fresh build

**Why use it:**
- Ensures no stale or outdated compiled files interfere with your new build
- Helps debug weird build issues

**Example:**
```bash
mvn clean
```

---

### 2. `mvn install`

**What it does (in detail):**

`mvn install` is actually a shortcut that runs the entire Maven default lifecycle up to and including the `install` phase. Here's what happens step by step:

#### **Phase 1: Validation**
- Maven checks if the project structure and `pom.xml` are correct
- Verifies all required information is present

#### **Phase 2: Download Dependencies**
- Maven reads your `pom.xml` and identifies all declared dependencies
- Checks your local repository (`~/.m2/repository`) for each dependency
- **If a dependency is missing locally**, Maven downloads it from the configured remote repository (e.g., Maven Central Repository) over the internet
- **Only happens once per dependency** — subsequent runs find it locally and skip the download

#### **Phase 3: Compile**
- Maven compiles all `.java` files in `src/main/java/` into bytecode (`.class` files)
- Stores compiled classes in `target/classes/`
- Uses the compile classpath, which includes all dependencies with scope `compile`, `provided`, and resolved transitive dependencies

#### **Phase 4: Test**
- Maven compiles test source code from `src/test/java/` into `target/test-classes/`
- Runs all tests using the test classpath
- **If any test fails, the build stops** (unless you explicitly skip tests)

#### **Phase 5: Package**
- Maven packages the compiled classes and resources into a distributable format
- Creates a `.jar` file (Java Archive) or `.war` file (for web apps)
- Stores the packaged artifact in `target/`
- Example: `target/my-app-1.0.jar`

#### **Phase 6: Install**
- Maven takes the packaged artifact (the `.jar` file) from `target/`
- Copies it into your **local Maven repository** (`~/.m2/repository`)
- Uses your project's Maven coordinates (groupId, artifactId, version) to determine the location
- **Why?** Other local projects can now declare this project as a dependency without rebuilding it from source

**Local Repository Path Format:**
```
~/.m2/repository/<groupId with slashes>/<artifactId>/<version>/<artifactId>-<version>.jar
```

**Example:**
- If your `pom.xml` has:
  ```xml
  <groupId>com.example</groupId>
  <artifactId>my-app</artifactId>
  <version>1.0</version>
  ```
- After `mvn install`, the jar will be at:
  ```
  ~/.m2/repository/com/example/my-app/1.0/my-app-1.0.jar
  ```

---

### Common `mvn install` Variations

**Compile and install (run all tests):**
```bash
mvn install
```

**Compile and install (skip running tests, but still compile them):**
```bash
mvn -DskipTests install
```

**Compile and install (skip compiling and running tests entirely):**
```bash
mvn -Dmaven.test.skip=true install
```

---

### `mvn clean install` (Most Common)

**What it does:**
- Runs `clean` first (removes the `target/` directory)
- Then runs `install` (downloads deps, compiles, tests, packages, installs)
- Provides a completely fresh build from scratch

**When to use:**
- Starting a new build cycle
- Debugging build issues
- Ensuring no old artifacts interfere

**Example:**
```bash
mvn clean install
```

---

### Important Notes

- **`install` does NOT upload to a remote repository** — that's done by `mvn deploy` (requires `distributionManagement` config in `pom.xml`)
- **`install` is for your local machine only** — places artifacts in `~/.m2/repository` so other projects on the same machine can use them as dependencies
- **Dependency downloads happen automatically** — you don't need to manually download JARs; Maven handles it
- **Transitive dependencies are resolved** — if your declared dependency also depends on other libraries, Maven pulls those in automatically (unless scope prevents it)

---
You can copy this into your repository README or keep it under `backend/` for quick reference.

## Three ways to define/set a classpath

**(for runtime)**

1. Using the `-cp` flag (one-off, per-run):
```bash
java -cp C:\path\to\lib1.jar;C:\path\to\lib2.jar;C:\path\to\classes com.example.App
```

2. Using the `CLASSPATH` environment variable (persistent for the shell session):
```bash
export CLASSPATH=/path/to/lib1.jar:/path/to/lib2.jar:/path/to/classes
java com.example.App
```

3. Using Maven (automatic — Maven builds the correct classpath from `pom.xml`):
```bash
mvn compile         # Maven sets up the compile classpath
mvn test            # Maven sets up the test classpath
mvn exec:java       # Maven sets up the runtime classpath and runs the app
```

**(for compile time)**

1. Using `javac -cp` to set the compile-time classpath:
```bash
javac -cp /path/to/lib.jar src/main/java/com/example/App.java -d target/classes
```

