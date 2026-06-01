---
name: springboot-brute-force-protection
description: Implementation guidelines for defending against login brute force and account enumeration attacks. Use when building authentication logic.
---

# Spring Boot Brute Force Protection Skill

This project implements robust protection against credential stuffing, brute force, and account enumeration attacks natively through the `LoginAttemptService` and Spring Security events.

## 1. Tracking by Username, Not IP
- A core security philosophy here is to track failed attempts by **username**, not solely by IP address.
- Tracking by IP is easily bypassed by botnets rotating through proxies.
- By tracking by username, we immediately lock the targeted account after `MAX_ATTEMPTS` (5), regardless of where the attacker is coming from.

## 2. Preventing Enumeration (Timing Attacks)
- During login, the response time must be identical whether the username exists or not.
- The `LoginAttemptService` uses a `ConcurrentHashMap` caching layer to track failures *before* touching the database, ensuring that an attacker cannot infer if a user exists based on DB latency.

## 3. Account Lockout Mechanisms
- When a user hits the threshold (5 attempts), the account is locked for `LOCK_TIME_MINUTES` (30 minutes).
- **Critical Integration**: You must wire this into Spring Security by listening to `AuthenticationSuccessEvent` and `AbstractAuthenticationFailureEvent` via an `@EventListener` (e.g., `AuthenticationEventListener`).
  
```java
@EventListener
public void authenticationFailed(AbstractAuthenticationFailureEvent event) {
    String username = event.getAuthentication().getName();
    loginAttemptService.loginFailed(username);
}
```

## 4. Security Logging
- Always use the `LogSanitizer` before logging the username during a failed attempt to prevent log injection.
- Emitting standard log formats allows SIEMs (like Datadog or Splunk) to trigger real-time alerts on distributed brute force attacks.
