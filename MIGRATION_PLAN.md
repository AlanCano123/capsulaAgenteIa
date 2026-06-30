# Migration Plan: Java 8 → Java 25 & Spring Boot 2.7.x → 3.x

> **Project:** PhotoAlbum-Java  
> **Current Stack:** Java 8, Spring Boot 2.7.18, Hibernate 5.x (managed), Oracle DB  
> **Target Stack:** Java 25, Spring Boot 3.x, Hibernate 6.x, Jakarta EE 10  
> **Date:** 2026-06-30

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Dependency Audit](#dependency-audit)
3. [Risk Detection](#risk-detection)
4. [Baseline & Test Strategy](#baseline--test-strategy)
5. [Migration Phases](#migration-phases)
   - [Phase 0: Preparation & Baseline](#phase-0-preparation--baseline)
   - [Phase 1: Java 8 → Java 11 (LTS)](#phase-1-java-8--java-11-lts)
   - [Phase 2: Java 11 → Java 17 (LTS) + Spring Boot 3.x](#phase-2-java-11--java-17-lts--spring-boot-3x)
   - [Phase 3: Java 17 → Java 21 (LTS)](#phase-3-java-17--java-21-lts)
   - [Phase 4: Java 21 → Java 25](#phase-4-java-21--java-25)
6. [Docker & Infrastructure Updates](#docker--infrastructure-updates)
7. [Rollback Strategy](#rollback-strategy)

---

## Executive Summary

This document outlines the incremental migration of the PhotoAlbum application from **Java 8 / Spring Boot 2.7.18** to **Java 25 / Spring Boot 3.x**. The migration follows a **phase-by-phase approach** aligned with Java LTS releases, allowing incremental testing and validation at each step.

### Key Challenges

| Challenge | Impact | Severity |
|-----------|--------|----------|
| `javax.*` → `jakarta.*` namespace migration | All JPA entities, validation annotations must change | **Critical** |
| Spring Boot 2.7 → 3.x breaking changes | Configuration properties, Hibernate 6, security changes | **Critical** |
| Oracle JDBC driver upgrade (ojdbc8 → ojdbc11) | Required for Java 17+ compatibility | **High** |
| Removed Java APIs (Java EE modules) | `java.xml.bind` (JAXB) removed in Java 11 | **High** |
| Docker base images | Must update to Java 25 runtime images | **Medium** |
| Hibernate dialect changes | OracleDialect changes in Hibernate 6 | **Medium** |

---

## Dependency Audit

### Current Dependencies (from pom.xml)

| Dependency | Current Version | Java 11 | Java 17 | Java 21+ | Action Required |
|------------|----------------|---------|---------|----------|-----------------|
| Spring Boot (parent) | 2.7.18 | ✅ | ❌ | ❌ | **Upgrade to 3.x** (requires Java 17+) |
| spring-boot-starter-web | managed by 2.7.18 | ✅ | ❌ | ❌ | Managed by Spring Boot 3.x |
| spring-boot-starter-thymeleaf | managed by 2.7.18 | ✅ | ❌ | ❌ | Managed by Spring Boot 3.x |
| spring-boot-starter-data-jpa | managed by 2.7.18 | ✅ | ❌ | ❌ | Managed by Spring Boot 3.x (Hibernate 6) |
| spring-boot-starter-validation | managed by 2.7.18 | ✅ | ❌ | ❌ | Managed by Spring Boot 3.x (Jakarta) |
| spring-boot-starter-json | managed by 2.7.18 | ✅ | ❌ | ❌ | Managed by Spring Boot 3.x |
| **ojdbc8** | managed by 2.7.18 | ✅ | ❌ | ❌ | **Must upgrade to ojdbc11** |
| commons-io | 2.11.0 | ✅ | ✅ | ✅ | Update to 2.18.0+ (optional) |
| h2 (test) | managed by 2.7.18 | ✅ | ❌ | ❌ | Managed by Spring Boot 3.x |
| spring-boot-devtools | managed by 2.7.18 | ✅ | ❌ | ❌ | Managed by Spring Boot 3.x |
| spring-boot-starter-test | managed by 2.7.18 | ✅ | ❌ | ❌ | Managed by Spring Boot 3.x |

### Dependencies Requiring Immediate Attention

1. **🔴 Spring Boot 2.7.18** → Must upgrade to 3.x (requires Java 17+)
2. **🔴 ojdbc8** → Must upgrade to `ojdbc11` (or `ojdbc10` minimum) for Java 17+
3. **🟡 commons-io 2.11.0** → Compatible but should update to latest (2.18.0+)
4. **🟢 All other dependencies** → Managed by Spring Boot parent, will auto-upgrade

---

## Risk Detection

### 🔴 Critical: `javax.*` → `jakarta.*` Namespace Migration

**Files affected:**

| File | Current Import | Required Change |
|------|---------------|-----------------|
| `src/main/java/com/photoalbum/model/Photo.java` | `javax.persistence.*` | `jakarta.persistence.*` |
| `src/main/java/com/photoalbum/model/Photo.java` | `javax.validation.constraints.*` | `jakarta.validation.constraints.*` |
| `src/main/java/com/photoalbum/service/impl/PhotoServiceImpl.java` | `javax.imageio.ImageIO` | `java.awt.image.BufferedImage` (stays) + `javax.imageio` → `javax.imageio` (still valid) |

**Note:** `javax.imageio.ImageIO` is part of Java SE (not Java EE), so it remains `javax.imageio` and does NOT change to Jakarta. Only Java EE / javax.enterprise / javax.persistence / javax.validation packages change.

### 🔴 Critical: Spring Boot 3.x Breaking Changes

1. **Configuration properties renamed:**
   - `spring.datasource.driver-class-name` → still valid
   - `spring.jpa.database-platform` → still valid but OracleDialect class path changed in Hibernate 6
   - `spring.jpa.hibernate.ddl-auto` → still valid
   - `server.servlet.encoding.*` → still valid
   - `spring.servlet.multipart.*` → still valid

2. **Hibernate 6 changes:**
   - `org.hibernate.dialect.OracleDialect` → Use `org.hibernate.dialect.OracleDialect` (still works) or better: `org.hibernate.dialect.OracleDialect` in Hibernate 6.x
   - Native queries may need adjustment for Hibernate 6 compatibility
   - `@Lob` behavior changed for byte[] fields

3. **Spring Data JPA changes:**
   - `JpaRepository` still works but some method signatures changed
   - Native query result mapping may differ in Hibernate 6

### 🟡 Medium: Oracle JDBC Driver

- `ojdbc8` is compatible with Java 8, 11, but NOT Java 17+
- Must upgrade to `ojdbc11` (supports Java 11+) or `ojdbc10` (supports Java 8, 11, 17)
- **Recommendation:** Use `ojdbc11` for Java 17+ compatibility

### 🟡 Medium: Docker Images

- `maven:3.9.6-eclipse-temurin-8` → Must update to `maven:3.9.9-eclipse-temurin-25` (or latest)
- `eclipse-temurin:8-jre` → Must update to `eclipse-temurin:25-jre` (or `eclipse-temurin:21-jre` as intermediate)

### 🟢 Low: Code Style Improvements (Java 11+)

- `!photoOpt.isPresent()` → `photoOpt.isEmpty()` (Java 11+)
- `new HashMap<>()` diamond operator already used
- `new ArrayList<>()` diamond operator already used
- `Optional<Photo>empty()` → `Optional.empty()` (already used correctly)

### 🟢 Low: Removed APIs in Java 11+

- `java.xml.bind` (JAXB) → Not used in this project
- `java.xml.ws` (JAX-WS) → Not used in this project
- `java.corba` (CORBA) → Not used in this project
- `sun.misc.*` → Not used in this project
- `java.awt.image.BufferedImage` → Still part of Java SE, no issue

---

## Baseline & Test Strategy

### Current Test Suite

| Test File | Type | Framework |
|-----------|------|-----------|
| `PhotoAlbumApplicationTests.java` | Context load test | JUnit 5 + Spring Boot Test |

### How to Run Tests (Baseline)

```bash
# Run full test suite (uses H2 in-memory database)
mvn clean test

# Run with specific profile
mvn clean test -Dspring.profiles.active=test

# Run with verbose output
mvn clean test -X

# Generate test report
mvn clean test site
```

### Baseline Verification Steps

1. **Before any migration changes**, run the test suite and confirm all tests pass:
   ```bash
   mvn clean test
   ```
2. **Capture test output** and save as baseline reference
3. **Verify the application builds successfully:**
   ```bash
   mvn clean package -DskipTests
   ```
4. **Document the current test count, pass rate, and execution time**

### Test Strategy Per Phase

| Phase | Test Focus | Environment |
|-------|-----------|-------------|
| Phase 0 | Baseline tests, build verification | Current (Java 8) |
| Phase 1 | Compilation, unit tests | Java 11 |
| Phase 2 | Compilation, unit tests, integration tests | Java 17 + Spring Boot 3.x |
| Phase 3 | Full test suite, regression | Java 21 |
| Phase 4 | Full test suite, performance, Docker | Java 25 |

---

## Migration Phases

---

### Phase 0: Preparation & Baseline

**Goal:** Establish a reliable baseline before making any changes.

#### Tasks

- [ ] Run `mvn clean test` and save results as baseline
- [ ] Run `mvn clean package -DskipTests` and verify JAR builds
- [ ] Create a Git branch `migration/baseline` and tag the current state
- [ ] Verify Java 8 installation: `java -version`
- [ ] Verify Maven installation: `mvn -version`
- [ ] Document current test count and pass rate
- [ ] Set up CI/CD pipeline to run tests automatically (if not already)

#### Verification

```bash
mvn clean test 2>&1 | tee baseline-test-results.txt
mvn clean package -DskipTests 2>&1 | tee baseline-build-results.txt
```

---

### Phase 1: Java 8 → Java 11 (LTS)

**Goal:** Migrate to Java 11 while staying on Spring Boot 2.7.x.

#### Tasks

- [ ] **Update `pom.xml` properties:**
  ```xml
  <java.version>11</java.version>
  <maven.compiler.source>11</maven.compiler.source>
  <maven.compiler.target>11</maven.compiler.target>
  ```
- [ ] **Update `ojdbc8` to `ojdbc10`** (supports Java 8, 11, 17):
  ```xml
  <dependency>
      <groupId>com.oracle.database.jdbc</groupId>
      <artifactId>ojdbc10</artifactId>
      <scope>runtime</scope>
  </dependency>
  ```
- [ ] **Update `commons-io`** to latest compatible version (optional but recommended):
  ```xml
  <commons-io.version>2.18.0</commons-io.version>
  ```
- [ ] **Refactor `Optional` usage** (Java 11+ style):
  - `!photoOpt.isPresent()` → `photoOpt.isEmpty()`
  - Files: `PhotoServiceImpl.java` (line 181), `DetailController.java` (line 40), `PhotoFileController.java` (line 48)
- [ ] **Add `javax.annotation-api` dependency** (removed from JDK 11):
  ```xml
  <dependency>
      <groupId>javax.annotation</groupId>
      <artifactId>javax.annotation-api</artifactId>
      <version>1.3.2</version>
  </dependency>
  ```
- [ ] **Update `Dockerfile`** for Java 11:
  ```dockerfile
  FROM maven:3.9.6-eclipse-temurin-11 AS build
  FROM eclipse-temurin:11-jre
  ```
- [ ] **Run tests and fix any compilation errors**
- [ ] **Run full test suite** and verify all tests pass
- [ ] **Commit** to branch `migration/java-11`

#### Verification

```bash
mvn clean test
mvn clean package -DskipTests
```

#### Potential Issues

- `javax.annotation` package removed from JDK 11 → added as explicit dependency above
- `java.xml.bind` (JAXB) removed → not used in this project, no action needed
- `javax.imageio.ImageIO` is part of Java SE → still available, no change needed

---

### Phase 2: Java 11 → Java 17 (LTS) + Spring Boot 3.x

**Goal:** Upgrade to Java 17 AND Spring Boot 3.x simultaneously (Spring Boot 3.x requires Java 17+).

#### Tasks

- [ ] **Update `pom.xml` properties:**
  ```xml
  <java.version>17</java.version>
  <maven.compiler.source>17</maven.compiler.source>
  <maven.compiler.target>17</maven.compiler.target>
  ```
- [ ] **Update Spring Boot parent to 3.x:**
  ```xml
  <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>3.4.4</version>  <!-- Latest 3.x stable -->
      <relativePath/>
  </parent>
  ```
- [ ] **Migrate `javax.*` → `jakarta.*` in `Photo.java`:**
  - `javax.persistence.*` → `jakarta.persistence.*`
  - `javax.validation.constraints.*` → `jakarta.validation.constraints.*`
- [ ] **Update `ojdbc10` to `ojdbc11`** (required for Java 17+):
  ```xml
  <dependency>
      <groupId>com.oracle.database.jdbc</groupId>
      <artifactId>ojdbc11</artifactId>
      <scope>runtime</scope>
  </dependency>
  ```
- [ ] **Remove `javax.annotation-api`** (no longer needed with Jakarta EE 9+)
- [ ] **Update Hibernate dialect** in `application.properties` and `application-docker.properties`:
  ```properties
  # Hibernate 6.x - OracleDialect is still valid but use specific version
  spring.jpa.database-platform=org.hibernate.dialect.OracleDialect
  ```
  (Note: In Hibernate 6, `OracleDialect` is deprecated in favor of version-specific dialects like `Oracle12cDialect` → actually in Hibernate 6.x, `OracleDialect` maps to the latest supported version. Verify during migration.)

- [ ] **Review native queries** in `PhotoRepository.java` for Hibernate 6 compatibility:
  - `NVL()` function → still supported in Oracle and Hibernate 6
  - `ROWNUM` → still supported but consider migrating to `OFFSET FETCH` for modern Oracle
  - `TO_CHAR()` → still supported
  - `RANK() OVER` → still supported
- [ ] **Update `Dockerfile`** for Java 17:
  ```dockerfile
  FROM maven:3.9.9-eclipse-temurin-17 AS build
  FROM eclipse-temurin:17-jre
  ```
- [ ] **Review Spring Boot 3.x configuration changes:**
  - `spring.jpa.properties.hibernate.format_sql` → still valid
  - `spring.datasource.driver-class-name` → still valid
  - `spring.servlet.multipart.*` → still valid
  - Check for any deprecated property warnings in logs
- [ ] **Run tests and fix compilation errors**
- [ ] **Run full test suite** and verify all tests pass
- [ ] **Commit** to branch `migration/java-17-spring-boot-3`

#### Verification

```bash
mvn clean test
mvn clean package -DskipTests
```

#### Potential Issues

- **Hibernate 6 native query mapping:** The `@Query` annotations with `nativeQuery = true` may return different result types. The `findPhotosWithStatistics()` method returns `List<Object[]>` which may need adjustment.
- **`@Lob` + `byte[]`:** Hibernate 6 handles BLOB/Lob mapping differently. May need `@Column(columnDefinition = "BLOB")` or `@Lob @Basic(fetch = FetchType.LAZY)`.
- **`spring.jpa.hibernate.ddl-auto=create`:** In production, this should be `validate` or `update`. Consider changing for safety.
- **Thymeleaf:** Spring Boot 3.x uses Thymeleaf 3.1.x. Check for any template compatibility issues.

---

### Phase 3: Java 17 → Java 21 (LTS)

**Goal:** Upgrade to Java 21, leveraging new language features.

#### Tasks

- [ ] **Update `pom.xml` properties:**
  ```xml
  <java.version>21</java.version>
  <maven.compiler.source>21</maven.compiler.source>
  <maven.compiler.target>21</maven.compiler.target>
  ```
- [ ] **Update `Dockerfile`** for Java 21:
  ```dockerfile
  FROM maven:3.9.9-eclipse-temurin-21 AS build
  FROM eclipse-temurin:21-jre
  ```
- [ ] **Refactor code to use Java 21 features (optional but recommended):**
  - Pattern matching for `instanceof` (if applicable)
  - Record classes for DTOs (e.g., `UploadResult` could become a record)
  - Text blocks for multi-line strings (e.g., native queries in `PhotoRepository.java`)
  - Sequenced collections (`List.getFirst()`, `List.getLast()`)
  - Virtual threads (Spring Boot 3.x supports virtual threads)
- [ ] **Run full test suite** and verify all tests pass
- [ ] **Commit** to branch `migration/java-21`

#### Verification

```bash
mvn clean test
mvn clean package -DskipTests
```

#### Potential Issues

- Minimal breaking changes from Java 17 → 21
- Some internal APIs may have been removed (check for any `--add-opens` JVM flags needed)

---

### Phase 4: Java 21 → Java 25

**Goal:** Final upgrade to Java 25 (latest).

#### Tasks

- [ ] **Update `pom.xml` properties:**
  ```xml
  <java.version>25</java.version>
  <maven.compiler.source>25</maven.compiler.source>
  <maven.compiler.target>25</maven.compiler.target>
  ```
- [ ] **Update `Dockerfile`** for Java 25:
  ```dockerfile
  FROM maven:3.9.9-eclipse-temurin-25 AS build
  FROM eclipse-temurin:25-jre
  ```
- [ ] **Update Maven wrapper** (if using `mvnw`) to latest version supporting Java 25
- [ ] **Check for any removed APIs or deprecations** in Java 22-25:
  - Java 22: No major removals affecting this project
  - Java 23: No major removals affecting this project
  - Java 24: No major removals affecting this project
  - Java 25: Verify against release notes
- [ ] **Run full test suite** and verify all tests pass
- [ ] **Run integration tests** with Oracle database (if available)
- [ ] **Performance test** to verify no regression
- [ ] **Commit** to branch `migration/java-25`

#### Verification

```bash
mvn clean test
mvn clean package -DskipTests
# If Oracle DB is available:
mvn clean verify -Pintegration-test
```

#### Potential Issues

- Java 22-25 may have additional restrictions on reflection/access to internal APIs
- May need `--add-opens` JVM flags for libraries that use reflection
- Check Spring Boot 3.x compatibility with Java 25 (Spring Boot 3.4.x should support Java 25)

---

## Docker & Infrastructure Updates

### Dockerfile Migration Path

| Phase | Build Image | Runtime Image |
|-------|------------|---------------|
| Current | `maven:3.9.6-eclipse-temurin-8` | `eclipse-temurin:8-jre` |
| Phase 1 | `maven:3.9.6-eclipse-temurin-11` | `eclipse-temurin:11-jre` |
| Phase 2 | `maven:3.9.9-eclipse-temurin-17` | `eclipse-temurin:17-jre` |
| Phase 3 | `maven:3.9.9-eclipse-temurin-21` | `eclipse-temurin:21-jre` |
| Phase 4 | `maven:3.9.9-eclipse-temurin-25` | `eclipse-temurin:25-jre` |

### docker-compose.yml

Review `docker-compose.yml` for any Java-version-specific configuration. The Oracle DB service should remain unchanged.

### CI/CD Pipeline

- Update GitHub Actions / Azure Pipelines to use the appropriate JDK version for each phase
- Add matrix testing to validate against multiple Java versions during transition

---

## Rollback Strategy

### Per-Phase Rollback

Each phase should be developed on a separate Git branch:

```
main (current: Java 8)
  └── migration/java-11
  │     └── migration/java-17-spring-boot-3
  │           └── migration/java-21
  │                 └── migration/java-25
```

### Rollback Procedure

1. **If a phase fails tests:**
   ```bash
   git checkout main  # or previous phase branch
   git branch -D migration/<failed-phase>
   ```

2. **If production deployment fails:**
   - Revert Docker image tag to previous version
   - Redeploy with previous image
   - Investigate and fix in development branch

### Safe Deployment

- Use blue/green deployment strategy
- Keep previous Docker images tagged with version
- Test with canary deployments before full rollout

---

## Summary of File Changes

| File | Phase 1 | Phase 2 | Phase 3 | Phase 4 |
|------|---------|---------|---------|---------|
| `pom.xml` | ✅ Java 11, ojdbc10 | ✅ Java 17, SB 3.x, ojdbc11, Jakarta | ✅ Java 21 | ✅ Java 25 |
| `Photo.java` | - | ✅ Jakarta imports | - | - |
| `PhotoServiceImpl.java` | ✅ Optional.isEmpty() | - | - | - |
| `DetailController.java` | ✅ Optional.isEmpty() | - | - | - |
| `PhotoFileController.java` | ✅ Optional.isEmpty() | - | - | - |
| `application.properties` | - | ✅ Hibernate dialect check | - | - |
| `application-docker.properties` | - | ✅ Hibernate dialect check | - | - |
| `Dockerfile` | ✅ Java 11 images | ✅ Java 17 images | ✅ Java 21 images | ✅ Java 25 images |

---

## Appendix: Useful Commands

```bash
# Check Java version
java -version

# Check Maven version
mvn -version

# Run tests
mvn clean test

# Build without tests
mvn clean package -DskipTests

# Build with tests
mvn clean package

# Check dependency tree
mvn dependency:tree

# Check for outdated dependencies
mvn versions:display-dependency-updates

# Check for plugin updates
mvn versions:display-plugin-updates

# Run with specific Java version (if multiple JDKs installed)
JAVA_HOME=/path/to/jdk-25 mvn clean test