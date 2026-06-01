---
name: springboot-testing
description: Guidelines for writing unit and integration tests using JUnit 5, Mockito, MockMvc, and Testcontainers. Use when adding tests for features or bug fixes.
---

# Spring Boot Testing Strategy Skill

Ensuring correctness via automated testing is critical. Follow these patterns for unit and integration testing.

## 1. Unit Testing (Services)
- Focus unit tests on the Service layer to verify business logic.
- Use Mockito (`@ExtendWith(MockitoExtension.class)`) to mock dependencies.
- Use `@InjectMocks` on the service being tested, and `@Mock` for its dependencies (like repositories).
- Avoid loading the Spring Context for pure business logic unit tests to keep test execution fast.

## 2. Integration Testing (Controllers & Repositories)
Integration tests verify the HTTP layer and database interactions.

### Setup Annotations
- Use `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)`.
- Enable Testcontainers by importing the configuration: `@Import(TestcontainersConfiguration.class)`. This ensures a real PostgreSQL database is spun up via Docker for the tests.
- Activate the test profile: `@ActiveProfiles("test")`.
- Use `@Transactional` on the test class or methods so changes are rolled back automatically after each test, ensuring a clean slate.

## 5. Testcontainers
- For repository tests, use `@DataJpaTest` alongside `@Testcontainers` to spin up a real PostgreSQL instance.

## Real-world Examples from Codebase

### `ReceiptParserServiceTest.java`
Demonstrates clean usage of `@Nested`, `@DisplayName`, and Java 15+ Text Blocks for complex string inputs.

```java
@DisplayName("ReceiptParserService Tests")
class ReceiptParserServiceTest {

    private ReceiptParserService parser;

    @BeforeEach
    void setUp() {
        parser = new ReceiptParserService();
    }

    @Nested
    @DisplayName("Receipt Type Detection")
    class ReceiptTypeDetectionTests {

        @Test
        @DisplayName("Should detect Jago Syariah receipt")
        void shouldDetectJagoSyariahReceipt() {
            String ocrText = """
                Bank Jago Syariah
                Transfer Berhasil
                Rp 1.500.000
                """;

            ReceiptParserService.ParsedReceipt result = parser.parse(ocrText);

            assertThat(result).isNotNull();
            assertThat(result.receiptType()).isEqualTo("jago");
        }

        @Test
        @DisplayName("Should detect CIMB Octo receipt")
        void shouldDetectCimbOctoReceipt() {
            String ocrText = """
                OCTO Mobile
                Transfer Success
                IDR 2.500.000
                """;

            ReceiptParserService.ParsedReceipt result = parser.parse(ocrText);

            assertThat(result).isNotNull();
            assertThat(result.receiptType()).isEqualTo("cimb");
        }
    }
}
```

### Controller Testing (MockMvc)
- Autowire `WebApplicationContext` and build `MockMvc` manually, or use `@AutoConfigureMockMvc`.
- Use `mockMvc.perform(...)` to simulate HTTP requests.
- Verify status codes and validate JSON response structures using `jsonPath`.

```java
@Test
@DisplayName("Should create draft from valid text")
void shouldCreateDraftFromValidText() throws Exception {
    String requestJson = """
        {
            "merchant": "PLN",
            "amount": 350000,
            "currency": "IDR"
        }
    """;

    mockMvc.perform(post("/api/drafts/from-text")
            .contentType(MediaType.APPLICATION_JSON)
            .content(requestJson))
        .andExpect(status().isCreated())
        .andExpect(jsonPath("$.draftId").exists())
        .andExpect(jsonPath("$.amount").value(350000));
}
```

## 3. Security in Testing
- If the endpoint requires authentication, use Spring Security test annotations like `@WithMockUser(username = "testuser", roles = {"USER"})` on the test method or class to simulate an authenticated session.
