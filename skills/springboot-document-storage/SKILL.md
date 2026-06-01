---
name: springboot-document-storage
description: Security guidelines and best practices for implementing document storage, file uploads, and transparent file encryption in Spring Boot. Use when writing services that handle files.
---

# Spring Boot Document Storage & Security Skill

Handling file uploads correctly is critical to application security. This project implements strict controls in `DocumentStorageService.java`. When managing files, you must follow these patterns:

## 1. Path Traversal Prevention
- Define storage paths centrally in `application.properties`.
- Use `Paths.get(storagePath).toAbsolutePath().normalize()` to get the true root location.
- **Mandatory Check**: Before reading, writing, or deleting *any* file, you MUST verify the target path stays within the root directory using `.startsWith()`.
  ```java
  Path targetPath = rootLocation.resolve(storedFilename).normalize();
  if (!targetPath.startsWith(rootLocation)) {
      throw new SecurityException("Access denied: path traversal attempt detected");
  }
  ```

## 2. Magic Byte Validation
- Do not trust the `ContentType` header or the file extension provided by the client.
- You must use a service (e.g., `FileValidationService.validateMagicBytes()`) to read the raw file headers (magic bytes) to guarantee the file content matches its declared type, preventing content-type spoofing.

## 3. Transparent Encryption
- Files should be encrypted at rest.
- Utilize a `FileEncryptionService` to intercept the byte stream:
  - Encrypt: `byte[] contentToStore = fileEncryptionService.encrypt(fileContent);`
  - Decrypt: `byte[] decryptedContent = fileEncryptionService.decrypt(encryptedContent);`
- Handle decryption dynamically when loading files into a `ByteArrayResource`.

## 4. Log Injection Prevention
- Never log raw, unsanitized filenames provided by the user. Malicious users can inject newline characters to forge log entries.
- Always wrap filenames in a sanitization utility:
  ```java
  log.debug("Stored file: {}", LogSanitizer.filename(originalFilename));
  ```

## 5. Storage Organization
- Do not store all files in a single flat directory.
- Dynamically generate subdirectories chronologically (e.g., `YYYY/MM`) and use UUIDs for the final filename.
  ```java
  LocalDate today = LocalDate.now();
  String subPath = String.format("%d/%02d", today.getYear(), today.getMonthValue());
  ```
