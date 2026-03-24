

> A structured, production-focused reference for developers moving beyond beginner level.

----------

# 📚 Table of Contents

1.  Control Flow
2.  Functions
3.  Argument Patterns (`*args`, `**kwargs`)
4.  Type Hints & Typing
5.  Classes & OOP
6.  Methods
7.  Decorators
8.  Generators & Iterators
9.  Comprehensions
10.  Context Managers
11.  Data Classes
12.  Enums
13.  Error Handling
14.  Async Programming
15.  Meta Classes

----------

# 1️⃣ Control Flow

## Conditionals

```python
x = 42

if x > 50:
    print("High")
elif x > 10:
    print("Medium")
else:
    print("Low")

```

## Ternary Expression

```python
result = "even" if x % 2 == 0 else "odd"

```

## Match Statement (Python 3.10+)

```python
def check_status(code):
    match code:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case _:
            return "Unknown"

```

## Loop Control

```python
for i in range(10):
    if i == 3:
        continue
    if i == 8:
        break
    print(i)

```

----------

# 2️⃣ Functions

## Basic Function

```python
def add(a, b):
    return a + b

```

## Default Parameters

```python
def connect(host="localhost", port=5432):
    return f"{host}:{port}"

```

## Keyword-Only Parameters

```python
def create_user(name, *, role="user"):
    return {"name": name, "role": role}

```

## Positional-Only Parameters (Python 3.8+)

```python
def divide(a, b, /):
    return a / b

```

## Lambda Functions

```python
square = lambda x: x ** 2

```

----------

# 3️⃣ Argument Patterns

## *args

```python
def sum_all(*args):
    return sum(args)

```

## **kwargs

```python
def print_config(**kwargs):
    for key, value in kwargs.items():
        print(key, value)

```

## Combined Usage

```python
def flexible_function(a, *args, option=True, **kwargs):
    print(a)
    print(args)
    print(option)
    print(kwargs)

```

----------

# 4️⃣ Type Hints & Typing

```python
from typing import List, Dict, Optional, Union


def process(data: List[int]) -> Dict[str, int]:
    return {"count": len(data)}


def maybe_get(value: int) -> Optional[int]:
    return value if value > 0 else None


def stringify(value: Union[int, float]) -> str:
    return str(value)

```

## Generic Types

```python
from typing import TypeVar, Generic

T = TypeVar("T")

class Box(Generic[T]):
    def __init__(self, value: T):
        self.value = value

```

----------

# 5️⃣ Classes & OOP

```python
class User:
    def __init__(self, name: str):
        self.name = name

    def greet(self) -> str:
        return f"Hello {self.name}"

```

## Inheritance

```python
class Admin(User):
    def delete_user(self, user: User):
        print(f"Deleted {user.name}")

```

----------

# 6️⃣ Methods

## Instance Method

```python
class Example:
    def instance_method(self):
        return "instance"

```

## Class Method

```python
class Example:
    @classmethod
    def class_method(cls):
        return cls

```

## Static Method

```python
class Example:
    @staticmethod
    def static_method():
        return "static"

```

----------

# 7️⃣ Decorators

## Function Decorator

```python
def logger(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@logger
def greet(name):
    return f"Hello {name}"

```

## Decorator with Arguments

```python
def repeat(times):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for _ in range(times):
                func(*args, **kwargs)
        return wrapper
    return decorator

```

----------

# 8️⃣ Generators & Iterators

## Generator Function

```python
def count_up_to(n):
    for i in range(n):
        yield i

```

## Generator Expression

```python
squares = (x * x for x in range(10))

```

## Custom Iterator

```python
class Counter:
    def __init__(self, max_value):
        self.max_value = max_value
        self.current = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.current >= self.max_value:
            raise StopIteration
        self.current += 1
        return self.current

```

----------

# 9️⃣ Comprehensions

```python
squares = [x**2 for x in range(10)]

even_set = {x for x in range(10) if x % 2 == 0}

mapping = {x: x**2 for x in range(5)}

```

----------

# 🔟 Context Managers

## Using with

```python
with open("file.txt") as f:
    content = f.read()

```

## Custom Context Manager

```python
class ManagedResource:
    def __enter__(self):
        print("Acquire")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Release")

```

----------

# 1️⃣1️⃣ Dataclasses

```python
from dataclasses import dataclass

@dataclass
class Product:
    name: str
    price: float

```

----------

# 1️⃣2️⃣ Enums

```python
from enum import Enum

class Status(Enum):
    SUCCESS = 1
    FAILURE = 0

```

----------

# 1️⃣3️⃣ Error Handling

```python
class CustomError(Exception):
    pass

try:
    raise CustomError("Something went wrong")
except CustomError as e:
    print(e)
finally:
    print("Cleanup")

```

----------

# 1️⃣4️⃣ Async Programming

```python
import asyncio

async def fetch_data():
    await asyncio.sleep(1)
    return "Done"

async def main():
    result = await fetch_data()
    print(result)

asyncio.run(main())

```

----------

# 1️⃣5️⃣ Metaclasses (Advanced)

```python
class Meta(type):
    def __new__(cls, name, bases, dct):
        dct["added_attribute"] = 42
        return super().__new__(cls, name, bases, dct)

class MyClass(metaclass=Meta):
    pass

print(MyClass.added_attribute)

```


# 🏗 Engineering Patterns

## 1️⃣ Service Layer Pattern

Separate business logic from delivery layer (API/UI).

```python
class UserService:
    def __init__(self, repository):
        self.repository = repository

    def create_user(self, name: str):
        user = {"name": name}
        return self.repository.save(user)

```

----------

## 2️⃣ Repository Pattern

Abstract persistence layer.

```python
class UserRepository:
    def __init__(self):
        self.storage = []

    def save(self, user):
        self.storage.append(user)
        return user

```

----------

## 3️⃣ Dependency Injection

Inject dependencies instead of hardcoding.

```python
repo = UserRepository()
service = UserService(repo)

```

Benefits:

-   Easier testing
-   Replaceable infrastructure
-   Cleaner architecture

----------

## 4️⃣ Factory Pattern

Encapsulate object creation.

```python
class DatabaseFactory:
    @staticmethod
    def create(db_type: str):
        if db_type == "sqlite":
            return "SQLite Connection"
        if db_type == "postgres":
            return "Postgres Connection"
        raise ValueError("Unknown DB type")

```

----------

## 5️⃣ Strategy Pattern

Swap algorithms dynamically.

```python
class PaymentStrategy:
    def pay(self, amount):
        raise NotImplementedError

class CreditCardPayment(PaymentStrategy):
    def pay(self, amount):
        return f"Paid {amount} with credit card"

```

----------

## 6️⃣ Middleware Pattern (Common in APIs)

```python
def middleware(handler):
    def wrapper(request):
        print("Before request")
        response = handler(request)
        print("After request")
        return response
    return wrapper

```

----------

## 7️⃣ Configuration Management Pattern

Use environment-driven configuration.

```python
import os

class Config:
    DEBUG = os.getenv("DEBUG", "False") == "True"

```

----------

# 🧠 Production-Level Guidance

### Structure Large Python Projects Like This:

```
project/
 ├── domain/
 ├── services/
 ├── repositories/
 ├── api/
 ├── config/
 └── main.py

```

----------

## Principles to Apply

-   Keep business logic out of controllers
-   Prefer composition over inheritance
-   Type everything in production code
-   Make side effects explicit
-   Keep functions pure where possible
-   Use async only when I/O bound    

----------

# 🚀 End of Guide

This document now includes:

-   Intermediate & Advanced Python features
-   Printable cheat sheet tables
-   Real-world architecture patterns
-   Production engineering guidance

Ready for professional backend and systems-level development.

