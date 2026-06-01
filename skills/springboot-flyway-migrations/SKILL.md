---
name: springboot-flyway-migrations
description: Conventions for structuring, naming, and writing idempotent Flyway SQL database migrations. Use when altering database schemas.
---

# Spring Boot Flyway Migrations Skill

This project uses Flyway for strict, version-controlled database schema evolution. All migrations live in `src/main/resources/db/migration`.

## 1. Naming Convention
- All migration files MUST follow the strict Flyway naming standard: `V{version}__{description}.sql`.
- **Note the double underscore (`__`)**. If you use a single underscore, Flyway will ignore or fail to parse the file.
- Example: `V001__security.sql`, `V002__core_schema.sql`.
- Pad version numbers with zeros to ensure they sort correctly in the file system (e.g., `V015` instead of `V15`).

## 2. Logical Grouping
- Do not create a single monolithic SQL file.
- Separate schemas into logical bounds. For example:
  - `V001__security.sql` (Users, Roles)
  - `V002__core_schema.sql` (Tenants, Config)
  - `V003__feature_schema.sql` (Transactions, Projects)
  - `V004__seed_data.sql` (Initial required data)

## 3. Writing Migrations
- Write pure, dialect-specific SQL (this project uses PostgreSQL).
- **No altering past migrations**: Once a migration has been committed and merged into the `main` branch, you MUST NOT edit it. Flyway checksums will fail. If you need to change a table, create a new `VXXX__alter_table.sql` file.

## 3. Constraints and Indexes
- Always define a `PRIMARY KEY` (usually `UUID`).
- Name indices clearly (`idx_users_username`).
- Use `ON DELETE CASCADE` appropriately for relational cleanup.

## 4. Seed Data
- Put foundational system data (e.g., default Admin users, mandatory system settings) in a dedicated seed migration.
- Use `INSERT INTO ... ON CONFLICT DO NOTHING` (or PostgreSQL equivalent) if you are worried about idempotency, though Flyway guarantees execution only happens once per version.

## Real-world Examples from Codebase

### `V001__security.sql`
Here is how the initial users and roles schema is defined:

```sql
-- V001: Security Schema + Seed Data
-- Tables: users, user_roles, audit_logs

CREATE TABLE users (
    id UUID PRIMARY KEY,
    row_version BIGINT NOT NULL DEFAULT 0,
    username VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by VARCHAR(100),
    updated_by VARCHAR(100),
    deleted_at TIMESTAMP
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_active ON users(active);

CREATE TABLE user_roles (
    id UUID PRIMARY KEY,
    id_user UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by VARCHAR(100),
    UNIQUE(id_user, role)
);

CREATE INDEX idx_user_roles_user ON user_roles(id_user);
CREATE INDEX idx_user_roles_role ON user_roles(role);
```
