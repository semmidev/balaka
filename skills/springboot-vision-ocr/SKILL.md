---
name: springboot-vision-ocr
description: Pattern for integrating Google Cloud Vision API for OCR and receipt scanning. Use when working with image processing or Cloud Vision configurations.
---

# Spring Boot Google Cloud Vision OCR Skill

This application integrates with Google Cloud Vision (`ImageAnnotatorClient`) to perform Optical Character Recognition (OCR) on uploaded receipts and invoices.

## 1. Secure Credential Loading
- Google Cloud credentials (JSON) are loaded dynamically based on the `google.cloud.vision.credentialsPath` property.
- To prevent **Path Traversal Attacks** (where a malicious admin might configure the path to read `/etc/shadow`), the `GoogleCloudVisionConfig` strictly validates the path:
  ```java
  Path normalizedPath = Paths.get(credentialsPath).normalize().toAbsolutePath();
  if (!Files.isRegularFile(normalizedPath) || !Files.isReadable(normalizedPath)) {
      throw new IOException("...");
  }
  ```
- This method is intentionally annotated with `@SuppressFBWarnings("PATH_TRAVERSAL_IN")` because the validation is manual but exhaustive.

## 2. Conditional Instantiation
- The Google Cloud Vision beans should not prevent the application from starting if OCR is disabled (e.g., in local development).
- Always use `@ConditionalOnProperty(name = "google.cloud.vision.enabled", havingValue = "true")` on the configuration beans.
- Services that depend on the OCR client (e.g., `VisionOcrService`) should handle a `null` client gracefully, skipping the OCR logic or throwing a standard "Feature Disabled" exception.

## 3. Batch Processing
- When sending images to the API, use `batchAnnotateImages` to optimize network calls.
- Wrap the remote API call in a `try-catch` block to handle `StatusRuntimeException` from gRPC, ensuring network timeouts do not crash the calling thread.

## 4. Conditional Beans
- Wrap the OCR integration behind a `@ConditionalOnProperty` so the application can still boot if GCP credentials aren't provided.

## Real-world Examples from Codebase

### `VisionOcrService.java`
Uses `google-cloud-vision` to extract text and fails safely if the integration is disabled.

```java
@Service
@ConditionalOnProperty(name = "google.cloud.vision.enabled", havingValue = "true")
public class VisionOcrService {

    private static final Logger log = LoggerFactory.getLogger(VisionOcrService.class);

    private final ImageAnnotatorClient imageAnnotatorClient;
    private final GoogleCloudVisionConfig config;

    public VisionOcrService(ImageAnnotatorClient imageAnnotatorClient, GoogleCloudVisionConfig config) {
        this.imageAnnotatorClient = imageAnnotatorClient;
        this.config = config;
    }

    public OcrResult extractText(byte[] imageBytes) {
        if (!config.isEnabled() || imageAnnotatorClient == null) {
            return OcrResult.error("Google Cloud Vision is not enabled");
        }

        try {
            ByteString imgBytes = ByteString.copyFrom(imageBytes);
            Image image = Image.newBuilder().setContent(imgBytes).build();
            Feature feature = Feature.newBuilder().setType(Feature.Type.DOCUMENT_TEXT_DETECTION).build();
            AnnotateImageRequest request = AnnotateImageRequest.newBuilder().addFeatures(feature).setImage(image).build();

            BatchAnnotateImagesResponse response = imageAnnotatorClient.batchAnnotateImages(List.of(request));
            // ... (Process response)
            return OcrResult.success(extractedText);
        } catch (Exception e) {
            return OcrResult.error("OCR processing failed: " + e.getMessage());
        }
    }
}
```
