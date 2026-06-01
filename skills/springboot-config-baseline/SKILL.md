---
name: springboot-config-baseline
description: Best practices for establishing the baseline configuration classes, security filters, and OpenAPI setup for a new Spring Boot application. Use when structuring the application's configuration layer.
---

# Spring Boot Configuration Baseline Skill

When scaffolding a new application, establishing a strong configuration and security baseline is mandatory. All configuration files should reside in the `com.artivisi.accountingfinance.config` and `com.artivisi.accountingfinance.security` packages.

## 1. Security Baseline (`SecurityConfig.java`)
- **Disable Default Forms**: Disable HTTP Basic and default login forms if the app relies on API keys, OAuth2, or custom token handling.
- **CSRF & CSP**: Always configure Cross-Site Request Forgery (CSRF) protection appropriately. Define a strict Content Security Policy (CSP).
- **Custom Filters**: Register necessary custom security filters explicitly in the security filter chain:
  - `RateLimitFilter`: To prevent abuse and DoS attacks.
  - `CspNonceFilter`: To attach dynamic nonces to the request for inline script execution.
  - `BearerTokenAuthenticationFilter` (if using token-based auth).

## 2. Global Web Configuration (`WebMvcConfig.java`)
- Implement `WebMvcConfigurer`.
- **CORS**: Define strict Cross-Origin Resource Sharing (CORS) rules. Do not use `*` for allowed origins in production.
- **Interceptors**: Register any custom interceptors (e.g., `FirstRunSetupInterceptor` to check if the database needs initial seeding, or `ThemeInterceptor` for UI preferences).

## 3. OpenAPI Documentation
- Provide a centralized `OpenApiConfig.java`.
- Define the API structure, global security requirements (e.g., Bearer auth headers), and server URLs based on the environment profiles.

## 4. Specific Utility Configurations
Depending on the project requirements, establish modular `@Configuration` classes early:
- `ThymeleafConfig.java`: If rendering views, configure layout dialects and security extras.
- Integrations: e.g., `TelegramApiConfig.java` or `GoogleCloudVisionConfig.java`. Ensure secrets are NEVER hardcoded, but injected via `@Value("${...}")` or `@ConfigurationProperties` mapped to the environment.

## 4. Environment Injection
- Use `@Value("${property.name}")` inside `@Configuration` classes, never hardcode paths or secret keys.

## Real-world Examples from Codebase

### `SecurityConfig.java`
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Value("${app.remember-me.key}")
    private String rememberMeKey;

    @Bean
    public SecurityFilterChain securityFilterChain(
            HttpSecurity http,
            UserDetailsService userDetailsService,
            BearerTokenAuthenticationFilter bearerTokenAuthenticationFilter) throws Exception {
        
        http
            .userDetailsService(userDetailsService)
            .addFilterBefore(bearerTokenAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .authorizeHttpRequests(auth -> {
                auth.requestMatchers("/css/**", "/js/**", "/img/**").permitAll()
                    .requestMatchers("/login", "/error", "/setup/**").permitAll()
                    .requestMatchers("/v3/api-docs/**", "/swagger-ui/**").permitAll()
                    .anyRequest().authenticated();
            })
            // CSRF ignored for API endpoints to allow REST client interoperability
            .csrf(csrf -> csrf.ignoringRequestMatchers("/*/api/**", "/api/**"))
            .cors(cors -> cors.disable())
            .formLogin(form -> form.loginPage("/login").defaultSuccessUrl("/dashboard", false).permitAll())
            .rememberMe(remember -> remember.key(rememberMeKey).tokenValiditySeconds(604800))
            
            // Security Headers
            .headers(headers -> headers
                .addHeaderWriter(new CspNonceHeaderWriter())
                .frameOptions(frame -> frame.deny())
                .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true).maxAgeInSeconds(31536000))
            );

        return http.build();
    }
}
```
