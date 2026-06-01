---
name: springboot-frontend-tailwind
description: Pattern for integrating Tailwind CSS v4 directly into a Spring Boot Maven build without requiring a separate Node.js server. Use when modifying CSS or UI styles.
---

# Spring Boot Frontend Tailwind Skill

This project uses Tailwind CSS v4 for styling Thymeleaf templates. However, it does **not** use a separate standalone Node.js server (like Next.js or Vite). Instead, the frontend build is orchestrated entirely by Maven.

## 1. Maven Frontend Plugin
- The `frontend-maven-plugin` is configured in `pom.xml`.
- During the `generate-resources` phase of the Maven lifecycle, it automatically:
  1. Downloads Node.js and NPM locally.
  2. Runs `npm install` in the `src/main/frontend` directory.
  3. Runs `npm run build`.

## 2. Tailwind Compilation
- The `package.json` in `src/main/frontend` defines the build script:
  `"build": "tailwindcss -i ./src/app.css -o ../resources/static/css/app.css --minify"`
- This takes the source CSS (`src/main/frontend/src/app.css`) and compiles it directly into the Spring Boot static resources folder (`src/main/resources/static/css/app.css`), minifying it for production.

## 3. Development Workflow
- **Do not modify `src/main/resources/static/css/app.css` directly.** Your changes will be overwritten on the next Maven build.
- **Always edit `src/main/frontend/src/app.css`** or the Thymeleaf templates themselves.
- To enable hot-reloading of CSS during local development without running a full Maven build, open a terminal in `src/main/frontend` and run:
  ```bash
  npm run watch
  ```
