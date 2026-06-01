---
name: springboot-logging-logback
description: Security and compliance guidelines for configuring Logback in Spring Boot. Use when adding new loggers or modifying log retention policies.
---

# Spring Boot Secure Logging (Logback) Skill

Logging is a critical security surface area. This project uses `logback-spring.xml` to enforce strict defense-in-depth measures against log injection and to meet compliance retention requirements.

## 1. Preventing CRLF Log Injection
- Malicious users can input strings containing Carriage Return / Line Feed (`\r\n`) characters to forge fake log entries (Log Injection).
- While application code should sanitize inputs using `LogSanitizer`, the `logback-spring.xml` enforces this at the framework level.
- **Rule**: All log patterns MUST include the `%replace` directive to strip newlines from the message payload:
  ```xml
  <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %logger{36} - %replace(%msg){'[\r\n]', ''}%n</pattern>
  ```

## 2. Security Audit Log Segregation
- Standard application logs (debug/info) are noisy and have a short retention period (30 days).
- Security events (logins, permission changes, sensitive data access) MUST be routed to a dedicated security audit log.
- **Rule**: Security events should be logged via `SecurityAuditService`.
- In `logback-spring.xml`, the `SECURITY_AUDIT` appender handles this specifically:
  - It writes to `logs/security-audit.log`.
  - It has a strict **365-day** retention policy (`<maxHistory>365</maxHistory>`) to meet compliance requirements.

## 3. Profile-Based Logging
- Use Spring profiles (`<springProfile name="prod">`) to adjust log levels.
- Production environments should elevate the root logger to `INFO` and suppress noisy framework logs (e.g., Hibernate, Spring core) to `WARN` to save disk space and reduce SIEM ingestion costs.
