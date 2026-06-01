---
name: springboot-repository-queries
description: Best practices for implementing Spring Data JPA repositories, writing JPQL, and configuring native PostgreSQL queries. Use when building data access layers or optimizing queries.
---

# Spring Boot Repository & Query Design Skill

Follow these guidelines when implementing data access in the Spring Boot project.

## 1. Repository Setup
- Extend `JpaRepository<Entity, UUID>`.
- Keep repositories in the `com.artivisi.accountingfinance.repository` package.

## 2. JPQL Queries & Mitigating N+1
- Avoid relying solely on basic Spring Data method names (like `findAll`) if they trigger N+1 queries due to uninitialized lazy proxies.
- Use `@Query` with `LEFT JOIN FETCH` to eagerly load required relationships in a single SQL round-trip.
  ```java
  @Query("SELECT t FROM Transaction t LEFT JOIN FETCH t.journalEntries WHERE t.id = :id")
  Optional<Transaction> findByIdWithJournalEntries(@Param("id") UUID id);
  ```
- Use `DISTINCT` when fetching collections to avoid duplicate root entities in the result set.

## 3. Native Queries & PostgreSQL
When dynamic filtering or complex joins exceed JPQL's capabilities, native queries are required.
- Set `nativeQuery = true` on `@Query`.
- **Type Casting**: PostgreSQL requires explicit type casting when checking parameters for nulls or evaluating complex where clauses. ALWAYS cast your parameters:
  ```sql
  WHERE (CAST(:status AS VARCHAR) IS NULL OR t.status = CAST(:status AS VARCHAR))
  AND (CAST(:projectId AS UUID) IS NULL OR t.id_project = CAST(:projectId AS UUID))
  ```

## 4. Projections
- Use Interface-based Projections (`interface UserSummary { String getUsername(); }`) to select only required columns when full entity graphs are too heavy.

## 5. Pagination
- When using pagination (`Pageable`) with JPQL, Spring Data can often auto-generate the count query.
- When using native queries with pagination, you MUST provide a `countQuery`.
  ```java
  @Query(value = "SELECT t.* FROM transactions t WHERE ...",
         countQuery = "SELECT COUNT(*) FROM transactions t WHERE ...",
         nativeQuery = true)
  Page<Transaction> findByFilters(..., Pageable pageable);
  ```

## 5. Modifying Queries
- For bulk updates or deletes, use `@Modifying` in combination with `@Query`.
- Often, for highly specific database cleanups (like purging voided transactions), using `entityManager.createNativeQuery(...).executeUpdate()` in the Service layer is preferred to bypass JPA cache issues and give precise control over query execution order.
