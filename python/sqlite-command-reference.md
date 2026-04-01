# SQLite CLI and Syntax Reference

## Table of Contents
1. [Essential Dot-Commands (CLI)](#1-essential-dot-commands-cli-only)
2. [Core SQL Data Manipulation (DML)](#2-core-sql-data-manipulation-dml)
3. [Data Definition Language (DDL)](#3-data-definition-language-ddl)
4. [Query Operations](#4-query-operations)
5. [Advanced SQL Commands](#5-advanced-sql-commands)
6. [Pragmas (Configuration)](#6-pragmas-configuration)
7. [Less Common but Useful Commands](#7-less-common-but-useful-commands)

---

## 1. Essential Dot-Commands (CLI Only)
| # | Command | Description | Usage | Notes |
| :---: | :--- | :--- | :--- | :--- |
| 1 | `.open` | Opens a database file | `.open my_data.db` | Creates file if it doesn't exist. |
| 2 | `.tables` | Lists all tables | `.tables` | Quick overview of database structure. |
| 3 | `.schema` | Shows table definitions| `.schema users` | Displays `CREATE` statements. Omit table name to see all. |
| 4 | `.mode` | Sets output format | `.mode column` | Options: csv, column, html, insert, line, list, markdown, table. |
| 5 | `.headers` | Toggles column names | `.headers on` | Use with `.mode column` or `table`. |
| 6 | `.quit` | Exits SQLite CLI | `.quit` | Also `.exit` works. |
| 7 | `.help` | Shows all dot commands | `.help` | Essential reference for CLI usage. |
| 8 | `.import` | Imports CSV to table | `.import file.csv users` | Requires `.separator ","` first for CSV. |
| 9 | `.output` | Redirects output to file | `.output results.txt` | Use `.output stdout` to reset. |
| 10 | `.read` | Executes SQL from file | `.read script.sql` | Useful for batch operations. |

---

## 2. Core SQL Data Manipulation (DML)
| # | Statement | Description | Usage Example | Notes |
| :---: | :--- | :--- | :--- | :--- |
| 11 | `SELECT` | Retrieves data from tables | `SELECT * FROM users;` | Most fundamental query command. |
| 12 | `INSERT INTO` | Adds new records | `INSERT INTO users (name, age) VALUES ('John', 25);` | Can insert multiple rows with comma separation. |
| 13 | `UPDATE` | Modifies existing records | `UPDATE users SET age = 26 WHERE name = 'John';` | Always use WHERE to avoid updating all rows. |
| 14 | `DELETE FROM` | Removes records | `DELETE FROM users WHERE id = 5;` | Without WHERE, deletes all records. |
| 15 | `WHERE` | Filters query results | `SELECT * FROM users WHERE age > 18;` | Essential for conditional operations. |
| 16 | `ORDER BY` | Sorts results | `SELECT * FROM users ORDER BY age DESC;` | Use ASC (default) or DESC. |
| 17 | `LIMIT` | Restricts number of results | `SELECT * FROM users LIMIT 10;` | Often used with ORDER BY. |
| 18 | `OFFSET` | Skips rows in results | `SELECT * FROM users LIMIT 10 OFFSET 20;` | Useful for pagination. |

---

## 3. Data Definition Language (DDL)
| # | Statement | Description | Usage Example | Notes |
| :---: | :--- | :--- | :--- | :--- |
| 19 | `CREATE TABLE` | Defines a new table | `CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT);` | Foundation of database structure. |
| 20 | `DROP TABLE` | Deletes a table | `DROP TABLE users;` | Permanently removes table and all data. |
| 21 | `ALTER TABLE` | Modifies table structure | `ALTER TABLE users ADD COLUMN email TEXT;` | SQLite has limited ALTER support. |
| 22 | `CREATE INDEX` | Creates an index | `CREATE INDEX idx_name ON users(name);` | Speeds up queries on indexed columns. |
| 23 | `DROP INDEX` | Removes an index | `DROP INDEX idx_name;` | Frees up space but slows queries. |
| 24 | `PRIMARY KEY` | Defines unique identifier | `id INTEGER PRIMARY KEY AUTOINCREMENT` | Enforces uniqueness and creates index. |
| 25 | `FOREIGN KEY` | Links tables together | `FOREIGN KEY (user_id) REFERENCES users(id)` | Requires `PRAGMA foreign_keys = ON;` |
| 26 | `UNIQUE` | Enforces unique values | `email TEXT UNIQUE` | Prevents duplicate entries. |

---

## 4. Query Operations
| # | Statement | Description | Usage Example | Notes |
| :---: | :--- | :--- | :--- | :--- |
| 27 | `COUNT()` | Counts rows | `SELECT COUNT(*) FROM users;` | Returns number of records. |
| 28 | `SUM()` | Adds numeric values | `SELECT SUM(price) FROM orders;` | Aggregate function for totals. |
| 29 | `AVG()` | Calculates average | `SELECT AVG(age) FROM users;` | Returns mean value. |
| 30 | `MIN()` / `MAX()` | Finds minimum/maximum | `SELECT MIN(age), MAX(age) FROM users;` | Useful for ranges. |
| 31 | `GROUP BY` | Groups rows by column | `SELECT city, COUNT(*) FROM users GROUP BY city;` | Used with aggregate functions. |
| 32 | `HAVING` | Filters grouped results | `SELECT city FROM users GROUP BY city HAVING COUNT(*) > 5;` | Like WHERE but for groups. |
| 33 | `INNER JOIN` | Combines matching rows | `SELECT * FROM orders JOIN users ON orders.user_id = users.id;` | Most common join type. |
| 34 | `LEFT JOIN` | Includes all left table rows | `SELECT * FROM users LEFT JOIN orders ON users.id = orders.user_id;` | Shows users even without orders. |
| 35 | `DISTINCT` | Removes duplicates | `SELECT DISTINCT city FROM users;` | Returns unique values only. |
| 36 | `LIKE` | Pattern matching | `SELECT * FROM users WHERE name LIKE 'J%';` | % matches any characters, _ matches one. |

---

## 5. Advanced SQL Commands
| # | Statement | Description | Usage Example | Notes |
| :---: | :--- | :--- | :--- | :--- |
| 37 | `UPSERT` | Insert or update on conflict | `INSERT INTO users (id, name) VALUES (1, 'John') ON CONFLICT(id) DO UPDATE SET name = excluded.name;` | SQLite 3.24.0+ feature. |
| 38 | `CASE` | Conditional expressions | `SELECT name, CASE WHEN age < 18 THEN 'Minor' ELSE 'Adult' END FROM users;` | Inline if-then-else logic. |
| 39 | `COALESCE()` | Returns first non-NULL | `SELECT COALESCE(phone, email, 'N/A') FROM users;` | Fallback values for NULL. |
| 40 | `CAST()` | Converts data types | `SELECT CAST(age AS TEXT) FROM users;` | Explicit type conversion. |
| 41 | `SUBSTR()` | Extracts substring | `SELECT SUBSTR(name, 1, 3) FROM users;` | String manipulation. |
| 42 | `REPLACE()` | Replaces text | `SELECT REPLACE(name, 'John', 'Jane') FROM users;` | String substitution. |

---

## 6. Pragmas (Configuration)
| # | Pragma | Description | Usage | Notes |
| :---: | :--- | :--- | :--- | :--- |
| 43 | `foreign_keys` | Enables FK constraints | `PRAGMA foreign_keys = ON;` | Off by default for backward compatibility. |
| 44 | `journal_mode` | Changes concurrency mode | `PRAGMA journal_mode = WAL;` | WAL improves concurrent read performance. |
| 45 | `synchronous` | Controls disk sync | `PRAGMA synchronous = NORMAL;` | FULL (safe), NORMAL (faster), OFF (risky). |
| 46 | `cache_size` | Sets page cache size | `PRAGMA cache_size = -64000;` | Negative = KB, positive = pages. |
| 47 | `table_info` | Shows column details | `PRAGMA table_info(users);` | Lists columns, types, and constraints. |

---

## 7. Less Common but Useful Commands
| # | Command | Description | Usage Example | Notes |
| :---: | :--- | :--- | :--- | :--- |
| 48 | `VACUUM` | Rebuilds the database file | `VACUUM;` | Reclaims space after deletions. |
| 49 | `EXPLAIN QUERY PLAN` | Shows query execution plan | `EXPLAIN QUERY PLAN SELECT * FROM users WHERE id = 1;` | Debug query performance. |
| 50 | `ATTACH DATABASE` | Connects another database | `ATTACH DATABASE 'other.db' AS other;` | Query multiple databases at once. |
| 51 | `DETACH DATABASE` | Disconnects attached DB | `DETACH DATABASE other;` | Closes attached database. |
| 52 | `.dump` | Exports database to SQL | `.dump users` | Creates SQL script to recreate data. |
| 53 | `.backup` | Backs up database | `.backup backup.db` | Creates binary copy of database. |
| 54 | `BEGIN TRANSACTION` | Starts a transaction | `BEGIN TRANSACTION;` | Groups operations atomically. |
| 55 | `COMMIT` | Saves transaction changes | `COMMIT;` | Makes transaction permanent. |
| 56 | `ROLLBACK` | Undoes transaction | `ROLLBACK;` | Reverts changes since BEGIN. |
| 57 | `CREATE VIEW` | Creates virtual table | `CREATE VIEW active_users AS SELECT * FROM users WHERE active = 1;` | Reusable query with table-like syntax. |

---

## Quick Reference Examples

### Creating a Complete Database
```sql
-- Create table with constraints
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    email TEXT UNIQUE NOT NULL,
    age INTEGER CHECK(age >= 18),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Insert data
INSERT INTO users (username, email, age) VALUES 
    ('john_doe', 'john@example.com', 25),
    ('jane_smith', 'jane@example.com', 30);

-- Query with filtering and sorting
SELECT username, age FROM users 
WHERE age > 20 
ORDER BY age DESC 
LIMIT 10;
```

### Working with Joins
```sql
-- Create related tables
CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,
    total REAL,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Join query
SELECT users.username, COUNT(orders.id) as order_count, SUM(orders.total) as total_spent
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.id
HAVING order_count > 0;
```

---

**Last Updated:** 2026-02-18  
**SQLite Version:** 3.x compatible  
**Total Commands:** 57
