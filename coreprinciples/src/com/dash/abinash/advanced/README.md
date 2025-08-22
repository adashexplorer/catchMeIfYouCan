#
## Java Exception Handling Patterns

### Exception Wrapping / Translation
- Catch a checked exception and rethrow it is an un checked exception.
- Preserves cause with `new RuntimeException("message", cause);`
- Use case : API design, frameworks, layers where you don't want checked exceptions exposed.
### Catch and Rethrow with more context
- Add more `domain-specific information` before throwing.
```shell
try {
    userRepository.save(user);
} catch (SQLException ex) {
    throw new DataAccessException("Failed to save user: " + user.getId(), ex);
}
```
- Benefit: Callers see a meaningful error (`DataAccessException`) instead of a low-level one.
- Used heavily in Spring (E.g. `JDBC Exceptions` wrapped into `DataAccessException`) 

### Centralized Exception Handling
- One place to handle error instead of scattered `try-catch`.
- Example :
    - Sprint Boot -> `@ControllerAdvice` + `@ExceptionHandler`
    - JAVA EE -> Exception Mappers
```shell
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleAll(Exception ex) {
        return ResponseEntity.status(500).body("Something went wrong!");
    }
}
```
### Graceful Recovery / Fallback
- Instead of catching, try an alternative path.
```shell
try {
    return remoteService.call();
} catch (IOException e) {
    return localCache.getFallbackData();
}
```
- Often combined with `Retry patterns` and `Circuit Breakers` (In Distributed Systems)
### Try-with-resources (Automatic Cleanup)
- Introduced in `Java 7`
- Ensures resources (files, sockets, DB Connections) are closed even when exceptions happen.
```shell
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    return br.readLine();
}
```
- Benefit: No resource leaks, cleaner code
### Exception suppression / Logging only
- Sometimes you don't throw, but just log and continue.
```shell
try {
    auditService.logAction(user);
} catch (AuditException ex) {
    logger.warn("Audit failed: " + ex.getMessage());
    // continue without stopping main flow
}
```
- Use-case : Non-critical failures where business flow must continue.
### Custom application Exceptions
- Define your own exception hierarchy (Checked or Unchecked) for domain logic.
```shell
public class InvalidOrderException extends RuntimeException {
    public InvalidOrderException(String message) { super(message); }
}
```
- Helps distinguish between business vs system errors
### Wrapping multiple exceptions (Suppressed Exceptions)
- With `try-with-recources`, multiple exceptions may occur
- You can capture suppressed ones
```shell
try (MyResource r = new MyResource()) {
    r.use();
} catch (Exception e) {
    for (Throwable sup : e.getSuppressed()) {
        logger.error("Suppressed: " + sup);
    }
}
```
### Fail-fast vs Fail-safe
- Fail-fast: Throw immediately when something is wrong (NULL, invalid state)
- Fail-safe: Handle errors internally and continue gracefully.
- Choice depends on domain/business critically.
### Exception Shielding (For APIs & Services)
- never expose `internal exceptions` to API consumers.
- Convert internal stack traces into `safe error messages`.
```shell
catch (SELException e) {
  throw new ApiException("Database temporarily unavailable. Please retry later.")
}
```
### Retry Pattern
- Retry an operation a certain number of time before calling
- Common in network/database calls.
```shell
for (int i = 0; i < 3; i++) {
    try {
        return apiClient.call();
    } catch (IOException e) {
        logger.warn("Retrying... attempt " + (i + 1));
        Thread.sleep(1000);
    }
}
throw new RuntimeException("API call failed after retries");
```
- Often abstracted via libraries like `Spring Retry` or `Resilience4j`
### Circuit Breaker Pattern
- Protects system from cascading failures when a downstream service keeps failing.
- Temporarily `Opens the circuit` and fails fast instead of trying again.
- Example :
```shell
CircuitBreaker cb = new CircuitBreaker.ofDefaults("backendService");
Supplier<String> supplier = CircuitBreaker.decorateSupplier(cb, () -> backend.call());
```
### Bulkhead Pattern
- Isolate failures in one component from affecting the entire system
- Example - Run risky tasks in a separate thread pool with bounded size.
- If that pool is exhausted, main system still works.
### Compensating Transaction Pattern
- Instead of rolling back (hard in distributed systems), you issue a `compensating action` when something fails.
```shell
try {
    paymentService.charge(card);
    orderService.confirm(order);
} catch (Exception ex) {
    paymentService.refund(card);
    throw new OrderFailedException("Order creation failed", ex);
}
- Common in `Saga Pattern` for Micro services.
```
### Notification or Observer Pattern for Exceptions
- Instead of throwing, `publish exception events` to observers / listeners.
- Example : Logging Framework, Monitoring Systems (Sentry, ELK)
### Fail-over Pattern
- If one service fails, switch to a `back-up/fallback` service.
```shell
try {
    return primaryService.call();
} catch (Exception e) {
    return backupService.call();
}
```
- Often combined with `Retry + Circuit Breaker`
### Idempotent Exception Handling
- Design operations so that if an exception happens midway, retrying the same request won't cause `duplicate efforts`.
- Example - Payment Systems (Charge should not happen twice)
### Poison Message Pattern (In messaging system)
- If a message repeatedly causes exceptions, move it to a dead-letter queue instead of retrying endlessly.
### Escalation Pattern
- If the current layer cannot decide how to handle an exception, `bubble it up` to a higher level (UI / API Gateway or orchestrator)
Example -
```shell
catch (SQLException ex) {
   throw new ServiceUnavailableException("DB down, escalate to admin", ex);
}
```
### Graceful Degradation Pattern
- Instead of crashing, return `partial results` or a `friendly block`
```shell
try {
    return recommendationService.get();
} catch (Exception e) {
    return Collections.emptyList(); // show no recommendations instead of error
}
```
### Policy-Based exception handling
- Define exception handling policies outside of business code (Like configs or annotations)
- Example - Spring's `@Retryable`, Quarkus `@Fallback`
- Advantages - Consistent handling without scattering `try-catch` everywhere
### Contextual Exception Enrichment
- Add more metadata/context before rethrowing, so debugging is easier.
```shell
catch (SQLException e) {
    throw new DataAccessException("Failed query: " + query, e);
}
```
- Makes logs much more useful in production.
### Localization / User-friendly Wrapping
- Technical exceptions -> converted into business/user-friendly messages at boundary layers (API / UI)
```shell
catch (IOException e) {
    throw new UserFacingException("Unable to fetch your file, please try again.");
}
```
### Sentinel / Special Value Pattern
- Instead of throwing exceptions for `expected` problems, return a sentinel object.
```shell
Optional<User> user = repo.findById(id); // instead of throwing NoSuchElementException
```
- Reduces noisy `try-catch`
### Exception Shielding (Security Pattern)
- Prevent leaking sensitive info (like stack traces, SQL queries) to clients
- Always replace low-level exceptions with generic safe messages before sending outside.
### Throttling Exceptions Pattern
- Avoid Log spam when an error happens repeatedly.
- E.g. - Only log first N times or log `x100 times in last 5 minutes`
- Helps in production monitoring.
### Audit / Forensics Exception Pattern
- Every exception is captured, enriched, and persisted for later auditing (especially in financial/legal systems).
- usually done by logging frameworks integrated with a DB or monitoring tool.
### Delayed Exception Aggregation
- Instead of failing fast on first exception, collect all errors and throw one aggregated exception
```shell
List<Exception> errors = new ArrayList<>();
for (Task t : tasks) {
    try { t.run(); } catch (Exception e) { errors.add(e); }
}
if (!errors.isEmpty()) throw new BatchProcessingException(errors);
```
- Useful in `validation frameworks` (Hibernate Validator, Bean validation)
### Escaped Safe Mode
- When exception happens, system goes into `safe/restricted mode` instead of full crash.
- Example - IDEs disabling some features if plugin fails, keep editor running
### Compartmentalized Exceptions Domains
- Large apps separate exception classes into `domains` (e.g., `DataAccessException`, `BusinessException`, `SecurityException`) and handle them differently at global boundaries.


---

## 📌 Core Patterns

| Pattern | Description / Use Case | Example |
|---------|-------------------------|---------|
| Try-with-resources | Automatic resource cleanup (`Java 7+`) | `try (BufferedReader br = ...) { ... }` |
| Exception Wrapping / Translation | Convert checked → unchecked, preserve cause | `throw new RuntimeException("msg", e)` |
| Catch & Rethrow with Context | Add domain-specific context | `throw new DataAccessException("User " + id, e)` |
| Custom Exceptions | Define domain hierarchy | `class InvalidOrderException extends RuntimeException {}` |
| Suppressed Exceptions | Capture multiple exceptions from try-with-resources | `e.getSuppressed()` |
| Fail-fast vs Fail-safe | Decide between immediate failure vs safe continuation | Fail-fast on invalid state, fail-safe on optional features |
| Logging-only Exceptions | Log and continue for non-critical failures | `logger.warn("Audit failed", e)` |
| Escalation | Bubble exception up to higher layer | `throw new ServiceUnavailableException("DB down", e)` |

---

## 🛡 Resilience Patterns

| Pattern | Description / Use Case | Example |
|---------|-------------------------|---------|
| Retry | Retry N times before failing | Loop with retries / Spring Retry |
| Circuit Breaker | Stop calls to failing services | Resilience4j `CircuitBreaker.decorateSupplier(...)` |
| Fallback | Alternative path on failure | Return from cache if remote fails |
| Graceful Degradation | Partial results instead of failure | Return empty recommendations |
| Fail-over | Switch to backup service | Try primary, else call backup |
| Bulkhead | Isolate risky tasks (separate pool) | ThreadPool isolation |
| Compensating Transaction | Undo via compensating action | `paymentService.refund()` after failure |
| Idempotent Handling | Avoid duplicates on retry | Safe payment retries |
| Throttling Exceptions | Prevent log spam | Log once per interval |
| Escaped Safe Mode | Run in limited mode on failure | IDE disables plugin but runs editor |

---

## 🌐 Distributed / API Patterns

| Pattern | Description / Use Case | Example |
|---------|-------------------------|---------|
| Centralized Exception Handling | Single point of handling | Spring `@ControllerAdvice` |
| Standard Error Response | Consistent API errors | `{ code, message, traceId }` |
| Exception Shielding | Hide sensitive info in APIs | Return safe messages, not stack traces |
| Exception Translation | Map vendor exceptions to generic | SQL → Spring `DataAccessException` |
| Remote Boundary Wrapping | Convert remote checked → local unchecked | Wrap RMI/HTTP exceptions |
| Poison Message | Send unprocessable messages to DLQ | Messaging dead-letter queue |
| Audit / Forensics | Persist errors for compliance | Banking/finance logging |

---

## 🚀 Advanced / Project Patterns

| Pattern | Description / Use Case | Example |
|---------|-------------------------|---------|
| Policy-based Handling | Externalized handling via annotations/config | Spring `@Retryable`, `@Recover` |
| Contextual Enrichment | Add metadata before rethrowing | `throw new DAOException("Query: " + q, e)` |
| Localization Wrapping | User-friendly error conversion | `throw new UserFacingException("File missing")` |
| Sentinel / Special Value | Use Optional/special value instead of exception | `Optional<User>` instead of NoSuchElementException |
| Delayed Aggregation | Collect multiple errors, throw together | Batch/validation frameworks |
| Notification/Observer | Publish exception events | Monitoring tools, ELK, Sentry |
| Compartmentalized Domains | Different exception hierarchies per layer | `BusinessException`, `SecurityException` |

---