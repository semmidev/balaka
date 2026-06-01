---
name: springboot-ci-cd-pipeline
description: Best practices for managing the GitHub Actions CI/CD pipeline, including static analysis (SAST), software composition analysis (SCA), and testing. Use when modifying or adding GitHub Action workflows.
---

# Spring Boot CI/CD Pipeline Skill

This project utilizes a highly advanced, multi-stage GitHub Actions workflow (`ci.yml`) to ensure strict code quality and security. Maintain these patterns when modifying the pipeline.

## 1. Fast Secret Scanning (Blocking)
- **Gitleaks & TruffleHog**: The very first job in the pipeline MUST be a secret scan. 
- If secrets are detected, the pipeline must immediately fail (`--exit-code 1`), blocking all subsequent build or test jobs. This prevents secrets from ever reaching deployment artifacts.

## 2. Parallel SAST (Static Application Security Testing)
Do not rely on a single SAST tool. The pipeline runs multiple analyzers in parallel (depending on the `secret-scan` job):
- **SpotBugs**: Runs with the `FindSecBugs` plugin via the Maven `spotbugs:check` goal.
- **CodeQL**: Uses GitHub's native `github/codeql-action` targeting Java with `security-extended` and `security-and-quality` queries.
- **Semgrep**: Runs via the `semgrep/semgrep` Docker container, utilizing multiple rule configurations (`p/java`, `p/security-audit`).

## 3. SCA (Software Composition Analysis)
- **OWASP Dependency-Check**: Runs in parallel using `dependency-check:check` via Maven to identify vulnerable third-party libraries (CVEs). Ensure the `NVD_API_KEY` is provided as a secret to prevent rate limiting.

## 4. Comprehensive Testing & Coverage
- **Playwright**: The UI tests require Playwright browsers. Ensure `mvn exec:java -Dexec.mainClass="com.microsoft.playwright.CLI"` is run before the verify phase.
- **Verification**: Run `./mvnw clean verify`. This executes unit, integration, and functional tests.
- **Jacoco Coverage**: 
  - After testing, parse the `jacoco.csv` file using a bash script to output a Markdown summary (`$GITHUB_STEP_SUMMARY`) displaying exact Line Coverage percentages in the GitHub Actions UI.
  - Upload the raw Jacoco XML reports to Codecov and post a PR comment via `madrapps/jacoco-report`.
