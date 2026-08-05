---
title: "Pydantic in Practice (Part 1) - Fundamentals and Data Validation in Python"
date: 2026-08-05
draft: false
tags: ["python", "pydantic", "data-validation", "backend", "fastapi"]
categories: ["Python"]
description: "An introduction to Pydantic V2 covering its core concepts, installation with uv, basic models, automatic type validation, and a real-world API payload validation example."
series: ["Pydantic in Practice"]
cover: pydantic-fundamentals.png
---

## Introduction

If you build APIs, consume external data, or have ever debugged a `TypeError` that only showed up in production because a field arrived as a string instead of a number, this post is for you. Let's talk about **Pydantic**, one of the most widely used libraries in the modern Python ecosystem - one that's probably already running under the hood of some project of yours, even if you never imported it directly (FastAPI, for instance, is built on top of it).

This is **Part 1** of a two-post series. Here we'll cover the fundamentals: what Pydantic is, how to install it, how to build your first models, and how to validate data in practice, wrapping up with a real-world use case. In **Part 2**, we'll move on to intermediate/advanced topics: custom validators, `model_config`, nested types, `Annotated`, Pydantic Settings, performance, and more.

## Who created Pydantic?

Pydantic was created by **Samuel Colvin**, a British engineer who started the project in 2017 to solve a problem he kept running into himself: validating input data in Python applications without writing a mountain of repetitive `if`/`raise` code. Colvin later founded **Pydantic Services Inc.**, the company that now maintains the project and also builds related tools such as **Pydantic AI** (an AI agent framework) and **Pydantic Logfire** (observability). The project is open source, maintained by a core team and an active community on GitHub, and is used by companies such as Amazon, Microsoft, NASA, NVIDIA, Netflix, and Anthropic itself.

Today Pydantic is on its second architectural generation: starting with version 2.0, the core validation engine was rewritten in **Rust** (the `pydantic-core` package), which brought significant performance gains over V1 - the official docs describe it as one of the fastest data-validation libraries in the Python ecosystem. This post uses the latest release in the 2.x series.

## What is Pydantic and why use it?

Pydantic is a **data validation and modeling library** for Python. In practice, you declare the "shape" of your data using native Python **type hints** (`str`, `int`, `list[str]`, `datetime`, and so on), and Pydantic handles three things for you:

1. **Validation** - makes sure incoming data actually matches the declared types, rejecting anything that doesn't with a clear error.
2. **Coercion** - when it's possible and safe, it automatically converts compatible types (e.g., the string `"25"` becomes the integer `25`).
3. **Serialization** - turns Python objects back into dictionaries or JSON, ready to travel over an API, be stored in a database, or logged.

The key insight behind Pydantic is that it reuses the typing syntax you already write (or should be writing) in modern Python. It's not a new DSL to learn - it's `class User(BaseModel)` with type annotations, and everything else is automatic.

### What problem does Pydantic solve?

Every backend that receives data from the outside world (an HTTP request, a queue message, an environment variable, a config file, a response from another API) faces the same issue: **you can't blindly trust the shape of that data**. Without a validation layer, it's common to end up writing code like this:

```python
def create_user(data: dict):
    if "name" not in data or not isinstance(data["name"], str):
        raise ValueError("invalid name")
    if "age" not in data or not isinstance(data["age"], int):
        raise ValueError("invalid age")
    # ... and so on, for every single field
```

That code grows, becomes repetitive, is easy to under-cover with a missing check, and gets harder to maintain as the data model evolves. Pydantic replaces those manual checks with a **single, typed declaration**, validated automatically every time an object is created. The result is less boilerplate, standardized error messages, and much more confidence at every boundary where "untrusted" data enters your system.

### When should you use Pydantic?

- Validating the request body of an API (it's literally FastAPI's validation engine).
- Validating and typing environment variables and configuration files (via `pydantic-settings`).
- Guaranteeing the integrity of messages received from queues (RabbitMQ, Celery, Kafka).
- Modeling domain data explicitly, instead of passing loose `dict`s around your codebase.
- Serializing/deserializing data for structured logs, cache (Redis), or API responses.

## Installation and initial setup (with uv)

We'll use **uv** for the setup, since it's currently the fastest and most convenient package/environment manager in the Python ecosystem.

```bash
# Create a new project (if you don't have one already)
uv init pydantic-lab
cd pydantic-lab

# Add Pydantic as a project dependency
uv add pydantic

# (optional) Add pydantic-settings, used for env-var-based configuration
uv add pydantic-settings
```

`uv add` resolves the version, updates `pyproject.toml`, and creates/updates the `uv.lock` file - no need to manage a `venv` manually. To run a script from the project, use:

```bash
uv run python my_script.py
```

If you prefer the traditional `pip` flow, that still works fine:

```bash
pip install pydantic
```

To check the installed version:

```bash
uv run python -c "import pydantic; print(pydantic.VERSION)"
```

## Introduction to basic models (`BaseModel`)

The starting point for almost everything in Pydantic is the `BaseModel` class. You subclass it to create your own models, declaring fields as class attributes with type hints:

```python
from pydantic import BaseModel


class User(BaseModel):
    name: str
    age: int
    email: str
    active: bool = True  # field with a default value -> not required
```

A few important things already show up in this simple example:

- `name`, `age`, and `email` are **required**, since they have no default value.
- `active` is **optional**, because it already has a default (`True`). If it's not provided at creation time, it falls back to that value.
- Field order doesn't matter for object creation (Pydantic accepts `**kwargs`), but it does matter for readability.

Creating an instance:

```python
user = User(name="Alison", age=34, email="alison@example.com")
print(user)
#> name='Alison' age=34 email='alison@example.com' active=True
```

## Automatic type validation with Pydantic

Every time you instantiate a model, Pydantic runs validation on all fields automatically - you don't need to call any explicit method for that. Let's see it in action:

```python
from pydantic import BaseModel, ValidationError


class User(BaseModel):
    name: str
    age: int
    email: str


# Valid data
valid_user = User(name="Mary", age=30, email="mary@example.com")
print(valid_user)
#> name='Mary' age=30 email='mary@example.com'

# Invalid data
try:
    invalid_user = User(name="John", age="thirty", email="john@example.com")
except ValidationError as e:
    print(e)
```

The resulting error (abbreviated):

```text
1 validation error for User
age
  Input should be a valid integer, unable to parse string as an integer
  [type=int_parsing, input_value='thirty', input_type=str]
```

Two important details here:

1. Pydantic raises **`ValidationError`** (not a generic `ValueError`) - its own exception type that carries a structured list of every error found, field by field.
2. The message already tells you exactly **which field failed**, **why**, and **what value was received** - great both for debugging and for returning as an API error response.

## Working with valid and invalid data

### Type coercion (valid data with conversion)

By default (in "lax" mode), Pydantic tries to convert compatible types automatically instead of rejecting them outright:

```python
user = User(name="Anna", age="25", email="anna@example.com")
print(user)
#> name='Anna' age=25 email='anna@example.com'
print(type(user.age))
#> <class 'int'>
```

Even though `"25"` was received as a string, the `age` field gets converted to `int`. This is extremely useful when data comes from sources like HTML forms, query strings, or environment variables, where everything naturally arrives as text.

### Validation errors (genuinely invalid data)

When conversion isn't possible or safe, Pydantic rejects the data and lists every problem at once:

```python
from pydantic import BaseModel, ValidationError


class User(BaseModel):
    name: str
    age: int
    email: str


try:
    user = User(name=123, age=25.5, email=None)
except ValidationError as e:
    print(e)
```

```text
3 validation errors for User
name
  Input should be a valid string [type=string_type, input_value=123, input_type=int]
age
  Input should be a valid integer, got a number with a fractional part [type=int_from_float, input_value=25.5, input_type=float]
email
  Input should be a valid string [type=string_type, input_value=None, input_type=NoneType]
```

Notice that Pydantic **doesn't stop at the first error** - it collects every validation error for the model at once, which is much better than fixing one field at a time through trial and error.

### Strict mode

If you don't want any automatic coercion at all - for example, refusing `"25"` as a valid `age` - Pydantic also offers **strict mode**, available both per-field and per-model. This is useful when data already comes from a fully trusted source (like another internal service that also uses Pydantic), where any type mismatch should be treated as a bug rather than something to silently "fix."

## JSON serialization and deserialization

This is, in practice, one of the most-used features in the day-to-day work of building APIs.

### Serialization (model → JSON)

```python
user = User(name="Charles", age=40, email="charles@example.com")

# To a Python dict
print(user.model_dump())
#> {'name': 'Charles', 'age': 40, 'email': 'charles@example.com'}

# Directly to a JSON string
print(user.model_dump_json())
#> {"name":"Charles","age":40,"email":"charles@example.com"}
```

`model_dump()` returns a Python `dict`; `model_dump_json()` returns a JSON string, ready to go straight into an HTTP response.

### Deserialization (JSON → model)

```python
json_data = '{"name": "Louise", "age": 22, "email": "louise@example.com"}'

user = User.model_validate_json(json_data)
print(user)
#> name='Louise' age=22 email='louise@example.com'
```

If you're starting from a `dict` you already have in memory (say, an already-parsed request body), use `model_validate()`:

```python
dict_data = {"name": "Peter", "age": 28, "email": "peter@example.com"}
user = User.model_validate(dict_data)
```

> 💡 In older versions of Pydantic (V1) you probably saw `parse_raw()` and `.dict()`. Those methods still exist for compatibility, but they're **deprecated** - in V2, the standard methods are `model_validate_json()`, `model_validate()`, `model_dump()`, and `model_dump_json()`.

## A bit more: `Field`, optional types, and default values

Before jumping into the practical case, it's worth knowing about `Field`, used to add constraints and metadata to fields:

```python
from typing import Optional
from pydantic import BaseModel, Field


class Product(BaseModel):
    name: str = Field(min_length=2, max_length=100)
    price: float = Field(gt=0, description="Price in dollars, must be positive")
    stock: int = Field(default=0, ge=0)
    description: Optional[str] = None
```

- `min_length` / `max_length`: size constraints for strings.
- `gt` / `ge` / `lt` / `le`: numeric constraints ("greater than", "greater or equal", etc.).
- `default`: an explicit default value (an alternative to `= value` directly on the attribute).
- `Optional[str] = None`: a field that accepts `None` and isn't required.

## Practical case (real scenario): validating a webhook payload

An extremely common day-to-day backend scenario: you expose an endpoint that receives **webhooks** from an external service (a payment gateway, CI/CD, a third-party system) and need to validate the payload before processing anything. Let's build this scenario with FastAPI + Pydantic, validating an "order created" event from an e-commerce system:

```python
from datetime import datetime
from typing import Literal

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, EmailStr, Field, ValidationError


class OrderItem(BaseModel):
    sku: str = Field(min_length=1)
    quantity: int = Field(gt=0)
    unit_price: float = Field(gt=0)


class OrderCreatedWebhook(BaseModel):
    event: Literal["order.created"]
    order_id: str
    customer_email: EmailStr
    items: list[OrderItem]
    created_at: datetime
    total_amount: float = Field(gt=0)


app = FastAPI()


@app.post("/webhooks/orders")
def receive_webhook(payload: OrderCreatedWebhook):
    # By the time we get here, Pydantic has already validated:
    # - the type of every field
    # - that "event" is exactly "order.created"
    # - that the email is a valid one (EmailStr)
    # - that every item has a positive quantity and price
    # - that the date is a valid ISO 8601 string
    calculated_total = sum(item.quantity * item.unit_price for item in payload.items)

    if abs(calculated_total - payload.total_amount) > 0.01:
        raise HTTPException(status_code=422, detail="total_amount doesn't match the items")

    return {"status": "processed", "order_id": payload.order_id}
```

What we gain here, in practice:

- **Zero manual validation** of types, email format, or date format inside the function body - FastAPI automatically rejects (with `422 Unprocessable Entity`) any payload that doesn't match `OrderCreatedWebhook`, before `receive_webhook` is even called.
- **`Literal["order.created"]`** ensures only that specific event type is accepted on this endpoint - useful when the same webhook source can send different kinds of events.
- **`EmailStr`** validates the email format automatically (requires the extra `email-validator` package, installable with `uv add pydantic[email]`).
- The business rule (**checking that the total matches the sum of the items**) stays isolated, once we already have the guarantee that the types are correct - without mixing structural validation with business logic.

Testing it locally with `curl`:

```bash
curl -X POST http://localhost:8000/webhooks/orders \
  -H "Content-Type: application/json" \
  -d '{
    "event": "order.created",
    "order_id": "ORD-001",
    "customer_email": "customer@example.com",
    "items": [{"sku": "ABC123", "quantity": 2, "unit_price": 49.9}],
    "created_at": "2026-08-04T10:00:00",
    "total_amount": 99.8
  }'
```

## Conclusion

In this first part, you saw:

- Who created Pydantic and why it exists.
- The real problem it solves (validating data from untrusted sources).
- How to install and set up a project with `uv`.
- How to build basic models with `BaseModel`.
- How automatic type validation and coercion work.
- How to work with valid and invalid data, including how to read a `ValidationError`.
- How to serialize and deserialize JSON with `model_dump_json()` and `model_validate_json()`.
- A complete, real-world example validating a webhook with FastAPI.

In **Part 2** of this series, we'll dig into intermediate and advanced topics: custom validators (`field_validator` and `model_validator`), `model_config`, nested and recursive models, `Annotated` with custom validation, `pydantic-settings` for environment-variable-based configuration, and a few notes on performance - again wrapping up with a real-world practical case.

## Links

- [Official Pydantic docs - Get Started](https://pydantic.dev/docs/validation/latest/get-started/)
- [Official GitHub repository](https://github.com/pydantic/pydantic)
- [FastAPI documentation (uses Pydantic internally)](https://fastapi.tiangolo.com/)
- [uv - Python package manager](https://docs.astral.sh/uv/)
