# SQLite MCP

A stdio MCP server for read-only queries on a sample shop database.
Uses Node's built-in SQLite engine with a real
in-memory database, so there is no database server, password, native npm driver,
or binary database file to publish. The database is recreated on each server start.

## Queries

The `query_database` MCP tool executes real read-only SQL and returns column
names, rows, a returned-row count, and a truncation flag. It supports joins,
aggregates, SELECT/WITH statements, and positional `?` parameters. It does not
use an LLM or invent database answers.

The demo database has these tables:

| Table | Columns | Synthetic Data |
|---|---|---|
| `customers` | `id`, `name`, `email` | Demo Alice and Demo Bob, with example.com addresses |
| `orders` | `id`, `customer_id`, `total_cents`, `status` | Two paid orders (12500 and 7500 cents), one pending order (3000 cents) |

Ask your MCP client: "What is the total value of paid orders?"

It can call `query_database` with:

```json
{
  "sql": "SELECT SUM(total_cents) AS total_cents FROM orders WHERE status = ?",
  "params": ["paid"]
}
```

SQLite returns `20000` cents, or $200. Claude can explain that result, but SQLite
does the actual calculation. For schema discovery, query `sqlite_master` or use
the schema included in the tool description.

## Start

Requires Node.js 22.13+; tested with Node.js 24.11. Built-in SQLite may print an
experimental-feature warning to stderr. Stdio protocol output remains on stdout.

```bash
npm ci
npm start
```

This starts a stdio MCP server, not an SQL terminal. Configure an MCP client to
run `node` with the absolute path to `index.js`. No secret or API key is required.
The server does not load `.env` files automatically.

## Limits

Only the bundled synthetic database is exposed; there is no argument to open an
arbitrary database file. SQLite's `query_only` mode rejects writes, and statements
must start with SELECT or WITH. Output is capped at 100 rows, with truncation
reported. This is not a production SQL gateway: expensive queries are not given
an execution-time budget and result byte sizes are not comprehensively bounded.
