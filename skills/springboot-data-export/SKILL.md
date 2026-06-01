---
name: springboot-data-export
description: Guidelines for generating massive data exports, handling ZipOutputStream, and streaming CSV/PDF reports safely. Use when creating reporting or data export services.
---

# Spring Boot Data Export & Reporting Skill

When dealing with large volumes of data (like financial reports or compliance exports), you must manage memory and transactional boundaries carefully.

## 1. Bulk Export with ZipOutputStream
- When exporting an entire database or multiple large CSV files, never store all strings in memory.
- Use `java.util.zip.ZipOutputStream` wrapping a `ByteArrayOutputStream` to stream files directly into an archive.
  ```java
  ByteArrayOutputStream baos = new ByteArrayOutputStream();
  try (ZipOutputStream zos = new ZipOutputStream(baos)) {
      addTextEntry(zos, "01_config.csv", exportConfig());
  }
  return baos.toByteArray();
  ```
- **Manifests**: Always include a `MANIFEST.md` file in the archive detailing the export date, formatting versions, and record counts.
- **Prefixing**: Prefix files with numbers (e.g., `01_clients.csv`, `02_projects.csv`) to enforce import order and maintain foreign key dependencies.

## 2. CSV Escaping
- Do not blindly concatenate strings with commas. Data can contain commas or quotes.
- You must manually escape CSV strings (or use Apache Commons CSV). If escaping manually:
  - If a value contains a comma or quote, wrap it in double quotes.
  - Double up any internal double quotes.

## 3. Read-Only Transactions
- Export methods often need to traverse deep lazy-loaded object graphs.
- Annotate the service method with `@Transactional(readOnly = true)` to keep the Hibernate session open and prevent `LazyInitializationException` without incurring write-locking overhead.

## Real-world Examples from Codebase

### `DataExportService.java`
This shows how the application builds a ZIP archive natively in Java using `ZipOutputStream` and returns it as a `byte[]` for the controller to stream to the user.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Slf4j
public class DataExportService {

    private final ChartOfAccountRepository accountRepository;
    // ... other repositories ...

    /**
     * Export all company data to a ZIP archive.
     */
    public byte[] exportAllData() throws IOException {
        log.info("Starting full data export");

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        try (ZipOutputStream zos = new ZipOutputStream(baos)) {
            // Export metadata
            addManifest(zos);

            // Export in dependency order (numbered for import sequence)
            addTextEntry(zos, "01_company_config.csv", exportCompanyConfig());
            addTextEntry(zos, "02_chart_of_accounts.csv", exportChartOfAccounts());
            
            // ... lots more CSVs added to ZIP ...
        }
        
        return baos.toByteArray();
    }
    
    private void addTextEntry(ZipOutputStream zos, String filename, String content) throws IOException {
        ZipEntry entry = new ZipEntry(filename);
        zos.putNextEntry(entry);
        zos.write(content.getBytes(StandardCharsets.UTF_8));
        zos.closeEntry();
    }
}
```

## 4. Specific Formats
- **PDF**: Use the `openpdf` library.
- **Excel**: Use `Apache POI` (`poi-ooxml`).
- Always throw custom exceptions like `DataExportException` or `ReportGenerationException` when streams fail, rather than returning generic IOExceptions to the controller layer.
