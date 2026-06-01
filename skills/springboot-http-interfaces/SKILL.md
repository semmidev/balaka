---
name: springboot-http-interfaces
description: Modern Spring Boot 3+ patterns for building REST clients using Declarative HTTP Interfaces instead of RestTemplate or Feign.
---

# Spring Boot Declarative HTTP Interfaces Skill

This project communicates with external APIs (like the Telegram Bot API) using Spring 6's **Declarative HTTP Interfaces**. 

**DO NOT USE `RestTemplate`, `WebClient`, or `OpenFeign`** to build REST clients. Always use this declarative `@HttpExchange` pattern.

## 1. Defining the Client Interface
- Create an interface representing the API endpoints.
- Annotate methods with `@GetExchange`, `@PostExchange`, `@PutExchange`, etc.
- Use standard Spring MVC annotations for parameters (`@RequestBody`, `@PathVariable`, `@RequestParam`).
- Define Request and Response DTOs as inline Java `record`s within the interface file to keep the payload shapes strongly coupled to the client.

```java
public interface TelegramApiClient {

    @PostExchange("/sendMessage")
    SendMessageResponse sendMessage(@RequestBody SendMessageRequest request);

    @GetExchange("/file/bot{token}/{filePath}")
    byte[] downloadFile(@PathVariable String token, @PathVariable String filePath);

    // Request/Response DTOs
    @JsonInclude(JsonInclude.Include.NON_NULL)
    record SendMessageRequest(Long chat_id, String text) {}
    record SendMessageResponse(Boolean ok, String description) {}
}
```

## 2. Bootstrapping the Client Factory
- Declarative interfaces need a proxy factory to instantiate them at runtime.
- Create an `@Configuration` class to bind the interface to a `RestClient`.
- Use `RestClientAdapter` and `HttpServiceProxyFactory` to generate the bean.

```java
@Configuration
public class TelegramApiConfig {

    @Bean
    public RestClient telegramRestClient(TelegramConfig config) {
        return RestClient.builder()
                .baseUrl("https://api.telegram.org/bot" + config.getToken())
                .build();
    }

    @Bean
    public TelegramApiClient telegramApiClient(RestClient restClient) {
        RestClientAdapter adapter = RestClientAdapter.create(restClient);
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();
        
        return factory.createClient(TelegramApiClient.class);
    }
}
```

## 3. Usage
- Simply inject the interface (`TelegramApiClient`) into your services using `@RequiredArgsConstructor`.
- Call the methods exactly as you would a local method. No boilerplate HTTP request construction is required.

## Real-world Examples from Codebase

### `TelegramApiClient.java`
```java
package com.artivisi.accountingfinance.service.telegram;

import com.fasterxml.jackson.annotation.JsonInclude;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.service.annotation.GetExchange;
import org.springframework.web.service.annotation.PostExchange;

/**
 * Spring HTTP Interface for Telegram Bot API.
 * Uses declarative HTTP clients with RestClient backing.
 */
public interface TelegramApiClient {

    @PostExchange("/sendMessage")
    SendMessageResponse sendMessage(@RequestBody SendMessageRequest request);

    @GetExchange("/file/bot{token}/{filePath}")
    byte[] downloadFile(@PathVariable String token, @PathVariable String filePath);

    // Request/Response DTOs defined as inline records
    @JsonInclude(JsonInclude.Include.NON_NULL)
    record SendMessageRequest(
        Long chat_id,
        String text,
        String parse_mode
    ) {}

    record SendMessageResponse(
        Boolean ok,
        String description
    ) {}
}
```
