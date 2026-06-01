---
name: springboot-project-bootstrap
description: Guidelines for bootstrapping a new Spring Boot project, configuring the Maven pom.xml, overriding dependencies for CVEs, and setting up build plugins. Use when setting up a new repository or modifying root build configurations.
---

# Spring Boot Project Bootstrap Skill

When starting a new project based on this architecture, the foundation is built in the `pom.xml`. Follow these strict guidelines to ensure security, modern Java features, and frontend integration.

## 1. Core Platform Versions
- **Java Version**: MUST be Java 25 (`<java.version>25</java.version>`).
- **Spring Boot**: Use `4.0.x` series.

## 2. Dependency Overrides (Security First)
Do not blindly rely on Spring Boot's managed dependency versions if they contain known CVEs. You must explicitly override them in the `<properties>` block of `pom.xml`:
- Override `jackson-bom.version` and `jackson-2-bom.version`.
- Override `tomcat.version`.
- Override `postgresql.version` (e.g., to fix SCRAM DoS vulnerabilities).
Always check for Dependabot alerts and establish these overrides on day zero.

## 3. Frontend Integration via Maven
If the project includes a frontend (e.g., Node/React/Vue or specific Thymeleaf assets built via NPM), tie it to the Maven build using the `frontend-maven-plugin`.
- Configure the plugin to install a specific Node (`v22.x.x`) and NPM (`11.x.x`) version locally in the project.
- Bind the `npm run build` execution to the Maven `generate-resources` phase.

## 4. Code Quality & Security Plugins
A secure and maintainable project requires strict build-time checks:
- **Jacoco**: Configure `jacoco-maven-plugin` to enforce test coverage thresholds (e.g., minimum 80% instruction coverage) during the `verify` phase.
- **SpotBugs**: Include `spotbugs-maven-plugin` attached to the `verify` phase to catch common bugs statically.
- **SBOM Generation**: Include `cyclonedx-maven-plugin` to automatically generate a Software Bill of Materials (SBOM) during `package`.
- **Dependency Check**: Optionally configure `dependency-check-maven` for CI environments to scan for vulnerable dependencies.
- **Environment Variables**: Create `.env.example` to track required variables without committing secrets.

## 5. Standard Starters
Always include:
- `spring-boot-starter-webmvc`
- `spring-boot-starter-data-jpa`
- `spring-boot-starter-security`
- `spring-boot-starter-validation`
- `spring-boot-starter-flyway` (with `flyway-database-postgresql`)
