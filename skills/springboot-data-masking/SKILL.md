---
name: springboot-data-masking
description: Best practices for masking and redacting sensitive PII before rendering it to the UI or API. Use when displaying financial data.
---

# Spring Boot PII Data Masking Skill

While `EncryptedStringConverter` protects data at rest in the database, `DataMaskingUtil` protects data in transit and on the screen by redacting it before it leaves the controller layer.

## 1. When to Mask Data
- Standard users should **never** see full, unredacted NIKs, NPWPs, Bank Account Numbers, or phone numbers.
- Apply masking in the Controller or DTO Mapper layer before the data is passed to the Thymeleaf `Model` or serialized to JSON.

## 2. Using `DataMaskingUtil`
- Use the provided static utility methods for consistency. Do not write custom regex masking on a per-controller basis.
- `maskNik(String)`: Shows first 4 and last 4. Example: `3201********0001`
- `maskNpwp(String)`: Shows first 4 and last 3. Example: `12.3***********345`
- `maskBankAccount(String)`: Shows first 3 and last 3. Example: `123****890`
- `maskEmail(String)`: Retains the domain. Example: `jo******@example.com`

## 3. Graceful Fallbacks
- The utility methods are designed to be extremely defensive.
- If a string is `null` or shorter than the required bounds, it will return the original string safely rather than throwing a `StringIndexOutOfBoundsException` and crashing the view renderer.
