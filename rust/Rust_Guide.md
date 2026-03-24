# Rust Guide: HTTP, JSON, HTML, CLI, and String Interpolation

Comprehensive reference covering built-in functions, types, idioms, data structures, and common third-party crates

---

## Table of Contents

1. Overview and conventions
2. Core language features and traits you’ll use daily
3. Common data structures and methods (Vec, String, slices, HashMap, Option, Result)
4. Iterators and common iterator adapters
5. Error handling idioms (Result, ? operator, thiserror, anyhow)
6. Async/await and executors (tokio, async-std) — minimal primer
7. JSON: serde and serde_json (derive, attributes, examples)
8. HTTP: reqwest, hyper, actix-web (client & server examples)
9. HTML: scraping and parsing (scraper crate, select), templating (askama, tera)
10. CLI: clap and structopt (derive usage, subcommands, arg parsing)
11. String interpolation and formatting: `format!`, `println!`, `write!`, formatting specifiers
12. Common third-party crates quick reference
13. Cheatsheet tables (methods, macros, traits) and examples

---

## 1. Overview and conventions

This guide mixes short code samples with line-by-line explanations. Rust is explicit about ownership, borrowing, and mutability. Where relevant, examples annotate ownership and lifetimes.

All examples use 2021 edition idioms. For async examples we use `tokio` where noted.

---

## 2. Core language features & traits

| Feature / Trait | Purpose | Typical methods / notes |
|---|---:|---|
| `Drop` | Cleanup when a value goes out of scope | implement `fn drop(&mut self)` |
| `Debug` | Printable for debugging with `{:?}` | Derive with `#[derive(Debug)]` |
| `Display` | User-facing formatting with `{}` | Implement `fmt::Display` |
| `From` / `Into` | Conversions | `impl From<T> for U` enables `Into` automatically |
| `AsRef` / `AsMut` | Borrow conversions (`&T`) | `AsRef<str>` commonly used |
| `Iterator` | Sequence iteration | `.next()`, `.map()`, `.filter()`, `.collect()` |

### Example: `Debug` vs `Display`

```rust
#[derive(Debug)]
struct Point { x: i32, y: i32 }

impl std::fmt::Display for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, \"({}, {})\", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!(\"Debug: {:?}\", p); // uses Debug
    println!(\"Display: {}\", p); // uses Display
}
```

Line-by-line: `#[derive(Debug)]` auto-implements `fmt::Debug`.
`impl Display` provides a user-facing format; `write!` writes formatted text into the `Formatter`.

---

## 3. Common data structures & methods

### `Vec<T>`

| Method | Description | Example |
|---|---|---|
| `push` | append element | `v.push(3)` |
| `pop` | remove last element | `v.pop()` returns `Option<T>` |
| `len` | length | `v.len()` |
| `iter` / `iter_mut` | iterate by reference | `for x in v.iter()` |
| `into_iter` | consumes vector, yields T | `let v: Vec<i32> = vec![1,2,3]; for x in v.into_iter()` |

```rust
let mut v = Vec::new(); // Vec::<i32>::new()
v.push(1);
v.push(2);
let last = v.pop(); // Option<i32>
```

### `String` and `&str`

Common methods: `push`, `push_str`, `len`, `as_str`, `replace`, `split`, `trim`.

```rust
let mut s = String::from(\"hello\");
s.push(',');
s.push_str(\" world\");
println!(\"{} ({} bytes)\", s, s.len());
let parts: Vec<&str> = s.split_whitespace().collect();
```

Notes: `String` owns heap-allocated string data; `&str` is a borrowed slice.

### `HashMap<K,V>`

```rust
use std::collections::HashMap;
let mut map = HashMap::new();
map.insert(\"apple\", 3);
if let Some(&count) = map.get(\"apple\") {
    println!(\"count = {}\", count);
}
```

Useful methods: `entry(key).or_insert(value)` to mutate or insert in place.

---

## 4. Iterators & adapters

Common adapter chain: `iter().map(...).filter(...).collect()`

```rust
let v = vec![1,2,3,4,5];
let evens: Vec<i32> = v.into_iter().filter(|x| x%2==0).collect();
```

Important iterator methods: `map`, `filter`, `fold`, `for_each`, `enumerate`, `zip`, `take`, `skip`.

Example `fold` line-by-line:

```rust
let sum = (1..=5).fold(0, |acc, x| acc + x);
// fold(start, |accumulator, element| new_acc)
```

---

## 5. Error handling idioms

### `Result<T, E>` and `?` operator

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_file(path: &str) -> Result<String, io::Error> {
    let mut s = String::new();
    let mut f = File::open(path)?; // propagates error early
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

`?` returns early with the error if the `Result` is `Err`.

### `thiserror` and `anyhow`

- `thiserror` is for defining error enums with `Display` implemented.
- `anyhow` is for application-level error handling with context; good for CLI.

Example `thiserror` usage:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error(\"IO error: {0}\")]
    Io(#[from] std::io::Error),

    #[error(\"Parse error: {0}\")]
    Parse(#[from] std::num::ParseIntError),
}
```

---

## 6. Async/Await primer

Crates: `tokio`, `async-std`, `futures`.

Example `tokio` async main and reqwest client usage:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let res = reqwest::get(\"https://httpbin.org/get\").await?;
    let body = res.text().await?;
    println!(\"len = {}\", body.len());
    Ok(())
}
```

Line notes: `#[tokio::main]` sets up tokio runtime. `reqwest::get` returns a `Future`; `.await` yields the `Response`.

---

## 7. JSON: serde / serde_json

Add to `Cargo.toml`:

```toml
serde = { version = \"1.0\", features = [\"derive\"] }
serde_json = \"1.0\"
```

Example: derive `Serialize`/`Deserialize` and round-trip JSON

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct User {
    id: u64,
    username: String,
    active: bool,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let u = User { id: 1, username: \"alice\".into(), active: true };
    let json = serde_json::to_string(&u)?; // serialize
    println!(\"json = {}\", json);
    let parsed: User = serde_json::from_str(&json)?; // deserialize
    println!(\"parsed = {:?}\", parsed);
    Ok(())
}
```

#### Common serde attributes

- `#[serde(rename = \"fieldname\")]` rename a field
- `#[serde(skip_serializing_if = \"Option::is_none\")]` skip None
- `#[serde(default)]` provide default value if missing

Example with attributes:

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Config {
    #[serde(rename = \"timeout_ms\")]
    timeout: Option<u64>,

    #[serde(default = \"default_retries\")]
    retries: u8,
}

fn default_retries() -> u8 { 3 }
```

---

## 8. HTTP

### Reqwest (high-level HTTP client)

Add:

```toml
reqwest = { version = \"0.11\", features = [\"json\", \"blocking\", \"tls\"] }
```

#### Async example

```rust
// async reqwest sample
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = reqwest::Client::new();
    let res = client
        .get(\"https://httpbin.org/get\")
        .query(&[(\"key\", \"value\")])
        .send()
        .await?;

    let status = res.status();
    let headers = res.headers().clone();
    let body: serde_json::Value = res.json().await?; // uses serde_json
    println!(\"status: {} body keys: {}\", status, body.as_object().map(|o| o.len()).unwrap_or(0));
    Ok(())
}
```

Line notes: `Client` is reusable and should be reused for efficiency. `.json().await?` deserializes response body to `serde_json::Value` or a typed struct.

#### Blocking example

```rust
let res = reqwest::blocking::get(\"https://httpbin.org/get\")?;
let text = res.text()?;
```

### Hyper (low-level, high performance)

Hyper is lower-level; common when building frameworks or for maximal control. Use `hyper` if building your own server or client with fine-grained control. Example omitted for brevity; prefer `reqwest` for typical client tasks.

### actix-web (server)

Add to `Cargo.toml`:

```toml
actix-web = \"4\"
serde = { version = \"1.0\", features = [\"derive\"] }
```

Minimal server example:

```rust
use actix_web::{get, web, App, HttpResponse, HttpServer, Responder};

#[get(\"/hello\")]
async fn hello() -> impl Responder {
    HttpResponse::Ok().json(serde_json::json!({\"message\":\"hello\"}))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(hello))
        .bind((\"127.0.0.1\", 8080))?
        .run()
        .await
}
```

Line notes: `#[get(\"/hello\")]` is route macro; handler returns a type that implements `Responder`.

---

## 9. HTML: parsing, scraping, templating

### `scraper` crate (parsing and scraping with selectors)

Add:

```toml
scraper = \"0.14\"
reqwest = { version = \"0.11\", features = [\"blocking\"] }
```

Example scraping titles:

```rust
use scraper::{Html, Selector};
let html = reqwest::blocking::get(\"https://www.rust-lang.org\")?.text()?;
let document = Html::parse_document(&html);
let selector = Selector::parse(\"h1\").unwrap();
for element in document.select(&selector) {
    println!(\"h1: {}\", element.text().collect::<Vec<_>>().join(\" \"));
}
```

### Templating: `tera` and `askama`

- `tera` uses template files (`.tera`) and runtime rendering
- `askama` uses Rust macros to compile templates into typesafe code

`askama` example (derive based):

```rust
use askama::Template;

#[derive(Template)]
#[template(path = \"hello.html\")]
struct HelloTemplate<'a> { name: &'a str }

// All templates are compiled into Rust code at build time.
```

---

## 10. CLI: `clap` (and `structopt` historical)

Add:

```toml
clap = { version = \"4\", features = [\"derive\"] }
```

Example with derive and subcommand:

```rust
use clap::{Parser, Subcommand};

#[derive(Parser, Debug)]
#[command(name = \"myapp\")]
struct Cli {
    #[arg(short, long, default_value_t = 1)]
    concurrency: u8,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand, Debug)]
enum Commands { 
    Run { path: String },
    Test { verbose: bool }
}

fn main() {
    let cli = Cli::parse();
    println!(\"cli: {:?}\", cli);
}
```

Line notes: `Parser::parse()` reads `std::env::args()` and constructs typed struct.

---

## 11. String interpolation & formatting

Rust uses macros for formatting: `format!`, `println!`, `write!`, and `format_args!`.

| Macro | Purpose |
|---|---|
| `format!` | Returns a `String` with formatted content |
| `println!` | Prints to stdout with newline |
| `print!` | Prints without newline |
| `write!` | Writes into a `fmt::Write` or `std::io::Write` buffer |

Formatting specifiers: `{:?}` (Debug), `{}` (Display), `{:.2}` (precision), `{:>8}` (width, alignment), `{:x}` (hex), `{:b}` (binary)

Example with explanations:

```rust
let pi = 3.14159265;
println!(\"Pi ~ {:.2}\", pi); // prints 3.14 (2 decimal places)
let s = format!(\"Hello, {}\", \"world\"); // returns String
let value = 255;
println!(\"hex: {:#x}\", value); // 0xff
```

`format_args!` is the underlying mechanism used by formatting macros; use it when writing custom wrappers.

---

## 12. Common third-party crates (quick reference)

- `serde`, `serde_json` — JSON (de)serialization
- `reqwest` — HTTP client (higher-level)
- `hyper` — HTTP low-level client/server
- `actix-web`, `axum`, `warp` — web frameworks (server)
- `tokio` — async executor and ecosystem
- `anyhow`, `thiserror` — error handling utilities
- `clap` — CLI parsing
- `structopt` — deprecated in favor of `clap` derive
- `tera`, `askama` — templating
- `scraper` — HTML scraping
- `select` — alternative scraping crate
- `regex` — regular expressions
- `lazy_static` / `once_cell` — singletons & static init

---

## 13. Cheatsheet tables & examples

### Option & Result methods (selected)

| Type | Method | Returns / Notes |
|---|---|---|
| `Option<T>` | `map` | transforms `Some(t)` to `Some(f(t))` |
| `Option<T>` | `and_then` / `flat_map` | chain fallible computations |
| `Result<T,E>` | `map_err` | transform error |
| `Result<T,E>` | `unwrap_or`, `unwrap_or_else` | fallback values |

### Common String & slice methods

| Type | Method | Notes |
|---|---|---|
| `str` | `split` | iterator of &str |
| `str` | `replace` | returns a new `String` |
| `String` | `push_str` | append slice |
| `String` | `into_bytes` | consume string into bytes |

### Example: HTTP JSON CLI flow (putting pieces together)

A small program outline: perform HTTP GET, parse JSON response into typed struct, accept CLI args for URL.

```rust
use clap::Parser;
use serde::Deserialize;

#[derive(Parser)]
struct Cli { url: String }

#[derive(Deserialize, Debug)]
struct ApiResp { id: u64, name: String }

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cli = Cli::parse();
    let res = reqwest::get(&cli.url).await?.json::<ApiResp>().await?;
    println!(\"got: {:?}\", res);
    Ok(())
}
```

Line-by-line:
- `Cli::parse()` builds CLI from command-line args.
- `reqwest::get(&cli.url).await?` performs GET and returns `Response`.
- `.json::<ApiResp>().await?` deserializes body into `ApiResp` using `serde`.

---

## Appendix: Recommended learning progression

1. Master ownership/borrowing and `Result`/`Option` handling.
2. Learn `iterators` and `lifetimes` patterns.
3. Learn `serde` and `reqwest` for practical I/O.
4. Learn `tokio` and async patterns when building networked services.
5. Use `clap` for CLI apps; adopt `thiserror`/`anyhow` for errors.

---

## License

This document is provided under MIT-style terms for reuse and adaptation.

---

*End of guide.*
