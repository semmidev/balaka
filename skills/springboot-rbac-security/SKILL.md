---
name: springboot-rbac-security
description: Guidelines for managing Role-Based Access Control (RBAC), permission mapping, and authority matrices in Spring Boot. Use when adding new permissions or roles.
---

# Spring Boot Role-Based Access Control (RBAC) Skill

This project utilizes a centralized authority matrix rather than scattering permission definitions across the database or application properties. This pattern lives entirely within `Permission.java`.

## 1. Defining Permissions
- Permissions must be defined as `public static final String` constants.
- The naming convention is `[ENTITY]_[ACTION]` (e.g., `TRANSACTION_CREATE`, `REPORT_VIEW`).
- Do not use enums for individual permissions; string constants allow for easier integration with Spring Security's `@PreAuthorize("hasAuthority(Permission.TRANSACTION_CREATE)")`.

## 2. The Authority Matrix
- All mapping between `Role` and permissions occurs in a single, exhaustive method: `getPermissionsForRole(Role role)`.
- Use an exhaustive Java 14+ `switch` expression to ensure compile-time safety. If a new `Role` is added, the compiler will force you to update the matrix.
- Return an immutable `Set.of(...)` containing the constants for each role.

```java
public static Set<String> getPermissionsForRole(Role role) {
    return switch (role) {
        case ADMIN -> Set.of(
            DASHBOARD_VIEW,
            TRANSACTION_CREATE,
            // ...
        );
        case STAFF -> Set.of(
            DASHBOARD_VIEW,
            TRANSACTION_CREATE
        );
        case EMPLOYEE -> Set.of(
            OWN_PAYSLIP_VIEW
        );
    };
}
```

## 3. Additive Permissions
- A user may have multiple roles. 
- The final granted authorities are calculated by unionizing the permission sets from all assigned roles (e.g., `permissions.addAll(getPermissionsForRole(role))`).
- **Never design subtractive permissions**. A role should only grant access, never explicitly revoke it.
