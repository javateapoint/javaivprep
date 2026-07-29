# Maven Dependency Scopes & Transitive Dependency Resolution

A reference guide for Apache Maven dependency management, covering direct vs. transitive dependencies, resolution mediation rules, dependency exclusions, and dependency scopes.

---

## 1. Direct vs. Transitive Dependencies

### Direct Dependencies
Direct dependencies are libraries explicitly declared inside the `<dependencies>` block of your project's `pom.xml`.

```xml
<dependencies>
    <!-- Direct Dependency on Spring Boot Starter Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>3.3.2</version>
    </dependency>
</dependencies>
```

### Transitive Dependencies
Transitive dependencies are dependencies required by your direct dependencies. Maven automatically resolves and downloads transitive libraries to complete your application runtime classpath.

#### Dependency Exclusions
If a transitive dependency conflicts with your application (e.g., duplicate logging bridges or unwanted vulnerable versions), you can explicitly exclude it:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.3.2</version>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

---

## 2. Dependency Scope Matrix

| Scope | Compile Classpath | Runtime Classpath | Test Classpath | Transitive | Typical Usage & Examples |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`compile`** | Yes | Yes | Yes | Yes | **Default Scope**. Required for compiling, testing, and running (e.g., `spring-web`, `jackson-databind`). |
| **`provided`** | Yes | No | Yes | No | Required for compilation but supplied by the container at runtime (e.g., `lombok`, `servlet-api`). |
| **`runtime`** | No | Yes | Yes | Yes | Not required for compilation, but needed during application execution (e.g., JDBC Drivers like `postgresql`, `h2`). |
| **`test`** | No | No | Yes | No | Used exclusively for testing; omitted from final build artifact (e.g., `junit-jupiter`, `mockito-core`). |
| **`system`** | Yes | No | Yes | No | Similar to `provided`, but explicitly points to a local system file path via `<systemPath>`. (Discouraged for portable builds). |
| **`import`** | N/A | N/A | N/A | N/A | Used **only** inside `<dependencyManagement>` to import Bill of Materials (BOM) configurations (e.g., `spring-boot-dependencies`). |

---

## 3. Maven Transitive Conflict Resolution Rules

When multiple dependencies pull in different versions of the same transitive library, Maven resolves conflicts using the following precedence rules:

1. **Nearest Definition Wins**: The version closest to the root `pom.xml` in the dependency tree takes precedence.
2. **First Declaration Wins**: If two versions exist at the exact same depth in the tree, the one declared first in the `pom.xml` wins.
3. **Explicit Override via `<dependencyManagement>`**: Declaring an explicit version in `<dependencyManagement>` forces all transitive occurrences to use that version across the entire project.

```xml
<dependencyManagement>
    <dependencies>
        <!-- Enforces explicit version across all transitive references -->
        <dependency>
            <groupId>org.yaml</groupId>
            <artifactId>snakeyaml</artifactId>
            <version>2.2</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```
