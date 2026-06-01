---
name: springboot-entity-design
description: Guidelines for designing JPA entities and database relationships in the Spring Boot project. Use when creating new database tables or modifying existing entity relationships.
---

# Spring Boot Entity Design Skill

When designing database entities for this Spring Boot project, strictly adhere to the following patterns.

## 1. Base Entity & Identifiers
All database entities MUST extend `BaseEntity`. This ensures consistent auditing and tracking across the system.
- **UUIDs**: Primary keys (`id`) are automatically managed as UUIDs in `BaseEntity`. Do not create secondary ID fields unless required by external integrations.
- **Auditing**: `createdAt`, `updatedAt`, `createdBy`, and `updatedBy` are provided.
- **Concurrency**: Optimistic locking is enabled via `@Version` on the `rowVersion` field.
- **Soft Deletes**: `BaseEntity` includes a `deletedAt` field and uses Hibernate `@FilterDef` for soft deletes. Do not use hard deletes unless explicitly building a purge routine.

## 2. Lombok Usage
- Use `@Getter`, `@Setter`, and `@NoArgsConstructor`.
- **CRITICAL**: NEVER use `@Data`, `@ToString`, or `@EqualsAndHashCode` on JPA entities. These can trigger massive recursive SQL queries when evaluating bidirectional relationships, leading to `StackOverflowError` or memory exhaustion.

## 3. Relationship Best Practices

### @ManyToOne
- **Lazy Fetching**: JPA defaults `*ToOne` relationships to `EAGER`. You MUST explicitly set them to `LAZY`.
  ```java
  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "id_project")
  private Project project;
  ```

### @OneToMany
- Use `fetch = FetchType.LAZY` (which is the default, but being explicit is good practice).
- Determine lifecycle dependencies for `cascade`:
  - If the child cannot exist without the parent (e.g., an Invoice Line on an Invoice), use `cascade = CascadeType.ALL, orphanRemoval = true`.
  - If the child has an independent lifecycle, avoid `cascade = CascadeType.ALL` and do not use `orphanRemoval`.
  ```java
  @OneToMany(mappedBy = "transaction", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
  private List<JournalEntry> journalEntries = new ArrayList<>();
  ```
- **Performance Warning**: Avoid placing `@OneToMany` on massive collections (e.g., millions of records). If pagination is required, query the child repository directly rather than traversing the parent's collection.

### @ManyToMany
- **Avoid @ManyToMany**: We generally avoid direct `@ManyToMany` annotations. 
- Instead, create an explicit join entity (mapping table) with two `@ManyToOne` fields. This is required because mapping tables in this system usually end up needing additional payload columns (like `createdAt` or specific configuration flags).

## 4. Serialization
- Always use `@JsonIgnore` on internal or bidirectional relationship fields to prevent infinite JSON recursion loops.
