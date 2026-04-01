

A practical, production-oriented reference covering frequently used Python standard library modules and a foundational guide to building RESTful APIs with Flask.

---

# 📘 Part I: Python Standard Library – Frequently Used Modules

## 🔎 Quick Reference Table

|Module|Primary Purpose|Common Use Cases|Key Functions / Classes|Example|
|---|---|---|---|---|
|`os`|Operating system interaction|File and directory management, environment variables|`getcwd()`, `listdir()`, `remove()`, `mkdir()`, `environ`|`os.listdir('.')`|
|`sys`|Python runtime interaction|CLI arguments, interpreter info, exiting programs|`argv`, `exit()`, `path`, `version`|`sys.argv`|
|`math`|Mathematical operations|Scientific math, rounding, constants|`sqrt()`, `ceil()`, `floor()`, `pi`, `sin()`|`math.sqrt(16)`|
|`datetime`|Date and time handling|Timestamps, scheduling, formatting dates|`datetime`, `date`, `time`, `timedelta`, `strftime()`|`datetime.now()`|
|`json`|JSON encoding and decoding|APIs, configuration files, data exchange|`dump()`, `dumps()`, `load()`, `loads()`|`json.loads(data)`|
|`random`|Random value generation|Simulations, sampling, shuffling|`random()`, `randint()`, `choice()`, `shuffle()`|`random.randint(1, 10)`|
|`re`|Regular expressions|Pattern matching, validation, parsing|`search()`, `match()`, `findall()`, `sub()`|`re.findall(pattern, text)`|
|`collections`|Specialized data structures|Counting, queues, structured dictionaries|`Counter`, `deque`, `defaultdict`, `namedtuple`|`Counter(list)`|
|`shutil`|High-level file operations|Copying and moving files, directory trees|`copy()`, `copytree()`, `move()`, `rmtree()`|`shutil.copy(src, dst)`|
|`argparse`|Command-line parsing|CLI tools, automation scripts|`ArgumentParser`, `add_argument()`, `parse_args()`|`parser.parse_args()`|

---

# 🧠 Practical Usage Guide

## 📁 `os` – Operating System Interface

### When to Use

- Managing files and directories
- Reading environment variables
- Handling cross-platform file system operations

```python
import os

print(os.getcwd())
os.mkdir("new_folder")
print(os.environ.get("HOME"))
```

---

## ⚙️ `sys` – Runtime System Control

### When to Use

- Reading command-line arguments
- Exiting programs safely
- Inspecting interpreter configuration

```python
import sys

print(sys.argv)
sys.exit(0)
```

---

## 📐 `math` – Mathematical Functions

### When to Use

- Scientific computing
- Geometric calculations
- Rounding and working with constants

```python
import math

print(math.pi)
print(math.sqrt(25))
```

---

## 🕒 `datetime` – Date and Time Handling

### When to Use

- Logging and auditing
- Scheduling operations
- Performing time-based calculations

```python
from datetime import datetime, timedelta

now = datetime.now()
print(now.strftime("%Y-%m-%d"))
print(now + timedelta(days=7))
```

---

## 🔄 `json` – JSON Data Handling

### When to Use

- Working with APIs
- Storing configuration files
- Serializing structured data

```python
import json

data = {"name": "Alice", "age": 30}
json_string = json.dumps(data)
print(json.loads(json_string))
```

---

## 🎲 `random` – Random Number Generation

### When to Use

- Simulations and modeling
- Sampling datasets
- Randomized selections

```python
import random

print(random.randint(1, 100))
items = ["a", "b", "c"]
print(random.choice(items))
```

---

## 🔍 `re` – Regular Expressions

### When to Use

- Input validation
- Log parsing
- Text extraction and transformation

```
import re

text = "Email: test@example.com"
pattern = r"\S+@\S+"
print(re.findall(pattern, text))
```

---

## 📦 `collections` – Advanced Containers

### When to Use

- Frequency counting
- Efficient queue operations
- Structured dictionary defaults

```
from collections import Counter, deque

print(Counter([1, 1, 2, 3]))
queue = deque([1, 2, 3])
queue.appendleft(0)
```

---

## 📂 `shutil` – High-Level File Operations

### When to Use

- Copying entire directory trees
- Moving files
- Removing directories safely

```
import shutil

shutil.copy("file.txt", "backup.txt")
```

---

## 🖥️ `argparse` – Command-Line Applications

### When to Use

- Building CLI tools
- Writing automation scripts
- Creating production-ready command-line utilities

```
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--name")
args = parser.parse_args()

print(f"Hello {args.name}")
```

---

## 🚀 Strategic Usage Patterns

- Use `argparse`, `os`, and `shutil` together to build automation tooling.
- Combine `json` and `datetime` for structured logging systems.
- Pair `collections.Counter` with `re` for log analytics.
- Use `sys` to make scripts robust and production-ready for CLI deployment.

This section serves as a compact operational reference for scripting, automation, backend services, and infrastructure tooling.