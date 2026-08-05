---
title: "Pydantic in Practice (Part 2) - Intermediate & Advanced Topics with a Real-World Project"
date: 2026-08-11
draft: false
tags: ["python", "pydantic", "data-validation", "backend", "fastapi", "rabbitmq", "celery"]
categories: ["Python"]
description: "Explore custom validators, model_config, nested models, Annotated, computed_field, discriminated unions, aliases, generics, and Pydantic Settings through a practical order-processing project."
series: ["Pydantic in Practice"]
---

## Recap of Part 1

In [Part 1](/blog/pydantic-part-1-fundamentals) of this series we covered the fundamentals: what Pydantic is, who created it, installation with `uv`, building models with `BaseModel`, automatic type validation, the difference between valid/invalid data, and JSON serialization/deserialization. If you haven't read it yet, it's worth starting there - this post assumes you already know how to build a simple model and understand what a `ValidationError` is.

Here we'll move on to what actually separates "basic" Pydantic usage from "professional" production usage: custom validation rules, fine-tuning model behavior, composing complex models, reusable types, and application configuration management. We'll close with a **complete practical project**, simulating a real order-processing service consuming messages from a queue.

## Custom validators: `field_validator` and `model_validator`

Not every business rule fits into a `Field(gt=0)`. Sometimes you need custom logic - validating a tax ID, normalizing text, or comparing two fields against each other.

### `field_validator`: validating (or transforming) a specific field

```python
from pydantic import BaseModel, field_validator


class Customer(BaseModel):
    name: str
    tax_id: str

    @field_validator("tax_id")
    @classmethod
    def validate_tax_id(cls, value: str) -> str:
        digits = "".join(filter(str.isdigit, value))
        if len(digits) != 11:
            raise ValueError("tax_id must contain 11 digits")
        return digits  # normalize by stripping punctuation
```

Notice that the validator doesn't just validate - it also **normalizes** the data: the `tax_id` coming out of the model is already stripped of punctuation, regardless of how it was entered (`"123.456.789-00"` or `"12345678900"`).

The `mode` parameter controls **when** the validator runs relative to the default type validation:

- `mode="after"` (default): runs after Pydantic has already validated/converted the basic type. You receive the value already in the expected Python type.
- `mode="before"`: runs before, with the "raw" value (as received, e.g. still a string or dict). Useful for pre-processing unexpected input formats before type validation.

```python
from pydantic import BaseModel, field_validator


class Product(BaseModel):
    price: float

    @field_validator("price", mode="before")
    @classmethod
    def clean_price(cls, value):
        # accepts "$49.90" and converts it before type validation
        if isinstance(value, str):
            value = value.replace("$", "").strip()
        return value
```

### `model_validator`: cross-field validation

When a rule depends on **more than one field**, `field_validator` isn't enough - that's what `model_validator` is for, receiving the whole model (or the raw data, depending on the mode):

```python
from datetime import date
from pydantic import BaseModel, model_validator


class Booking(BaseModel):
    check_in: date
    check_out: date

    @model_validator(mode="after")
    def validate_dates(self) -> "Booking":
        if self.check_out <= self.check_in:
            raise ValueError("check_out must be later than check_in")
        return self
```

With `mode="after"`, `self` is already a validated model instance - you return `self` (possibly modified) at the end. With `mode="before"`, you receive the raw input `dict`, before any field validation happens, which is useful for renaming or merging keys before standard validation kicks in.

## `model_config` / `ConfigDict`: configuring model behavior

Every `BaseModel` can be configured with `model_config`, using `ConfigDict`:

```python
from pydantic import BaseModel, ConfigDict


class Settings(BaseModel):
    model_config = ConfigDict(
        extra="forbid",             # rejects fields not declared on the model
        frozen=True,                # makes the instance immutable after creation
        str_strip_whitespace=True,  # strips extra whitespace from strings automatically
        populate_by_name=True,      # allows populating both by field name and by alias
    )

    environment: str
    debug: bool = False
```

A few options that make the biggest difference day to day:

| Option | What it does |
|---|---|
| `extra="forbid"` | Raises an error if the payload includes unknown fields (great for catching field-name typos). Default is `"ignore"`. |
| `extra="allow"` | Accepts and keeps undeclared extra fields, accessible via `model_extra`. |
| `frozen=True` | Makes instances immutable (like `dataclass(frozen=True)`) - useful for value objects. |
| `str_strip_whitespace=True` | Automatically strips leading/trailing whitespace from every string. |
| `validate_assignment=True` | Re-validates fields every time you do `instance.field = new_value`, not just on creation. |

## Nested and recursive models

Pydantic models can be composed freely - a field can be another `BaseModel`, a list of models, or even reference its own type (recursion):

```python
from pydantic import BaseModel


class Address(BaseModel):
    street: str
    city: str
    zip_code: str


class Customer(BaseModel):
    name: str
    address: Address                    # nested model
    extra_addresses: list[Address] = []  # list of nested models
```

```python
customer = Customer(
    name="Anna",
    address={"street": "Main St 123", "city": "Recife", "zip_code": "51021-000"},
)
print(customer.address.city)
#> Recife
```

Notice we passed a `dict` for `address` and Pydantic automatically converted it into an `Address`, recursively validating each nested field - including producing a detailed `ValidationError` with the **full path** down to the problematic field (e.g. `address.zip_code`) if something's wrong.

Recursive models (a comment tree, for example) are also possible, using `list["Comment"]` with either `from __future__ import annotations` or quoted type hints:

```python
from __future__ import annotations
from pydantic import BaseModel


class Comment(BaseModel):
    text: str
    replies: list[Comment] = []
```

## `Annotated`: reusable types with built-in validation

When the same validation rule repeats across several models (say, "valid tax ID" or "non-empty string"), it's worth extracting it into a **reusable type** with `Annotated`:

```python
from typing import Annotated
from pydantic import AfterValidator, BaseModel


def validate_tax_id(value: str) -> str:
    digits = "".join(filter(str.isdigit, value))
    if len(digits) != 11:
        raise ValueError("tax_id must contain 11 digits")
    return digits


TaxID = Annotated[str, AfterValidator(validate_tax_id)]


class Customer(BaseModel):
    name: str
    tax_id: TaxID


class Employee(BaseModel):
    name: str
    tax_id: TaxID  # same validation, reused, without repeating @field_validator
```

This is the "modern" approach (and the one recommended by the official docs) to sharing validation logic across different models, instead of copying the same `field_validator` into every class.

## `computed_field`: derived fields that show up in serialization

Sometimes you want to expose a value calculated from other fields, but have it also appear in `model_dump()`/`model_dump_json()` - that's what `@computed_field` is for:

```python
from pydantic import BaseModel, computed_field


class OrderItem(BaseModel):
    unit_price: float
    quantity: int

    @computed_field
    @property
    def subtotal(self) -> float:
        return round(self.unit_price * self.quantity, 2)
```

```python
item = OrderItem(unit_price=49.9, quantity=3)
print(item.model_dump())
#> {'unit_price': 49.9, 'quantity': 3, 'subtotal': 149.7}
```

Unlike a regular `@property` (which doesn't show up in `model_dump()`), `computed_field` is treated as part of the model's output "contract" - useful for exposing totals, ages calculated from a birth date, or formatted fields without duplicating data in the database.

## Discriminated unions: validating "one of several shapes"

A common problem: a payload can represent **different shapes of the same concept** - for example, a payment can be by card or by a bank transfer, each with completely different fields. Pydantic solves this with **discriminated unions**, using a shared field (the "discriminator") to decide which model to apply:

```python
from typing import Literal, Union, Annotated
from pydantic import BaseModel, Field


class CardPayment(BaseModel):
    type: Literal["card"]
    last_digits: str
    installments: int = Field(ge=1, le=12)


class BankTransferPayment(BaseModel):
    type: Literal["bank_transfer"]
    account_key: str


Payment = Annotated[
    Union[CardPayment, BankTransferPayment],
    Field(discriminator="type"),
]


class Order(BaseModel):
    id: str
    payment: Payment
```

```python
order = Order(id="1", payment={"type": "bank_transfer", "account_key": "customer@example.com"})
print(type(order.payment))
#> <class '__main__.BankTransferPayment'>
```

The advantage of an explicit discriminator (`Field(discriminator="type")`) over a "plain" `Union` is performance and error clarity: Pydantic only looks at the `type` field to know which model to validate against, instead of trying each union member until one "fits" - and if `type` is invalid, the error points exactly to the accepted values.

## Aliases: matching field names with external APIs

It's very common to consume an API that uses `camelCase` (the standard for JSON in many JavaScript-based services) while your Python code follows `snake_case`. Aliases solve this without forcing you to "dirty" your field names:

```python
from pydantic import BaseModel, ConfigDict, Field


class Customer(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    full_name: str = Field(alias="fullName")
    birth_date: str = Field(alias="birthDate")


# Populating from the external payload (camelCase)
customer = Customer.model_validate({"fullName": "Alison Lira", "birthDate": "1990-01-01"})
print(customer.full_name)
#> Alison Lira

# Also works via the Python field name, thanks to populate_by_name=True
customer2 = Customer(full_name="Anna Smith", birth_date="1995-05-05")
```

You can also have different aliases for input and output (`validation_alias` and `serialization_alias`), when your API's input and output formats need to diverge.

## Generics: reusable models for repeated patterns

An extremely common API pattern is a **paginated response envelope** - the same structure (`items`, `total`, `page`), only the item type changes. Instead of duplicating this for every entity, we use `Generic`:

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")


class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int


class Product(BaseModel):
    id: int
    name: str


response = PaginatedResponse[Product](
    items=[{"id": 1, "name": "Keyboard"}, {"id": 2, "name": "Mouse"}],
    total=2,
    page=1,
)
print(response.items[0].name)
#> Keyboard
```

`PaginatedResponse[Product]` works as a reusable "mold" - the same generic model paginates customers, orders, products, and so on, without rewriting the whole structure each time.

## Pydantic Settings: typed application configuration

The `pydantic-settings` package (installed separately, as we saw in Part 1) applies the same validation philosophy to **application configuration**, automatically reading from environment variables and/or `.env` files:

```python
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="APP_")

    environment: str = "development"
    debug: bool = False
    database_url: str
    rabbitmq_url: str = "amqp://guest:guest@localhost:5672/"
    max_connections: int = Field(default=10, ge=1)


settings = AppSettings()
```

With `env_prefix="APP_"`, the environment variable `APP_DATABASE_URL` automatically populates `database_url`. This gives you, for free, everything we've covered so far: type validation (`max_connections` must be an integer ≥ 1), automatic type coercion (environment variables always arrive as strings, and Pydantic converts them to `bool`/`int` as declared), and clear errors at application startup if a required variable is missing - much better than discovering that at runtime, in the middle of a request.

## Performance notes

- Pydantic V2's validation core (`pydantic-core`) is written in **Rust**, making it significantly faster than V1 (which was 100% Python) - the official docs describe validation as one of the fastest among Python libraries of its kind.
- To validate data **without** defining a full `BaseModel` class (for example, validating just a `list[int]` or a primitive type with rules), use `TypeAdapter`, which avoids the overhead of creating a model just for that:

```python
from pydantic import TypeAdapter

list_validator = TypeAdapter(list[int])
print(list_validator.validate_python(["1", "2", "3"]))
#> [1, 2, 3]
```

- Each model's validation schema is **built once** (at class definition time) and reused across all instances - so avoid dynamically recreating model classes inside loops or frequently-called functions.

## Practical case (real scenario): a queue-based order processing service

Let's pull together nearly everything from both posts into a realistic scenario: a **worker** (say, a Celery/RabbitMQ consumer) that receives order messages from an external system (in camelCase), validates them, computes totals, and branches its flow based on the payment method.

```python
from datetime import datetime
from typing import Annotated, Literal, Union

from pydantic import (
    AfterValidator,
    BaseModel,
    ConfigDict,
    EmailStr,
    Field,
    computed_field,
    field_validator,
    model_validator,
)
from pydantic_settings import BaseSettings, SettingsConfigDict


# --- Application configuration -------------------------------------------------

class WorkerSettings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="WORKER_")

    rabbitmq_url: str
    orders_queue: str = "orders.process"
    max_discount_percent: float = Field(default=20.0, ge=0, le=100)


# --- Reusable types --------------------------------------------------------------

def validate_zip_code(value: str) -> str:
    digits = "".join(filter(str.isdigit, value))
    if len(digits) != 8:
        raise ValueError("zip_code must contain 8 digits")
    return f"{digits[:5]}-{digits[5:]}"


ZipCode = Annotated[str, AfterValidator(validate_zip_code)]


# --- Nested models ------------------------------------------------------------------

class Address(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    street: str
    city: str
    zip_code: ZipCode = Field(alias="zipCode")


class OrderItem(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    sku: str
    quantity: int = Field(gt=0)
    unit_price: float = Field(gt=0, alias="unitPrice")

    @computed_field
    @property
    def subtotal(self) -> float:
        return round(self.quantity * self.unit_price, 2)


# --- Discriminated union for payment method -----------------------------------------

class CardPayment(BaseModel):
    type: Literal["card"]
    last_digits: str
    installments: int = Field(ge=1, le=12)


class BankTransferPayment(BaseModel):
    type: Literal["bank_transfer"]
    account_key: str


Payment = Annotated[Union[CardPayment, BankTransferPayment], Field(discriminator="type")]


# --- Main queue message model -----------------------------------------------------

class OrderMessage(BaseModel):
    model_config = ConfigDict(populate_by_name=True, extra="forbid")

    order_id: str = Field(alias="orderId")
    customer_email: EmailStr = Field(alias="customerEmail")
    shipping_address: Address = Field(alias="shippingAddress")
    items: list[OrderItem]
    payment: Payment
    discount_percent: float = Field(default=0, ge=0, alias="discountPercent")
    created_at: datetime = Field(alias="createdAt")

    @field_validator("items")
    @classmethod
    def validate_at_least_one_item(cls, items: list[OrderItem]) -> list[OrderItem]:
        if not items:
            raise ValueError("the order must have at least one item")
        return items

    @computed_field
    @property
    def total_amount(self) -> float:
        subtotal = sum(item.subtotal for item in self.items)
        return round(subtotal * (1 - self.discount_percent / 100), 2)

    @model_validator(mode="after")
    def validate_bank_transfer_discount(self) -> "OrderMessage":
        if isinstance(self.payment, BankTransferPayment) and self.discount_percent > 0:
            # fictional business rule: bank transfer payments don't stack with coupon discounts
            raise ValueError("coupon discount is not applicable to bank transfer payments")
        return self


# --- The "worker" processing the message --------------------------------------------

def process_message(raw_payload: dict, settings: WorkerSettings) -> None:
    try:
        order = OrderMessage.model_validate(raw_payload)
    except Exception as error:
        # In a real scenario: log this in a structured way and route to a dead-letter queue
        print(f"[REJECTED] invalid payload: {error}")
        return

    if order.discount_percent > settings.max_discount_percent:
        print(f"[REJECTED] discount above the allowed limit ({settings.max_discount_percent}%)")
        return

    print(f"[OK] order {order.order_id} - total: ${order.total_amount} "
          f"- payment: {order.payment.type} - shipping to: {order.shipping_address.city}")


# --- Simulating the flow ---------------------------------------------------------------

if __name__ == "__main__":
    settings = WorkerSettings(rabbitmq_url="amqp://guest:guest@localhost:5672/")

    external_payload = {
        "orderId": "ORD-777",
        "customerEmail": "customer@example.com",
        "shippingAddress": {
            "street": "Main St 500",
            "city": "Recife",
            "zipCode": "51.021-000",
        },
        "items": [
            {"sku": "KEY-01", "quantity": 1, "unitPrice": 350.0},
            {"sku": "MOU-02", "quantity": 2, "unitPrice": 89.9},
        ],
        "payment": {"type": "bank_transfer", "account_key": "customer@example.com"},
        "discountPercent": 0,
        "createdAt": "2026-08-11T14:30:00",
    }

    process_message(external_payload, settings)
    #> [OK] order ORD-777 - total: $529.8 - payment: bank_transfer - shipping to: Recife
```

### What this project demonstrates, end to end

1. **Typed configuration** (`WorkerSettings`) loaded from environment variables, with business limits (`max_discount_percent`) already validated at startup.
2. **Aliases** (`populate_by_name` + `Field(alias=...)`) automatically translating the external camelCase payload into the snake_case field names used internally.
3. **A reusable type via `Annotated`** (`ZipCode`) normalizing and validating the zip code on any model that uses it.
4. **Nested models** (`Address`, `OrderItem`) composing the main message, with automatic recursive validation.
5. **Discriminated union** deciding, at validation time, whether the payment is `CardPayment` or `BankTransferPayment` - no manual `if`/`isinstance` needed.
6. **`computed_field`** calculating `subtotal` per item and `total_amount` for the order, automatically reflected in any future serialization.
7. **`field_validator`** enforcing a simple collection rule (at least one item).
8. **`model_validator`** enforcing a business rule that depends on two fields at once (payment + discount).
9. A **realistic error flow**: if validation fails, the message is rejected in a controlled way (in production, this typically routes to a dead-letter queue) without crashing the worker.

## Conclusion

With both posts in this series, you now have a solid foundation - from the fundamentals all the way to a multi-layered data validation project (configuration, model composition, business rules, and integration with external formats), the way it would actually show up in a real production backend service.

## Links

- [Official Pydantic docs - Get Started](https://pydantic.dev/docs/validation/latest/get-started/)
- [Introdução ao Pydantic - Python Academy](https://pythonacademy.com.br/blog/introducao-ao-pydantic-validacao-e-modelagem-de-dados-simplificada-com-python)
- [Official docs - Validators](https://docs.pydantic.dev/latest/concepts/validators/)
- [Official docs - Pydantic Settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
- [Official GitHub repository](https://github.com/pydantic/pydantic)
- [Pydantic Part 1 - Fundamentals](/blog/pydantic-part-1-fundamentals)
