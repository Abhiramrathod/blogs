---
title: "Getting Started with TestNG"
description: "A practical introduction to TestNG — Java's powerful testing framework — covering annotations, test configuration, grouping, and data-driven testing."
date: 2026-04-10
tags: ["Java", "TestNG", "Testing"]
cover: "images/testng-cover.svg"
---

## Why TestNG?

JUnit is great for simple unit tests, but as your test suite grows, you need more control — parallel execution, test grouping, dependency management, and data-driven testing. TestNG (Test Next Generation) was built for exactly this.

## Setup

Add the dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>org.testng</groupId>
    <artifactId>testng</artifactId>
    <version>7.9.0</version>
    <scope>test</scope>
</dependency>
```

## Core Annotations

TestNG's annotations give you fine-grained control over test lifecycle:

```java
public class UserServiceTest {

    @BeforeClass
    public void setup() {
        // runs once before all tests in this class
    }

    @Test
    public void shouldCreateUser() {
        User user = new User("john@example.com");
        assertNotNull(user.getId());
    }

    @AfterClass
    public void teardown() {
        // runs once after all tests in this class
    }
}
```

| Annotation | Runs |
|---|---|
| `@BeforeSuite` / `@AfterSuite` | Once per entire test suite |
| `@BeforeClass` / `@AfterClass` | Once per test class |
| `@BeforeMethod` / `@AfterMethod` | Before/after each test method |
| `@Test` | Marks a method as a test |

## Grouping Tests

Group tests by feature or type and run only what you need:

```java
@Test(groups = "smoke")
public void shouldLoginSuccessfully() { ... }

@Test(groups = "regression")
public void shouldHandleInvalidCredentials() { ... }
```

Run specific groups via `testng.xml`:

```xml
<suite name="Suite">
    <test name="Smoke Tests">
        <groups>
            <run><include name="smoke"/></run>
        </groups>
        <classes>
            <class name="com.example.AuthTest"/>
        </classes>
    </test>
</suite>
```

## Data-Driven Testing with @DataProvider

Test the same logic with multiple inputs without duplicating test methods:

```java
@DataProvider(name = "invalidEmails")
public Object[][] invalidEmails() {
    return new Object[][] {
        {"notanemail"},
        {"missing@domain"},
        {""}
    };
}

@Test(dataProvider = "invalidEmails")
public void shouldRejectInvalidEmail(String email) {
    assertFalse(emailValidator.isValid(email));
}
```

## Parallel Execution

Run tests in parallel to cut down suite execution time — configure in `testng.xml`:

```xml
<suite name="Suite" parallel="methods" thread-count="4">
    <test name="All Tests">
        <classes>
            <class name="com.example.UserServiceTest"/>
        </classes>
    </test>
</suite>
```

## Key Takeaways

1. **Use `@DataProvider`** — eliminates repetitive test code for boundary cases
2. **Group your tests** — run smoke tests on every commit, full regression before release
3. **Leverage parallel execution** — large suites can run significantly faster
4. **Prefer `@BeforeClass` over `@BeforeMethod`** — for expensive setup like DB connections

> A well-organized test suite is as important as the production code it covers.

TestNG's flexibility makes it a strong choice for integration and end-to-end testing, especially in enterprise Java projects where test suite complexity grows quickly.
