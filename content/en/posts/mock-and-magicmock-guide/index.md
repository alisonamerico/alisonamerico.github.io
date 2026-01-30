---
title: Understanding Mock and MagicMock in Python
date: 2026-01-13
draft: false
tags: ["python", "tests", "unittest", "mock"]
categories: ["testing"]
description: "A didactic guide to Mock and MagicMock in Python, with simple explanations, practical examples, and solutions to common problems."
cover: mock-magicmock.png
---

<!-- ![](mock-magicmock.png) -->

This article was written so **anyone can understand it** (if there is anything unclear, let me know so it can be improved), even if you are just getting started with testing in Python.

We will explain **what each concept is, why it exists, and when to use it**, always with practical examples and clear explanations.

---

## 1. What is a Mock? (simple explanation)

A **Mock** is a **fake object**, created only for tests, that **pretends to be a real object**.

It allows you to:
- Simulate functions, methods, classes, and entire objects
- Define return values and exceptions
- Record calls (count, arguments, order)
- Isolate the unit of code under test

Instead of using:
- a real database,
- an external API,
- an email service,
- or a complex system,

we use a mock to **simulate this behavior**.

### Why is this important?

Because in tests we want:
- speed,
- predictability,
- isolation.

### Mental example

Imagine a function that sends emails:

```python
def send_email(to):
    print("Sending email")
```

In a test, you **do not want to send real emails**.
So you replace this function with a mock.

The mock:
- sends nothing,
- but records whether it was called,
- with which arguments,
- and how many times.

👉 That is why we say a mock is **flexible**: you define how it should behave.

👉 Any attribute accessed on a `Mock` **automatically becomes another Mock**.

---

## 2. What is MagicMock?

`MagicMock` is a special type and subclass of `Mock`.

It exists to simulate objects that use **Python magic methods**.

### What are magic methods?

They are methods that:
- start and end with `__`
- are called automatically by Python

Common examples:
- `__len__` → called when using `len(obj)`
- `__iter__` → used in `for` loops
- `__getitem__` → used with `obj[x]`
- `__contains__` → used with `in`
- `__str__` → used with `print(obj)`

These methods are **not called directly**, but by Python itself.

### Why does MagicMock exist?

A regular `Mock` **does not handle these methods well**.
`MagicMock` is already prepared for them.

---

## 3. Mock ≠ Stub ≠ Fake (important differences)

### Stub
A **stub** only returns fixed values.

```python
def get_user_stub():
    return {"id": 1}
```

### Fake
A **fake** has a simple but functional implementation.

```python
class FakeEmailService:
    def __init__(self):
        self.sent = []

    def send(self, to):
        self.sent.append(to)
```

### Mock
A **mock** records calls and validates interactions.

```python
from unittest.mock import Mock

email = Mock()
email.send("a@test.com")
email.send.assert_called_once_with("a@test.com")
```

---

## 4. Main attributes and methods

### return_value
Defines the value returned by the mock.
```python
mock.func.return_value = 10
```

### side_effect
Allows exceptions, functions, or multiple return values.
```python
mock.func.side_effect = Exception("Error")
```

### called / call_count
Indicate whether and how many times it was called.
```python
assert mock.func.called
assert mock.func.call_count == 1
```

### call_args / call_args_list
Show the arguments used.
```python
mock.func(1)
mock.func(2)

assert mock.func.call_args_list == [((1,),), ((2,),)]
```

---

## 5. patch — replacing dependencies

Use `patch` to temporarily replace real dependencies with mocks.

Golden rule:

> **You must patch where the object is USED, not where it is DEFINED.**

```python
# app/services.py
from app.email import send_email

def notify(user):
    send_email(user.email)
```

```python
with patch("app.services.send_email") as mock_send:
    notify(user)
```

❌ WRONG:
```python
patch("app.email.send_email")
```

---

## 6. spec and autospec

Use them to avoid silent errors and ensure correct signatures.

### spec
Restricts valid attributes.

```python
mock = Mock(spec=MyClass)
```

### autospec
Restricts attributes **and the function signature**.

```python
@patch("module.func", autospec=True)
def test(mock_func):
    ...
```

✅ Prevents silent errors.

---

## 7. Mock with logic → use Fake

> If a mock contains complex logic, turn it into a fake.

---

## 8. Common problems and how to solve them

### ❌ Mock does not work
➡️ Patch applied in the wrong place

### ❌ Test passes but breaks in production
➡️ Missing autospec

### ❌ Test is hard to understand
➡️ Too many mocks

### ❌ Mock with logic
➡️ Extract into a fake

---

## 9. When to use (and when NOT to use)

### Use mocks when:
- There are HTTP calls
- There is database access
- There is file read/write
- There are external services
- There are hard-to-reproduce dependencies

### Do NOT use mocks when:
- You are testing pure logic
- The test becomes a copy of the implementation
- The mock starts having its own logic

> Practical rule: **mock dependencies, not business rules**.

---

## 10. Real best practices

- Use `autospec=True`
- Mock less, test more behavior
- Name your mocks well
- One test, one scenario

---

## 11. Anti-patterns

🚫 Mocking private methods  
🚫 Mocking domain logic  
🚫 Testing only calls  
🚫 Excessive nested mocks  

---

## 12. Mental checklist

Before creating a mock:
- Is this an external dependency?
- Does this test really need it?
- Would a fake be simpler?
- Am I testing behavior or implementation?

---

## References

- https://docs.python.org/3/library/unittest.mock.html
