# Global Exception Handling in Spring Boot

## The Problem It Solves

Without global handling, every controller method needs its own try/catch blocks, and unhandled exceptions bubble up to the servlet container, producing ugly default error pages or stack traces. `@ControllerAdvice` + `@ExceptionHandler` centralize this logic so exceptions thrown anywhere in your controllers are funneled into one place and converted into consistent, meaningful HTTP responses.

## How the Two Annotations Work Together

**`@ControllerAdvice`** — marks a class as a global "aspect" for controllers. Spring applies it to all `@Controller` / `@RestController` beans by default. It can also be scoped:

```java
@ControllerAdvice(basePackages = "com.example.orders")   // only this package
@ControllerAdvice(annotations = RestController.class)    // only @RestController beans
```

**`@ExceptionHandler`** — marks a method inside the advice class as the handler for one or more exception types. When a matching exception propagates out of a controller, Spring invokes this method instead of the default error handling.

## Complete Example

```java
// Custom exceptions
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String msg) { super(msg); }
}

public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String msg) { super(msg); }
}
```

```java
// A standard error body
public record ErrorResponse(
    Instant timestamp,
    int status,
    String error,
    String message,
    String path
) {}
```

```java
@RestControllerAdvice   // @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex,
                                        HttpServletRequest request) {
        return new ErrorResponse(
            Instant.now(), 404, "Not Found",
            ex.getMessage(), request.getRequestURI());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, String> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        return errors;
    }

    // Catch-all for anything unexpected
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneric(Exception ex, HttpServletRequest request) {
        return new ErrorResponse(
            Instant.now(), 500, "Internal Server Error",
            "Something went wrong", request.getRequestURI());
    }
}
```

Usage in a controller — no try/catch needed:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userRepo.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User " + id + " not found"));
    }
}
```

Requesting `/users/99` (missing) now returns `404` with a clean JSON body instead of a `500` stack trace.

## Key Mechanics

- **Resolution order**: Spring picks the *most specific* matching `@ExceptionHandler`. A handler for `ResourceNotFoundException` beats one for `RuntimeException`, which beats `Exception`.
- **Local wins over global**: an `@ExceptionHandler` defined inside the controller itself takes precedence over `@ControllerAdvice` handlers.
- **`@RestControllerAdvice`** = `@ControllerAdvice` + `@ResponseBody` — use it for REST APIs so handler return values serialize to JSON automatically. Plain `@ControllerAdvice` is for view-returning (MVC) controllers where you might return a `ModelAndView` or error page name.
- **Flexible return types**: handlers can return a POJO, `ResponseEntity<T>` (to control headers/status dynamically), `ProblemDetail` (RFC 7807, supported since Spring 6), or a view name.

```java
@ExceptionHandler(IllegalArgumentException.class)
public ResponseEntity<ErrorResponse> handleBadInput(IllegalArgumentException ex) {
    return ResponseEntity
        .status(HttpStatus.BAD_REQUEST)
        .header("X-Error-Code", "BAD_INPUT")
        .body(new ErrorResponse(...));
}
```

## Modern Alternative: `ResponseEntityExceptionHandler`

Since Spring 6, you can extend `ResponseEntityExceptionHandler` in your advice class to get built-in handlers for Spring MVC's own exceptions (`MethodArgumentNotValidException`, `HttpRequestMethodNotSupportedException`, etc.) that produce RFC 7807 `ProblemDetail` responses — you only override/customize what you need:

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers,
            HttpStatusCode status, WebRequest request) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(status, "Validation failed");
        // ... attach field errors as a property
        return ResponseEntity.of(pd).build();
    }
}
```

## Best Practices

1. **Never leak stack traces or internals** to clients — log them server-side, return a sanitized message.
2. **Always include a catch-all** `Exception` handler so nothing escapes as raw HTML/whitelabel error.
3. **Map business exceptions to specific HTTP codes** (404 not found, 409 conflict, 422 business rule) rather than dumping everything as 500 or 400.
4. **Log with appropriate levels**: warn/debug for 4xx (client fault), error for 5xx (your fault).
5. Keep the error body **consistent** — a single `ErrorResponse` schema across the whole API makes client error handling trivial.
