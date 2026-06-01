---
name: springboot-rate-limiting
description: Guidelines for implementing DDOS mitigation and API rate limiting using a Sliding Window algorithm. Use when exposing new public endpoints.
---

# Spring Boot API Rate Limiting Skill

To prevent DDOS attacks, resource exhaustion, and aggressive scraping, this application enforces strict rate limits per IP address via `RateLimitService`.

## 1. In-Memory Sliding Window Algorithm
- The project does not rely on Redis for rate limiting (reducing infrastructure complexity).
- It uses an in-memory Sliding Window algorithm built on `ConcurrentHashMap<String, RateLimitInfo>`.
- **Scheduled Cleanup**: To prevent memory leaks, you must periodically prune expired entries using `@Scheduled(fixedRate = 60000)` calling `rateLimitService.cleanup()`.

## 2. Capacity Buckets
Do not apply a global flat limit. Route endpoints to specific capacity buckets in the `RateLimitFilter`:
- **LOGIN**: Extremely strict (10 req / min). Protects against brute force.
- **API**: Strict (100 req / min). Protects REST controllers from abuse.
- **GENERAL**: Moderate (300 req / min). Protects standard Thymeleaf UI routes.

## 3. Reverse Proxy Support (`X-Forwarded-For`)
- **CRITICAL**: If the application is behind a reverse proxy (Nginx, AWS ALB), `request.getRemoteAddr()` will always return the proxy's internal IP. Rate limiting the proxy will block ALL users.
- Always normalize the IP by parsing the `X-Forwarded-For` header. Take the *first* IP in the comma-separated list, as it represents the true client.

```java
private String normalizeIp(String ipAddress) {
    if (ipAddress != null && ipAddress.contains(",")) {
        return ipAddress.split(",")[0].trim().toLowerCase();
    }
    return ipAddress != null ? ipAddress.toLowerCase() : "";
}
```
