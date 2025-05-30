# Maven Dependency Scopes Overview

This document summarizes the different dependency scopes available in Maven. It explains each scope's classpath availability, transitivity, and the typical scenarios where they are used.

## Maven Dependency Scopes Table

| **Scope**    | **Compile Classpath** | **Runtime Classpath** | **Test Classpath** | **Transitive** | **Typical Usage**                                                                                                                                     |
|--------------|-----------------------|-----------------------|--------------------|----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| **compile**  | Yes                   | Yes                   | Yes                | Yes            | Default scope. Dependencies needed for compiling, running, and testing. Most libraries fall into this category.                                      |
| **provided** | Yes                   | No                    | Yes                | Yes            | For dependencies required during compilation but supplied by the runtime environment (e.g., servlet API, Java EE APIs).                                 |
| **runtime**  | No                    | Yes                   | Yes                | Yes            | For dependencies not needed during compilation but necessary at runtime, such as JDBC drivers or logging frameworks.                                   |
| **test**     | No                    | No                    | Yes                | No             | For dependencies used exclusively for testing purposes (e.g., JUnit, Mockito). They are not included in your final artifact.                           |
| **system**   | Yes                   | Not typically used    | Yes (if required)  | No             | Similar to provided but requires an explicit system path to locate the JAR. Use sparingly due to potential portability issues.                       |
| **import**   | N/A                   | N/A                   | N/A                | N/A            | Exclusively used within `<dependencyManagement>` for importing BOM (Bill Of Materials) configurations. Not part of the classpath.                     |

## Detailed Explanation

- **Compile Scope:**  
  This is the default scope in Maven. Dependencies marked with this scope are available during compilation, testing, and runtime. Due to its broad usage, you usually don't need to specify it explicitly.

- **Provided Scope:**  
  Use this scope for dependencies that the runtime environment provides, such as web containers supplying servlet APIs. This practice helps keep your packaged artifact lean by not including these externally provided libraries.

- **Runtime Scope:**  
  Dependencies required when your application is running, but not at compile time, fall under this scope. Typical examples include JDBC drivers or logging frameworks that are only utilized during execution.

- **Test Scope:**  
  Test scope dependencies are used solely for testing. They help ensure that your production code remains independent of libraries strictly for testing, such as JUnit or Mockito.

- **System Scope:**  
  Similar in concept to provided scope, but it requires specifying the exact path to the dependency's JAR on your file system. Use it cautiously as it can reduce the portability of your project.

- **Import Scope:**  
  This special scope is used only within the `<dependencyManagement>` section. It allows the import of BOMs to centralize dependency version management, and it doesn't contribute to the compile, runtime, or test classpath.
