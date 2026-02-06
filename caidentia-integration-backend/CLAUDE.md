# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
# Build (includes tests)
./gradlew build

# Run application
./gradlew bootRun

# Run tests
./gradlew test

# Run a single test class
./gradlew test --tests "caidentia.integration.SomeTestClass"

# Run a single test method
./gradlew test --tests "caidentia.integration.SomeTestClass.testMethodName"

# Generate OpenAPI code from spec
./gradlew openApiGenerate

# Clean build
./gradlew clean build

# Code quality
./gradlew spotbugsMain    # SpotBugs analysis
./gradlew sonar           # SonarQube analysis

# Release management
./gradlew createRelease -PreleaseType=patch   # or minor, major
./gradlew pushRelease

# Publish to Nexus
./gradlew publish
```

## Architecture

This is a Spring Boot 3.4 microservice using Java 21 with an API-first development approach.

### Key Patterns

- **OpenAPI-First**: API specification in `src/main/resources/openapi/openapi.yaml` drives code generation. Controllers are generated as interfaces (with `Repository` suffix) that implementation classes must implement.
- **Generated Code**: OpenAPI Generator outputs to `build/generated/src/main/java`. Packages: `com.example.client.api`, `com.example.client.model`.
- **Custom Framework**: Uses internal `caidentia-boot-starter-*` dependencies for web, JPA, and security (from internal Nexus).

### Technology Stack

- **Framework**: Spring Boot 3.4.4, Spring Cloud 2024.0.0
- **Database**: PostgreSQL with Hibernate/JPA
- **Mapping**: MapStruct 1.6.3 with Lombok integration
- **Code Generation**: OpenAPI Generator 7.3.0 (Spring generator)

### Project Structure

```
src/main/
├── java/caidentia/           # Application code
└── resources/
    ├── application.yml       # Spring configuration (local profile on port 8081)
    └── openapi/openapi.yaml  # API specification
gradle/
└── openapi-generator.gradle  # OpenAPI code generation config
```

### Version Management

Uses Axion Release plugin with Git-based versioning. Release branches: `master`, `main`. Version increment via `-PreleaseType` property (patch/minor/major).
