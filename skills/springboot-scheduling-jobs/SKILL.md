---
name: springboot-scheduling-jobs
description: Best practices for implementing background tasks and scheduled jobs using Spring's @Scheduled annotation. Use when creating recurring background processes.
---

# Spring Boot Background Jobs & Scheduling Skill

When implementing scheduled jobs in this project, reliability is the primary concern. Follow these guidelines strictly to ensure background tasks do not fail silently.

## 1. Class Structure
- Annotate the scheduler class with `@Component` and `@Slf4j`.
- Inject dependencies using Lombok's `@RequiredArgsConstructor`.
- **Delegation**: The Scheduler class must ONLY handle the trigger and the logging. It MUST delegate the actual business logic execution to a `@Service` class.

## 2. Configuration & Triggers
- Use `@Scheduled` on the triggering method.
- **NEVER hardcode CRON expressions**. Always inject them via properties with a safe default fallback. This allows environments (like Dev/Staging) to alter or disable schedules easily.
  ```java
  @Scheduled(cron = "${app.recurring.schedule:0 0 5 * * *}")
  public void processRecurringTransactions() { ... }
  ```

## 3. Handling Failures
- The `@Scheduled` method MUST wrap its entire body in a broad `try-catch` block.
- Uncaught exceptions will silently terminate the thread and may break the executor pool, preventing future schedules from running.

## 4. Logging Standards
- Always log at the beginning (`log.info("Starting scheduled...")`) and at the end of the method.
- Log the number of items processed to provide visibility.
- Log any exceptions explicitly in the catch block (`log.error("Scheduled processing failed", e)`).

### Full Example
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class RecurringTransactionScheduler {

    private final RecurringTransactionService recurringTransactionService;

    @Scheduled(cron = "${app.recurring.schedule:0 0 5 * * *}")
    public void processRecurringTransactions() {
        log.info("Starting scheduled recurring transaction processing");
        try {
            int processed = recurringTransactionService.processAllDue();
            log.info("Recurring transaction processing completed: {} transactions processed", processed);
        } catch (Exception e) {
            log.error("Scheduled recurring transaction processing failed", e);
        }
    }
}
```
