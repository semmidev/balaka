---
name: springboot-rest-controllers
description: Best practices for implementing REST APIs, structuring endpoints, handling security, and auditing. Use when creating or modifying API Controllers.
---

# Spring Boot REST Controllers & API Skill

Controllers are strictly responsible for handling HTTP requests, delegating to the service layer, and returning standardized responses.

## 1. Controller Definition
- Place controllers in the `com.artivisi.accountingfinance.controller.api` package.
- Annotate with `@RestController`.
- Define the base path using `@RequestMapping("/api/...")`.
- Document the API using OpenAPI annotations, starting with `@Tag(name = "...", description = "...")` at the class level.

## 2. Request Handling
- Use `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`.
- Map path variables with `@PathVariable` and query parameters with `@RequestParam`.
- For request payloads, use `@RequestBody` accompanied by `@Valid` to enforce DTO constraints.

## 3. Response Structure
- Always wrap the response payload in `ResponseEntity<T>`.
- Use appropriate HTTP status codes (e.g., `HttpStatus.CREATED` for successful POST requests, `HttpStatus.OK` for GET/PUT, `HttpStatus.NO_CONTENT` for DELETE).

```java
@PostMapping
public ResponseEntity<TransactionResponse> createTransaction(
        @Valid @RequestBody CreateTransactionRequest request) {
    TransactionResponse response = transactionService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

## 4. Input Validation
- Combine `@Valid` and `@RequestBody` to automatically reject bad input and trigger the `ApiExceptionHandler`.

## 4. Security & Authorization
- Use `@PreAuthorize` at the method level to restrict access based on user scopes or roles.
  ```java
  @PreAuthorize("hasAuthority('SCOPE_transactions:post')")
  ```
- To fetch the current authenticated user's ID or username, retrieve the `Authentication` object from `SecurityContextHolder.getContext().getAuthentication()`.

## 5. Security Auditing
- API endpoints performing sensitive or significant actions (like bypassing draft workflows or deleting data) must log an audit trail.
- Inject `SecurityAuditService` and use `auditApiCall()` or `securityAuditService.log(...)` to record the action, the parameters, and the user executing it.
- When logging sensitive information, use `LogSanitizer.sanitize()` to prevent injection attacks or data leakage.
