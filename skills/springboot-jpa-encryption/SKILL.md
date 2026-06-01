---
name: springboot-jpa-encryption
description: Guidelines for implementing transparent JPA attribute encryption using AES-256-GCM. Use when adding new sensitive PII fields (like NIK or NPWP) to entities.
---

# Spring Boot JPA Attribute Encryption Skill

This project handles sensitive PII (Personally Identifiable Information) such as NIKs and NPWPs. To comply with data privacy laws, these fields must be encrypted at rest in the database.

## 1. The `EncryptedStringConverter`
- We use a custom `AttributeConverter<String, String>` named `EncryptedStringConverter`.
- It implements **AES-256-GCM** authenticated encryption. This guarantees both confidentiality and integrity (data cannot be tampered with in the database without throwing decryption errors).
- It generates a random 12-byte IV for every single row, prepending it to the ciphertext, making identical plaintext values appear different in the database.

## 2. Applying Encryption to Entities
- Whenever you add a highly sensitive String field to a `@Entity`, you MUST apply the converter.
- Use the `@Convert` annotation from `jakarta.persistence`.

```java
@Entity
public class Client extends BaseEntity {
    
    @Column(length = 20)
    private String npwp;

    @Convert(converter = EncryptedStringConverter.class)
    @Column(length = 255) // MUST be large enough to hold Base64 IV + Ciphertext
    private String nik;
}
```

## 3. Database Column Sizing
- **CRITICAL**: Because the converter encodes the encrypted byte array (IV + Ciphertext + Auth Tag) into a Base64 string and prepends the `ENC:` prefix, the resulting string is significantly larger than the plaintext.
- Never set a strict `@Column(length = 16)` for an encrypted field. It must be at least 255 characters or `TEXT`.

## 4. Key Management
- DO NOT hardcode the encryption key. Inject it via `@Value("${app.encryption.key}")`.
- If the key is missing, fail fast or securely degrade (as seen below).

## Real-world Examples from Codebase

### `EncryptedStringConverter.java`
An AES-256-GCM converter providing confidentiality and integrity for PII fields. Note the fallback behavior when the key is missing.

```java
@Converter
@Component
@Slf4j
public class EncryptedStringConverter implements AttributeConverter<String, String> {

    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 128;
    private static final String PREFIX = "ENC:";
    private static final SecureRandom SECURE_RANDOM = new SecureRandom();

    private SecretKey secretKey;
    private boolean encryptionEnabled = false;

    @Value("${app.encryption.key:}")
    public void setEncryptionKey(String keyBase64) {
        if (keyBase64 == null || keyBase64.isBlank()) {
            log.warn("Encryption key not configured - PII fields will NOT be encrypted. " +
                     "Set app.encryption.key property or APP_ENCRYPTION_KEY env var for production.");
            encryptionEnabled = false;
            return;
        }

        try {
            byte[] decodedKey = Base64.getDecoder().decode(keyBase64);
            if (decodedKey.length != 32) {
                throw new IllegalArgumentException("Encryption key must be exactly 32 bytes (256 bits)");
            }
            this.secretKey = new SecretKeySpec(decodedKey, "AES");
            this.encryptionEnabled = true;
            log.info("AES-256-GCM database encryption enabled");
        } catch (Exception e) {
            log.error("Failed to initialize encryption key", e);
            throw new RuntimeException("Failed to initialize encryption key", e);
        }
    }
}
```

## 5. Graceful Degradation
- The converter detects the `app.encryption.key` property. If the key is missing (e.g., in local development without strict env vars), encryption is bypassed.
- It detects encrypted data via the `ENC:` prefix. If legacy unencrypted data exists, it will return the plaintext safely without crashing.
