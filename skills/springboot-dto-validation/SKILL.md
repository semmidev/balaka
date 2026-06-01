---
name: springboot-dto-validation
description: Guidelines for creating Data Transfer Objects (DTOs) and implementing Bean Validation. Use when defining API payloads and validating incoming requests.
---

# Spring Boot DTO & Validation Skill

Ensure data integrity at the edge of the application by following these DTO and validation patterns.

## 1. DTO Design (Java Records)
- **Use Records**: ALWAYS use Java `record` for Data Transfer Objects. Do not use standard classes or Lombok `@Data` classes for payloads. Records provide immutability, brevity, and automatic `equals`/`hashCode` implementations.
  ```java
  public record CreateTransactionRequest(
          String merchant,
          BigDecimal amount,
          LocalDate transactionDate
  ) {}
  ```
- Keep DTOs in the `com.artivisi.accountingfinance.dto` package or nested inside the Controller class if they are highly specific to a single endpoint.

## 2. Bean Validation
Rely exclusively on `jakarta.validation.constraints` to validate data. Avoid manual `if-null` validation logic inside services or controllers unless cross-field business logic is required.

- Add constraints directly to the record components.
  ```java
  public record CreateTransactionRequest(
          @NotBlank(message = "Merchant name is required")
          String merchant,
          
          @NotNull(message = "Amount is required")
          @DecimalMin(value = "0.01", message = "Amount must be strictly positive")
          BigDecimal amount
  ) {}
  ```
- Use `@Valid` in Controller method signatures to trigger the validation automatically:
  ```java
  @PostMapping
  public ResponseEntity<TransactionResponse> create(@Valid @RequestBody CreateTransactionRequest request) { ... }
  ```

## 3. Exception Handling Delegation
- **Do not catch validation exceptions**.
- The project has a global `@RestControllerAdvice` called `ApiExceptionHandler`. It is responsible for catching `MethodArgumentNotValidException` and `ConstraintViolationException`.
- The handler automatically transforms validation failures into a standardized JSON `ErrorResponse` containing a `fieldErrors` map.
- This ensures every API endpoint behaves consistently when returning 400 Bad Request errors.
