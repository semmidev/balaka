---
name: springboot-mvc-thymeleaf
description: Best practices for building Spring MVC controllers that render Thymeleaf templates and handle HTMX fragments. Use when creating UI-facing controllers.
---

# Spring Boot MVC & Thymeleaf Skill

This application relies heavily on Server-Side Rendering (SSR) via Thymeleaf, enhanced by HTMX for dynamic interactions. When writing UI controllers, strictly follow these conventions.

## 1. Controller Annotation & Architecture
- Use `@Controller`, NOT `@RestController` (unless you are in the `controller/api` package).
- Secure the entire controller at the class level using `@PreAuthorize("hasAuthority(...)")`.

## 2. Eliminate Magic Strings (`ViewConstants`)
- NEVER hardcode view paths, model attribute names, or redirect URLs.
- Always use the centralized `ViewConstants.java` class.
  - `ATTR_CURRENT_PAGE`: Used by the layout to highlight active navigation menus.
  - `PAGE_*`: Constants for identifying the current page (e.g., `PAGE_TRANSACTIONS`).
  - `REDIRECT_*`: Constants for standard redirects (e.g., `"redirect:/invoices/"`).
  - `VIEW_*`: Constants for Thymeleaf template paths.

## 3. Populating the Model
- Controllers must inject data into the `Model` object.
- Always set `ATTR_CURRENT_PAGE` so the sidebar navigation works correctly.

```java
@GetMapping
public String list(Model model) {
    model.addAttribute(ViewConstants.ATTR_CURRENT_PAGE, ViewConstants.PAGE_TRANSACTIONS);
    model.addAttribute(ViewConstants.ATTR_TRANSACTIONS, transactionService.findAll());
    return "transactions/list";
}
```

## 4. HTMX Fragment Responses
- To support seamless UI updates without full page reloads, check for the `HX-Request` header.
- If it's an HTMX request, return only a specific Thymeleaf fragment using the `::` syntax.
- Otherwise, return the full page view.

```java
@GetMapping
public String list(@RequestHeader(value = "HX-Request", required = false) String hxRequest, Model model) {
    // ... populate model ...
    if ("true".equals(hxRequest)) {
        return "fragments/transaction-table :: table";
    }
    return "transactions/list";
}
```
