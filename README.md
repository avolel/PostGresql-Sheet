# PostgreSQL Cheat Sheet

A practical, single-file reference for **PostgreSQL** — the open-source relational database known for strong standards compliance, rich data types, and a powerful extension ecosystem. Built for developers who need a quick lookup while writing queries, building APIs, or tuning production systems.

## 📄 Contents

- **[postgresql-cheat-sheet.md](./postgresql-cheat-sheet.md)** — the full cheat sheet

## What's Covered

The cheat sheet is organized into 25 sections grouped by what you're actually trying to do:

### Fundamentals
- Data types (including JSON, arrays, ranges, and UUID)
- DDL — tables, schemas, constraints, partitioning
- DML — insert, update, delete, all with `RETURNING`
- `SELECT` essentials including `LIMIT`/`OFFSET`, `DISTINCT ON`, and regex matching

### Querying
- All join types (inner, outer, cross, self, `LATERAL`)
- Aggregations with `FILTER`, `ARRAY_AGG`, `STRING_AGG`, `JSONB_AGG`
- Subqueries and CTEs (recursive, data-modifying, materialized hints)
- Window functions — ranking, running totals, named windows

### Built-in Functions
- String manipulation (including regex)
- Date/time with timezone-aware operations and `generate_series`
- Numeric, conversion, and `NULL` handling
- Conditional logic (`CASE`, `COALESCE`, `NULLIF`, `GREATEST`/`LEAST`)

### Postgres-Specific Power Features
- **Arrays** — creation, slicing, containment operators, `UNNEST`
- **JSON & JSONB** — `->`, `->>`, `@>`, `JSONB_SET`, path queries, shredding to rows
- **UPSERT** — `ON CONFLICT DO UPDATE` and the newer `MERGE` (15+)
- **Full-text search** — `tsvector`, `tsquery`, `ts_rank`, generated columns

### Procedural Code
- SQL and PL/pgSQL functions with volatility hints
- Set-returning functions (TVFs)
- Procedures with embedded transactions (11+)
- Triggers, including the common `updated_at` pattern

### Operations
- Index types — B-tree, GIN, GiST, BRIN, Hash, partial, expression, covering
- Transactions, isolation levels, row-level locking, `SKIP LOCKED` for queues
- Error handling and SQLSTATE codes
- Roles, privileges, default privileges, and Row-Level Security
- `psql` meta-commands (`\dt`, `\d`, `\timing`, etc.)
- Performance tips and system catalogs for diagnostics

## Who It's For

- Developers coming from MySQL, SQL Server, or Oracle who need PostgreSQL syntax and idioms quickly
- Backend engineers writing queries, migrations, or stored functions against Postgres
- Anyone working with JSONB, arrays, or full-text search and wanting a syntax reference
- DBAs and SREs who want a consolidated reference to share with their dev team

## Compatibility

Examples target **PostgreSQL 12 and newer**, with version flags called out inline for newer features:

- `MATERIALIZED` / `NOT MATERIALIZED` CTE hints — 12+
- `GENERATED ALWAYS AS ... STORED` columns — 12+
- `MERGE` statement — 15+
- `COMMIT` / `ROLLBACK` inside procedures — 11+
- `INCLUDE` clause for covering indexes — 11+
- `GENERATED ... AS IDENTITY` (preferred over `SERIAL`) — 10+
- Declarative partitioning — 10+

Almost everything else works on any reasonably modern Postgres.

## How to Use

### View it
Open `postgresql-cheat-sheet.md` in any Markdown viewer — GitHub, VS Code, Obsidian, or your favorite Markdown reader will render it with a clickable table of contents.

### Search it
Because it's a single plain-text file, `Ctrl+F` / `Cmd+F` is your friend. Every section has a heading that matches common terminology.

### Convert it
If you prefer a different format:

```bash
# PDF (requires pandoc + a LaTeX engine)
pandoc postgresql-cheat-sheet.md -o postgresql-cheat-sheet.pdf

# HTML
pandoc postgresql-cheat-sheet.md -o postgresql-cheat-sheet.html --standalone --toc

# DOCX
pandoc postgresql-cheat-sheet.md -o postgresql-cheat-sheet.docx
```

### Run it
The examples are written so you can paste most of them straight into `psql`. They reference a fictional `sales` schema with `customers` and `orders` — you don't need to create them to read along, but if you want to actually run things:

```sql
CREATE SCHEMA sales;
CREATE TABLE sales.customers (
    customer_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    first_name  TEXT NOT NULL,
    last_name   TEXT NOT NULL,
    email       TEXT UNIQUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_active   BOOLEAN NOT NULL DEFAULT TRUE
);
```

## Conventions

- Code examples use a fictional `sales` schema with `customers` and `orders` tables.
- `snake_case` for all identifiers — the PostgreSQL community standard, and it avoids quoting headaches with case-sensitive identifiers.
- Comments (`--`) explain intent; `>` blockquotes call out tips and gotchas.
- Postgres-specific idioms (like `RETURNING`, `ON CONFLICT`, `LATERAL`) are highlighted because they're often what users from other databases miss most.

## What's *Not* Covered

By design, the cheat sheet skips:

- Replication setup (streaming, logical, publications/subscriptions)
- Backup and restore (`pg_dump`, `pg_basebackup`, PITR)
- Extensions beyond `pg_stat_statements` mentions (PostGIS, TimescaleDB, etc. deserve dedicated references)
- Foreign data wrappers and `dblink`
- Deep `postgresql.conf` tuning
- Logical decoding / change data capture

If you need any of these, it's worth a dedicated reference rather than a one-pager.

## Companion Reference

If you also work with Microsoft SQL Server, there's a [T-SQL cheat sheet](./tsql-cheat-sheet.md) in the same style.

## Contributing

If you spot an error or want to add a section, edits to the Markdown file are welcome. Keep the tone terse, favor runnable examples over prose, and call out version requirements inline.

## License

Use freely for personal reference, team wikis, or training materials.
