---
title: "Getting Started with JUnit 5"
description: "A practical introduction to JUnit 5 — Java's go-to unit testing framework — covering annotations, assertions, parameterized tests, and best practices."
date: 2026-04-12
tags: ["Java", "JUnit", "Testing"]
cover: "images/junit-cover.svg"
---

# 🚀 Getting Started with JUnit 5

---

## 🤔 Why JUnit 5?

JUnit 5 is the **foundation of unit testing in Java**.

It provides:
- Clean API
- Modular architecture
- Java 8+ support (lambdas, streams)
- Better extensibility

> 💡 JUnit 5 = JUnit Platform + JUnit Jupiter + JUnit Vintage

---

## 🧩 JUnit 5 Architecture

```mermaid
flowchart TD
    A[JUnit Platform] --> B[JUnit Jupiter]
    A --> C[JUnit Vintage]
    B --> D[New Tests]
    C --> E[Legacy JUnit 4 Tests]

## Setup

Add the dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

## Core Annotations

```java
public class OrderServiceTest {

    @BeforeEach
    void setup() {
        // runs before each test
    }

    @Test
    void shouldCalculateTotalCorrectly() {
        Order order = new Order(List.of(new Item(100), new Item(50)));
        assertEquals(150, order.getTotal());
    }

    @AfterEach
    void teardown() {
        // runs after each test
    }
}
```

| Annotation | Runs |
|---|---|
| `@BeforeAll` / `@AfterAll` | Once per test class (must be static) |
| `@BeforeEach` / `@AfterEach` | Before/after each test method |
| `@Test` | Marks a method as a test |
| `@Disabled` | Skips a test |

## Assertions

JUnit 5's built-in assertions cover most scenarios:

```java
@Test
void assertionExamples() {
    assertEquals(42, calculator.add(20, 22));
    assertTrue(user.isActive());
    assertThrows(IllegalArgumentException.class, () -> new Order(null));
    assertAll(
        () -> assertEquals("John", user.getFirstName()),
        () -> assertEquals("Doe", user.getLastName())
    );
}
```

`assertAll` is particularly useful — it runs all assertions and reports every failure, not just the first one.

## Parameterized Tests

Test the same logic with multiple inputs using `@ParameterizedTest`:

```java
@ParameterizedTest
@ValueSource(ints = {-1, 0, -100})
void shouldRejectNonPositiveAmounts(int amount) {
    assertThrows(IllegalArgumentException.class, () -> new Order(amount));
}

@ParameterizedTest
@CsvSource({"apple,1.5", "banana,0.75", "cherry,3.0"})
void shouldReturnCorrectPrice(String item, double expectedPrice) {
    assertEquals(expectedPrice, priceService.getPrice(item));
}
```

## Mocking with Mockito

JUnit 5 pairs naturally with Mockito for isolating dependencies:

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock
    PaymentGateway gateway;

    @InjectMocks
    PaymentService paymentService;

    @Test
    void shouldProcessPaymentSuccessfully() {
        when(gateway.charge(100)).thenReturn(true);
        assertTrue(paymentService.pay(100));
        verify(gateway).charge(100);
    }
}
```

## Key Takeaways

1. **Use `assertAll`** — get a full picture of failures in one run
2. **`@ParameterizedTest` over copy-paste** — keep tests DRY for boundary cases
3. **One assertion concept per test** — makes failures easy to diagnose
4. **Name tests like sentences** — `shouldRejectNullInput` beats `testMethod1`

> Unit tests are documentation that never goes out of date.

JUnit 5 is the right default for unit testing in Java. Reach for TestNG when you need advanced features like cross-test dependencies or XML-driven suite configuration.
