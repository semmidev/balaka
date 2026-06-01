---
name: springboot-devsecops-pipeline
description: Conventions for maintaining the DevSecOps pipeline, including SAST, SCA, SBOM generation, and secret scanning. Use when updating CI/CD or Maven plugins.
---

# Spring Boot DevSecOps Pipeline Skill

This project implements a rigorous "Shift-Left" DevSecOps pipeline natively through Maven and GitHub Actions (`.github/workflows/ci.yml`).

## 1. Secret Scanning (Fail-Fast)
- The CI pipeline runs `gitleaks` and `Trufflehog` as the very first job (`secret-scan`).
- If hardcoded credentials or API keys are detected, the build fails immediately, blocking subsequent jobs.
- Never commit secrets. Always use environment variables or encrypted secrets managers.

## 2. Static Application Security Testing (SAST)
- The project uses `spotbugs-maven-plugin` paired with the `findsecbugs-plugin`.
- It analyzes the Java bytecode for common vulnerabilities (e.g., SQL Injection, Path Traversal, XSS).
- If a vulnerability is found, the build fails.
- To suppress false positives, use the `@SuppressFBWarnings` annotation from `edu.umd.cs.findbugs.annotations`, providing a detailed `justification`.

## 3. Software Composition Analysis (SCA)
- The project uses `dependency-check-maven` (OWASP Dependency-Check).
- It cross-references all dependencies in `pom.xml` against the National Vulnerability Database (NVD).
- It is configured to fail the build if a vulnerability with a CVSS score of 7 (High/Critical) or greater is found.
- False positives must be documented in `dependency-check-suppressions.xml`.

## 4. Software Bill of Materials (SBOM)
- The project generates a standard SBOM using `cyclonedx-maven-plugin` during the `package` phase.
- This creates an `sbom.json` or `sbom.xml` file containing a cryptographic manifest of every dependency used in the application, vital for supply chain security audits.

## 4. DAST (Dynamic Application Security Testing)
- Use OWASP ZAP in a separate CI stage for deep scanning of the running container.

## Real-world Examples from Codebase

### `.github/workflows/ci.yml` (Secret Scanning)
Fast-failing jobs placed at the very top of the CI file to block PRs immediately if secrets (Gitleaks) or verifiable leaked credentials (TruffleHog) are found.

```yaml
jobs:
  # Fast secret scanning - runs first, blocks everything if secrets found
  secret-scan:
    name: Secret Detection
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5
        with:
          fetch-depth: 0

      - name: Install Gitleaks
        run: |
          curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.21.2/gitleaks_8.21.2_linux_x64.tar.gz | tar xz
          sudo mv gitleaks /usr/local/bin/

      - name: Run Gitleaks Scan
        run: gitleaks detect --source . --verbose --redact --exit-code 1

      - name: TruffleHog Scan
        uses: trufflesecurity/trufflehog@main
        with:
          extra_args: --only-verified
```
