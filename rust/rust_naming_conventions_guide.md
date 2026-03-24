# Rust Project Naming Conventions and Tutorial Repository Guide

This document consolidates **Rust naming conventions**, **Cargo project structure rules**, and a **complete sample tutorial repository layout**. It is intended as a practical reference for building professional, idiomatic Rust projects—especially multi-program tutorials and workspaces.

---

## 1. Naming Conventions (Authoritative Reference)

### 1.1 Cargo Package / Crate Names

**Convention:** `kebab-case`

```toml
[package]
name = "http-client"
```

**Rules**
- Lowercase only
- Hyphen-separated
- No underscores or spaces

**Why**
- crates.io standard
- Maps cleanly to binary names
- Cargo internally converts `-` → `_` for imports

```rust
use http_client::Client;
```

---

### 1.2 Workspace Names

**Convention:** directory name, usually `kebab-case`

```text
rust-networking-tutorial/
```

Workspaces are filesystem-level constructs; consistency matters more than strict enforcement.

---

## 2. Binary Program Naming

### 2.1 Default Binary

```text
src/main.rs
```

Binary name defaults to the package name.

---

### 2.2 Additional Binaries (`src/bin/`)

**Convention:** `snake_case.rs`

```text
src/bin/
├── tcp_server.rs
├── tcp_client.rs
└── udp_probe.rs
```

Run with:

```bash
cargo run --bin tcp_server
```

Binary names exposed to users may be `kebab-case` via `[[bin]]` if desired.

---

## 3. Example Programs (`examples/`)

**Convention:** `snake_case.rs`

```text
examples/
├── ownership.rs
├── lifetimes.rs
└── trait_objects.rs
```

Run with:

```bash
cargo run --example trait_objects
```

Use examples for:
- Small demonstrations
- Concept-focused programs
- Non-production binaries

---

## 4. Source Files and Modules

### 4.1 Module Files

**Convention:** `snake_case.rs`

```text
src/
├── lib.rs
├── main.rs
├── parser.rs
├── config.rs
└── net/
    ├── tcp.rs
    └── udp.rs
```

Prefer `net.rs` + submodules over `net/mod.rs` in modern Rust.

---

## 5. Library Crates

Cargo package name:

```toml
name = "shared-utils"
```

Import name:

```rust
use shared_utils::helpers;
```

---

## 6. Types and Symbols

| Item | Convention |
|----|----|
| Structs | PascalCase |
| Enums | PascalCase |
| Traits | PascalCase |
| Functions | snake_case |
| Variables | snake_case |
| Constants | SCREAMING_SNAKE_CASE |
| Statics | SCREAMING_SNAKE_CASE |

Examples:

```rust
struct HttpClient;
trait Serializable;
const MAX_RETRIES: usize = 5;
```

---

## 7. Feature Flags

**Convention:** `kebab-case`

```toml
[features]
async-runtime = []
tls-support = []
```

---

## 8. Environment Variables

**Convention:** `SCREAMING_SNAKE_CASE`

```bash
export RUST_LOG=debug
export APP_CONFIG_PATH=/etc/app/config.toml
```

---

## 9. Tests and Benchmarks

### Unit Tests

```rust
#[test]
fn parses_valid_header() {}
```

### Integration Tests

```text
tests/
├── api.rs
├── cli.rs
└── networking.rs
```

### Benchmarks

```text
benches/
└── json_parse.rs
```

---

## 10. Directory Naming

**Convention:** `kebab-case` or `snake_case` (choose one and stay consistent)

```text
chapter-01/
shared-utils/
```

Avoid:
- CamelCase
- Spaces
- Ambiguous numbers

---

## 11. Tutorial-Specific Patterns

### Chapters

```text
01-basics/
02-ownership/
03-lifetimes/
```

Numbered prefixes preserve ordering across filesystems.

### Non-Crate Material

```text
snippets/
drafts/
pseudocode/
```

These are intentionally not Cargo crates.

---

## 12. Anti-Patterns to Avoid

- `Main.rs`
- `MyProgram.rs`
- `example1.rs`
- `utils2.rs`
- `final_final.rs`

These degrade tooling support and maintainability.

---

## 13. Complete Sample Tutorial Repository (Best Practice)

```text
rust-systems-tutorial/
├── Cargo.toml                # workspace root
├── README.md
├── shared-utils/
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── io.rs
│       └── net.rs
├── 01-basics/
│   ├── Cargo.toml
│   └── src/main.rs
├── 02-ownership/
│   ├── Cargo.toml
│   └── src/main.rs
├── 03-lifetimes/
│   ├── Cargo.toml
│   └── src/main.rs
├── misc-examples/
│   ├── Cargo.toml
│   └── examples/
│       ├── borrowing.rs
│       ├── traits.rs
│       └── async_intro.rs
├── snippets/
│   └── partial_code.rs
└── diagrams/
```

### Workspace `Cargo.toml`

```toml
[workspace]
members = [
    "shared-utils",
    "01-basics",
    "02-ownership",
    "03-lifetimes",
    "misc-examples"
]
resolver = "2"
```

### Shared Dependency Usage

```toml
[dependencies]
shared-utils = { path = "../shared-utils" }
```

---

## 14. Operational Rule (Memorable)

> If Cargo names it → **kebab-case**  
> If Rust compiles it → **snake_case**  
> If it’s a type → **PascalCase**

This structure mirrors production Rust repositories and scales cleanly as tutorials grow.

