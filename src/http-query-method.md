---
title: "HTTP QUERY Method — The Missing Piece in REST API Design"
description: "A deep dive into the proposed HTTP QUERY method — what it is, why it was needed, how it compares to GET and POST, and how to use it in your REST APIs with Spring Boot."
date: 2026-04-20
tags: ["HTTP", "REST API", "Spring Boot", "Java"]
cover: "images/http-query-method-cover.svg"
---

![HTTP QUERY Method](images/http-query-method-cover.svg)

## The Problem with Existing HTTP Methods

REST APIs have a fundamental tension when it comes to complex queries. You need to fetch data (read-only), but your query is too complex to fit in a URL.

Consider searching for users with multiple filters:

```
# Works for simple cases
GET /api/v1/users?age=25&city=NewYork&status=active

# Breaks down for complex queries — URL too long, hard to read
GET /api/v1/users?filter[age][gte]=25&filter[age][lte]=40&filter[city][]=NewYork&filter[city][]=Boston&filter[tags][]=premium&sort[field]=name&sort[order]=asc&page=1&limit=20
```

Developers typically resort to two workarounds, both with trade-offs:

| Approach | Problem |
|---|---|
| `GET` with query params | URL length limits (~2000 chars), poor readability, no structured body |
| `POST` for querying | Semantically wrong — POST implies mutation, not safe, not cacheable |

The HTTP `QUERY` method was proposed to solve this cleanly.

## What is the QUERY Method?

The `QUERY` method is defined in the IETF draft [draft-ietf-httpbis-safe-method-w-body](https://datatracker.ietf.org/doc/draft-ietf-httpbis-safe-method-w-body/). It is a **safe, idempotent, cacheable** HTTP method that allows a **request body** to describe the query.

```mermaid
graph TD
    A([HTTP Methods]) --> B([Unsafe Methods])
    A --> C([Safe Methods])

    B --> D[POST]
    B --> E[PUT]
    B --> F[DELETE]
    B --> G[PATCH]

    C --> H[GET]
    C --> I[HEAD]
    C --> J[OPTIONS]
    C --> K[QUERY ✦ New]

    style A fill:#1a1a2e,stroke:#b845f5,color:#f8f9fa
    style B fill:#1a1a2e,stroke:#ff6b6b,color:#ff6b6b
    style C fill:#1a1a2e,stroke:#69db7c,color:#69db7c
    style D fill:#1a1a2e,stroke:#ff6b6b,color:#ff6b6b
    style E fill:#1a1a2e,stroke:#ff6b6b,color:#ff6b6b
    style F fill:#1a1a2e,stroke:#ff6b6b,color:#ff6b6b
    style G fill:#1a1a2e,stroke:#ff6b6b,color:#ff6b6b
    style H fill:#1a1a2e,stroke:#4dabf7,color:#4dabf7
    style I fill:#1a1a2e,stroke:#69db7c,color:#69db7c
    style J fill:#1a1a2e,stroke:#69db7c,color:#69db7c
    style K fill:#b845f5,stroke:#b845f5,color:#fff
```

## QUERY vs GET vs POST

```mermaid
graph LR
    subgraph GET["GET"]
        G1["✅ Safe"]
        G2["✅ Idempotent"]
        G3["✅ Cacheable"]
        G4["❌ No body"]
    end

    subgraph POST["POST"]
        P1["❌ Not Safe"]
        P2["❌ Not Idempotent"]
        P3["❌ Not Cacheable"]
        P4["✅ Has body"]
    end

    subgraph QUERY["QUERY ✦"]
        Q1["✅ Safe"]
        Q2["✅ Idempotent"]
        Q3["✅ Cacheable"]
        Q4["✅ Has body"]
    end

    style GET fill:#1a1a2e,stroke:#4dabf7,color:#4dabf7
    style POST fill:#1a1a2e,stroke:#ff6b6b,color:#ff6b6b
    style QUERY fill:#1a1a2e,stroke:#b845f5,color:#b845f5
```

| Property | GET | POST | QUERY |
|---|---|---|---|
| Safe (no side effects) | ✅ | ❌ | ✅ |
| Idempotent | ✅ | ❌ | ✅ |
| Cacheable | ✅ | ❌ | ✅ |
| Request body | ❌ | ✅ | ✅ |
| Semantic meaning | Fetch resource | Create / action | Query with criteria |

## Request Format

A `QUERY` request looks like this:

```http
QUERY /api/v1/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Accept: application/json

{
  "filter": {
    "age": { "gte": 25, "lte": 40 },
    "city": ["New York", "Boston"],
    "status": "active",
    "tags": ["premium"]
  },
  "sort": {
    "field": "name",
    "order": "asc"
  },
  "page": 1,
  "limit": 20
}
```

The response is identical to what you'd expect from a `GET`:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": [...],
  "total": 142,
  "page": 1,
  "limit": 20
}
```

## Decision Flow — When to Use QUERY

```mermaid
flowchart TD
    A([Need to fetch data?]) --> B{Simple criteria?\nFits in URL?}
    B -->|Yes| C[Use GET]
    B -->|No| D{Mutating\nserver state?}
    D -->|Yes| E[Use POST / PUT / PATCH]
    D -->|No — read only| F{Need\ncaching?}
    F -->|Yes| G([Use QUERY ✦])
    F -->|No| H[Use POST as workaround]

    style A fill:#1a1a2e,stroke:#b845f5,color:#f8f9fa
    style G fill:#b845f5,stroke:#b845f5,color:#fff
    style C fill:#1a1a2e,stroke:#4dabf7,color:#4dabf7
    style E fill:#1a1a2e,stroke:#ff6b6b,color:#ff6b6b
    style H fill:#1a1a2e,stroke:#ff6b6b,color:#ff6b6b,stroke-dasharray:4
```

## Implementing QUERY in Spring Boot

Spring Boot doesn't natively support `QUERY` yet since it's still a draft standard. Here's how to handle it today:

### Step 1 — Define the Request Body

```java
public record UserQueryRequest(
    UserFilter filter,
    SortCriteria sort,
    int page,
    int limit
) {}

public record UserFilter(
    Integer ageGte,
    Integer ageLte,
    List<String> city,
    String status,
    List<String> tags
) {}

public record SortCriteria(String field, String order) {}
```

### Step 2 — Handle QUERY in the Controller

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @RequestMapping(method = RequestMethod.valueOf("QUERY"))
    public ResponseEntity<PagedResponse<User>> queryUsers(
            @RequestBody UserQueryRequest request) {
        return ResponseEntity.ok(userService.query(request));
    }
}
```

### Step 3 — Service Layer with JPA Specifications

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public PagedResponse<User> query(UserQueryRequest request) {
        Specification<User> spec = buildSpec(request.filter());
        Pageable pageable = PageRequest.of(
            request.page() - 1,
            request.limit(),
            Sort.by(Sort.Direction.fromString(request.sort().order()), request.sort().field())
        );
        Page<User> result = userRepository.findAll(spec, pageable);
        return new PagedResponse<>(result.getContent(), result.getTotalElements());
    }

    private Specification<User> buildSpec(UserFilter filter) {
        return Specification
            .where(ageGte(filter.ageGte()))
            .and(ageLte(filter.ageLte()))
            .and(cityIn(filter.city()))
            .and(statusEq(filter.status()));
    }

    private Specification<User> ageGte(Integer age) {
        return age == null ? null :
            (root, query, cb) -> cb.greaterThanOrEqualTo(root.get("age"), age);
    }

    private Specification<User> ageLte(Integer age) {
        return age == null ? null :
            (root, query, cb) -> cb.lessThanOrEqualTo(root.get("age"), age);
    }

    private Specification<User> cityIn(List<String> cities) {
        return (cities == null || cities.isEmpty()) ? null :
            (root, query, cb) -> root.get("city").in(cities);
    }

    private Specification<User> statusEq(String status) {
        return status == null ? null :
            (root, query, cb) -> cb.equal(root.get("status"), status);
    }
}
```

## Caching QUERY Responses

One of the biggest advantages of `QUERY` over `POST` is cacheability.

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Cache as Cache Layer
    participant Server

    Client->>Cache: QUERY /users { filter... }
    Cache->>Server: Cache MISS — forward request
    Server-->>Cache: 200 OK + Cache-Control: max-age=60
    Cache-->>Client: 200 OK (response stored)

    Note over Client,Cache: Same request again

    Client->>Cache: QUERY /users { filter... }
    Cache-->>Client: 200 OK ⚡ served from cache
```

Add cache headers in your controller:

```java
@RequestMapping(method = RequestMethod.valueOf("QUERY"))
public ResponseEntity<PagedResponse<User>> queryUsers(
        @RequestBody UserQueryRequest request) {
    PagedResponse<User> result = userService.query(request);
    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS))
        .body(result);
}
```

## Testing QUERY Endpoints

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {

    @Autowired
    MockMvc mockMvc;

    @Autowired
    ObjectMapper objectMapper;

    @Test
    void shouldReturnFilteredUsers() throws Exception {
        UserQueryRequest request = new UserQueryRequest(
            new UserFilter(25, 40, List.of("New York"), "active", null),
            new SortCriteria("name", "asc"),
            1, 20
        );

        mockMvc.perform(request("/api/v1/users")
                .method("QUERY", objectMapper.writeValueAsBytes(request))
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data").isArray())
            .andExpect(jsonPath("$.total").isNumber());
    }
}
```

## QUERY in GraphQL and OData Context

```mermaid
flowchart LR
    subgraph Today["Today"]
        A1["GraphQL\nPOST /graphql + body"]
        A2["OData\nGET with $filter params"]
        A3["REST\nPOST as query workaround"]
    end

    subgraph Future["With QUERY"]
        B1["GraphQL\nQUERY /graphql + body ✦"]
        B2["OData\nQUERY with structured body ✦"]
        B3["REST\nQUERY — clean semantics ✦"]
    end

    A1 -->|adopt QUERY| B1
    A2 -->|adopt QUERY| B2
    A3 -->|adopt QUERY| B3

    style Today fill:#1a1a2e,stroke:#495057,color:#6c757d
    style Future fill:#1a1a2e,stroke:#b845f5,color:#b845f5
    style B1 fill:#1a1a2e,stroke:#b845f5,color:#b845f5
    style B2 fill:#1a1a2e,stroke:#b845f5,color:#b845f5
    style B3 fill:#1a1a2e,stroke:#b845f5,color:#b845f5
```

GraphQL over HTTP spec is already discussing adopting `QUERY` to replace `POST /graphql` for read-only operations.

## Current Status and Timeline

```mermaid
timeline
    title HTTP QUERY Method — IETF Timeline
    2021 : Initial IETF draft proposed
         : Safe method with request body concept introduced
    2022 : Community feedback and revisions
         : Discussion on caching semantics
    2023 : Draft updated — gaining traction
         : GraphQL over HTTP references QUERY
    2024 : Wider adoption discussions
         : Framework authors begin evaluating support
    2025 : Still in draft — no official RFC yet
         : Spring and Express evaluating native support
    Future : Expected standardization as official RFC
```

| Environment | Support |
|---|---|
| Browsers (fetch / XHR) | ✅ Custom methods work |
| Spring Boot | ✅ Via `RequestMethod.valueOf("QUERY")` |
| Express.js (Node) | ✅ Via custom routing |
| Nginx / reverse proxies | ⚠️ May need explicit allowlist |
| CDN caching (CloudFront, etc.) | ⚠️ Needs custom cache policy |
| OpenAPI / Swagger | ❌ Not yet supported natively |

## Key Takeaways

1. `QUERY` fills the gap between `GET` (no body) and `POST` (not safe/cacheable) for complex read operations
2. It is safe, idempotent, and cacheable — making it semantically correct for querying
3. Spring Boot supports it today via `RequestMethod.valueOf("QUERY")`
4. It's still an IETF draft — check proxy and CDN support before using in production
5. GraphQL and OData communities are already aligning with `QUERY` for future specs

> The best HTTP method is the one that accurately describes what you're doing. `QUERY` finally gives complex read operations the method they deserve.

The `QUERY` method won't replace `GET` or `POST` — it completes the picture. As the draft moves toward standardization, expect native framework support to follow quickly.
