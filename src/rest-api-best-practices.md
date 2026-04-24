---
title: "REST API Best Practices for Java Developers"
description: "A practical guide to designing clean, consistent, and production-ready REST APIs with Spring Boot — covering naming conventions, status codes, error handling, and versioning."
date: 2026-04-16
tags: ["Java", "REST API", "Spring Boot"]
cover: "images/rest-api-cover.svg"
---

## Why API Design Matters

Your API is a contract. Once consumers depend on it, breaking changes are painful. Getting the design right from the start saves you from versioning headaches, confusing error messages, and frustrated clients.

At Amdocs, our billing microservices expose REST APIs consumed by dozens of internal teams. Here's what consistent, well-designed APIs look like in practice.

## URL Naming Conventions

Use nouns, not verbs. The HTTP method already describes the action:

```
# Good
GET    /api/v1/invoices
GET    /api/v1/invoices/{id}
POST   /api/v1/invoices
PUT    /api/v1/invoices/{id}
DELETE /api/v1/invoices/{id}

# Bad
GET    /api/v1/getInvoices
POST   /api/v1/createInvoice
DELETE /api/v1/deleteInvoice/{id}
```

Use plural nouns for collections and kebab-case for multi-word resources:

```
/api/v1/billing-accounts
/api/v1/billing-accounts/{id}/payment-history
```

## HTTP Status Codes

Return the right status code — don't return `200 OK` for everything:

| Scenario | Status Code |
|---|---|
| Resource fetched successfully | `200 OK` |
| Resource created | `201 Created` |
| No content to return | `204 No Content` |
| Validation failed | `400 Bad Request` |
| Unauthenticated | `401 Unauthorized` |
| Authenticated but forbidden | `403 Forbidden` |
| Resource not found | `404 Not Found` |
| Unexpected server error | `500 Internal Server Error` |

## Consistent Error Responses

Never return raw stack traces. Define a standard error response structure:

```java
public record ApiError(
    int status,
    String error,
    String message,
    Instant timestamp
) {}
```

Handle exceptions globally with `@RestControllerAdvice`:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404).body(new ApiError(
            404, "Not Found", ex.getMessage(), Instant.now()
        ));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.status(400).body(new ApiError(
            400, "Bad Request", message, Instant.now()
        ));
    }
}
```

## Request Validation

Use Bean Validation annotations to keep validation out of your service layer:

```java
public record CreateInvoiceRequest(
    @NotNull Long accountId,
    @Positive BigDecimal amount,
    @NotBlank String currency
) {}

@PostMapping
public ResponseEntity<Invoice> create(@Valid @RequestBody CreateInvoiceRequest request) {
    return ResponseEntity.status(201).body(invoiceService.create(request));
}
```

## API Versioning

Version your API from day one — it's much harder to add later:

```java
@RestController
@RequestMapping("/api/v1/invoices")
public class InvoiceV1Controller { ... }

@RestController
@RequestMapping("/api/v2/invoices")
public class InvoiceV2Controller { ... }
```

Keep old versions alive long enough for consumers to migrate. Deprecate with a response header:

```java
response.addHeader("Deprecation", "true");
response.addHeader("Sunset", "2026-12-31");
```

## Key Takeaways

1. **Nouns in URLs, verbs in HTTP methods** — keep URLs clean and predictable
2. **Always return a consistent error body** — never expose stack traces
3. **Validate at the controller layer** — use `@Valid` and Bean Validation
4. **Version from day one** — `/api/v1/` costs nothing upfront, saves pain later
5. **Document with OpenAPI** — add `springdoc-openapi` and get Swagger UI for free

> A good API is one your consumers never have to ask questions about.

REST API design is one of those skills that compounds over time. Every service you build teaches you something new about what your consumers actually need.
