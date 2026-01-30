---
title: Testing with Pytest - Fundamentals, Best Practices and Strategy
date: 2026-01-30
draft: false
tags: ["python", "tests", "pytest"]
categories: ["testing"]
description: "Complete theoretical guide on automated testing with Pytest, covering from fundamental concepts to advanced strategies. Learn about the test pyramid, AAA pattern, fixtures, mocks, markers, and essential plugins to build a solid foundation."
cover: testing-with-pytest.png
---

<!-- ![](testing-with-pytest.png) -->

Testing software is not just a technical development step, it's a **quality, confidence, and sustainability strategy**. Systems without tests tend to break frequently, create fear of changes, and drastically increase maintenance costs.

In this post, we'll focus **100% on theory**, explaining fundamental concepts of automated testing using **Pytest**, the most popular testing framework in the Python ecosystem.

> 👉 In the next post of the series, we'll apply all of this to a **real Flask project consuming the GitHub public API**.

---

## What is Pytest?

Pytest is a testing framework for Python that makes it easy to create simple, readable, and scalable tests. It allows testing from isolated functions to complete systems.

### Why is Pytest so widely used?
- Simple syntax (based on `assert`)
- Automatic test discovery
- Powerful **fixtures** system
- Excellent **plugin** support
- Scales well for large projects

Compared to `unittest`, Pytest is more expressive, less verbose, and more productive.

---

## What is the purpose of tests?

Tests exist to **reduce uncertainty**.

They help to:
- Ensure code works as expected
- Detect errors quickly
- Prevent regressions
- Enable safe refactoring
- Serve as living documentation of the system

Code without tests might work today, but it's fragile tomorrow.

---

## When to write tests?

- When creating new features
- When fixing bugs (especially regressions)
- Before refactoring existing code
- In business-critical code

### And when NOT to write tests?

- Very quick proofs of concept (POCs)
- Completely disposable scripts

Even in these cases, tests can still bring benefits.

---

## Types of Tests

Understanding test types is essential to create a **balanced strategy**.

### Unit Tests

- Test isolated functions or methods
- Don't access database, network, or file system
- Are fast and cheap

✔️ Foundation of the test pyramid

---

### Integration Tests

- Test communication between components
- Example: application + database, application + external API

✔️ Detect real integration failures
❌ Slower

---

### Functional Tests

- Test complete application flows
- Simulate user behavior

✔️ High confidence
❌ More complex

---

### System / End-to-End (E2E) Tests

- Test the system as a whole
- Include multiple layers

✔️ Simulate real usage
❌ Slow and fragile

---

### Regression Tests

- Ensure fixed bugs don't come back
- Usually arise after real incidents

✔️ Protect against recurring errors

---

### Performance Tests

- Evaluate response time and resource consumption
- Identify bottlenecks

✔️ Important in critical systems
❌ Should be used with discretion

---

## The Test Pyramid

A good testing strategy follows the pyramid:

- Many unit tests
- Some integration tests
- Few E2E tests

This ensures speed, confidence, and controlled cost.

---

## The AAA Pattern (Arrange, Act, Assert)

AAA is a test organization pattern that dramatically improves readability.

### Arrange
Prepare data, mocks, and context.

### Act
Execute the action being tested.

### Assert
Validate the result.

### Benefits
- Clearer tests
- Less ambiguity
- Easier maintenance

---

## Fixtures in Pytest

Fixtures are reusable functions responsible for **preparing and cleaning up the test environment**.

### Why use fixtures?
- Avoid code duplication
- Centralize setup/teardown
- Make tests more readable

### Fixture scopes
- `function`: default (runs for each test)
- `module`: once per file
- `session`: once per execution

---

## Pytest Marks

Marks allow you to **classify and control** tests.

### parametrize

Allows running the same test with multiple data sets.

**Benefits:**
- Less code
- More coverage
- More expressive tests

---

### skip

Consciously ignores tests.

**When to use:**
- Unavailable external dependency
- Disabled functionality

---

### xfail

Marks tests that **should fail**.

**Benefits:**
- Documents known bugs
- Doesn't break the pipeline

---

### slow

Marks slow tests.

**Benefits:**
- Selective execution
- Quick feedback in daily work

---

## What is Mock?

Mock is a technique used to **simulate external dependencies**.

### When to use mock?
- External APIs
- Database
- Third-party services

### When NOT to use?
- To test internal logic
- Excessively (overmocking)

---

## unittest.mock vs pytest-mock

### unittest.mock

- Python standard library
- More verbose
- Uses context managers

✔️ Doesn't depend on plugins
❌ Code harder to read

---

### pytest-mock

- Pytest plugin
- Integration with fixtures
- Cleaner syntax

✔️ More productive
✔️ More readable

---

## Essential Pytest Plugins

### pytest-cov

- Measures code coverage
- Helps identify untested areas

⚠️ High coverage ≠ quality code

---

### pytest-mock

- Facilitates mock creation
- Reduces boilerplate

---

## General Best Practices

- Tests should be simple
- One behavior per test
- Clear and descriptive names
- Avoid complex logic in tests
- Tests should be deterministic

---

## Conclusion

Automated tests are not a luxury, they are a **professional necessity**. Pytest provides the right tools to write clear, scalable, and reliable tests.

In the next post of the series, we'll **apply all these concepts in a real Flask project**, consuming the GitHub public API, with unit, integration, functional, and performance tests.

👉 Continue to **Post 2: Pytest in Practice with Flask and GitHub API**
