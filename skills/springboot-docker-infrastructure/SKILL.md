---
name: springboot-docker-infrastructure
description: Guidelines for setting up local development databases via Docker Compose and structuring multi-stage Dockerfiles for Spring Boot applications. Use when configuring container infrastructure.
---

# Spring Boot Docker & Infrastructure Skill

A robust and secure containerization strategy is essential for both local development and production deployments.

## 1. Local Database Environment (`compose.yml`)
When setting up local PostgreSQL, always mirror production security defaults as closely as possible.
- **Forced SSL**: Configure the Postgres container to require SSL (`postgres -c ssl=on -c ssl_cert_file=...`).
- **Initialization Container**: Use a lightweight `alpine` container (e.g., `db-ssl-init`) mapped to a shared volume to automatically generate self-signed certificates using `openssl` before the database service starts.
- **Depends_On**: Ensure the database service has a `depends_on: db-ssl-init` condition (`service_completed_successfully`).
- **Volumes**: Persist database data to a local directory (e.g., `./db-accounting:/var/lib/postgresql/data`) to prevent data loss across container restarts.

## 2. Multi-stage Dockerfile
Do not build "fat jars" directly into a single container layer. Use a layered approach to maximize Docker caching.

### Stage 1: Build & Extract
- **Base Image**: Use a Maven image matching the target Java version (e.g., `maven:3.9-eclipse-temurin-25-alpine`).
- **Dependency Cache**: Run `mvn dependency:go-offline` before copying the source code to cache dependencies in a Docker layer.
- **Spring Boot Layer Extraction**: After packaging the jar, extract it using Spring Boot's built-in tools:
  ```bash
  java -Djarmode=tools -jar app.jar extract --layers --destination layers
  ```

### Stage 2: Runtime
- **Base Image**: Use a minimal JRE image (e.g., `azul/zulu-openjdk-alpine:25-jre`).
- **Security**: Create a non-root user (`addgroup -S app && adduser -S app -G app`) and run the container as this `USER`.
- **Copy Layers**: Copy the extracted layers from the build stage in order from least frequently changed to most frequently changed:
  1. `dependencies/`
  2. `spring-boot-loader/`
  3. `snapshot-dependencies/`
  4. `application/`
- **Entrypoint**: Use `tini` as the init system to prevent zombie processes and handle OS signals properly.
  ```dockerfile
  ENTRYPOINT ["/sbin/tini", "--", "sh", "-c", "exec java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
  ```
- **Healthcheck**: Implement a `HEALTHCHECK` pinging the Spring Boot Actuator `/actuator/health/liveness` endpoint.
