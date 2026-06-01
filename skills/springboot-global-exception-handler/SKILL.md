---
name: springboot-global-exception-handler
description: Best practices for implementing a @RestControllerAdvice for structured, standardized JSON error responses. Use when handling new types of exceptions.
---

# Spring Boot Global Exception Handler Skill

Consistency in API error responses is vital for frontend integration. This project centralizes exception translation using a `@RestControllerAdvice` (e.g., `ApiExceptionHandler.java`).

## 1. The `@RestControllerAdvice` Component
- Always annotate the handler with `@RestControllerAdvice(basePackages = "com.artivisi.accountingfinance.controller.api")` to ensure it only intercepts REST calls, avoiding conflicts with Thymeleaf UI controllers.
- Use `@Slf4j` to log exceptions centrally before translating them to HTTP responses.

## 2. Standardized Error Response Structure
- Do not return raw strings or generic maps.
- All exceptions must be translated into a standard `ErrorResponse` JSON structure containing:
  - `code`: A machine-readable string constant (e.g., `"VALIDATION_ERROR"`, `"NOT_FOUND"`).
  - `message`: A human-readable summary.
  - `details`: An optional map (`Map<String, String>`) for field-specific errors.

## 3. Handling Bean Validation
- Intercept `MethodArgumentNotValidException` to handle `@Valid` / `@Validated` failures from DTOs.
- Extract the `BindingResult` and populate the `details` map where the key is the field name and the value is the constraint violation message.
- Map this to `HttpStatus.BAD_REQUEST` (400).

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidationException(MethodArgumentNotValidException ex) {
    Map<String, String> fieldErrors = new HashMap<>();
    for (FieldError error : ex.getBindingResult().getFieldErrors()) {
        fieldErrors.put(error.getField(), error.getDefaultMessage());
    }
    
    ErrorResponse error = new ErrorResponse("VALIDATION_ERROR", "Validation failed", fieldErrors);
    return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
}
```

## 4. Handling Entity State
- Intercept `EntityNotFoundException` and map it to `HttpStatus.NOT_FOUND` (404).
- Avoid exposing deep internal exceptions (like `SQLException`). Catch them and wrap them in custom runtime exceptions before they reach the advice layer.
