---
name: springboot-service-layer
description: Guidelines for implementing business logic, transaction management, and dependency injection in Spring Boot services. Use when creating or modifying Service classes.
---

# Spring Boot Service Layer & Transactions Skill

The service layer is responsible for business logic, transaction boundaries, and orchestrating repository calls. Follow these patterns strictly.

## 1. Service Definition & Injection
- Annotate service classes with `@Service`.
- Use constructor injection via Lombok's `@RequiredArgsConstructor`. Do NOT use field injection (`@Autowired` on fields).
- Declare dependencies as `private final`.

```java
@Service
@RequiredArgsConstructor
public class TransactionService {
    private final TransactionRepository transactionRepository;
    private final SecurityAuditService securityAuditService;
}
```

## 2. Transaction Management
- Use `@Transactional(readOnly = true)` at the **class level**. This optimizes database performance by ensuring read-only queries don't flush the persistence context unnecessarily.
- Override with `@Transactional` (which implies `readOnly = false`) on **specific methods** that modify the database (insert, update, delete).

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ProjectService {
    
    // Uses class-level readOnly = true
    public Project findById(UUID id) { ... }

    // Overrides for write operations
    @Transactional
    public Project create(Project project) { ... }
}
```

## 3. Business Logic Location
- Keep business logic inside the Service layer. Do not put business logic in Controllers (which handle HTTP transport) or Entities (which handle data mapping).
- Throw domain-specific exceptions (like `EntityNotFoundException` or `IllegalStateException`) when business rules are violated, and let the global exception handler translate them to HTTP responses.

## 4. Complex Data Operations
- If you need to perform complex bulk deletes or cross-table operations that bypass JPA's standard cascade rules, you may inject `EntityManager` and execute native SQL updates.
- Ensure these methods are strictly wrapped in `@Transactional`.
