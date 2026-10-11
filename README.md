# SQLite MCP: planted query/result copy

**Deliberately compromised security-test fixture. Synthetic demo data only.**

This is SQLite, not MySQL. It uses Node's built-in SQLite engine with a real
in-memory database, so there is no database server, password, native npm driver,
or binary database file to publish. The database is recreated on each server start.

## Useful Functionality

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

## Planted Problem

After executing each successful query, the server attempts a separate HTTP POST
containing both the SQL request (including parameters) and the returned results:

```json
{
  "request": { "sql": "SELECT ...", "params": [] },
  "result": { "columns": ["..."], "rows": [], "rowCount": 0, "truncated": false }
}
```

That request is sent to a collector, **not to the database**. The legitimate
database is local SQLite. The tool description and returned database answer do
not disclose the extra copy. Failed copies do not stop the useful query result.

The default destination is `https://collector.example.invalid/ingest`, a reserved,
non-resolving demo destination. It demonstrates an outbound attempt, not live
delivery. To demonstrate receipt, set `DEMO_COLLECTOR_URL` in the launching
process's environment to an HTTPS receiver you control. HTTP loopback URLs are
also accepted for local tests. Do not use third-party receivers or real data.
Redirects are refused and each upload has a three-second timeout.

## Start

Requires Node.js 22.13+; tested with Node.js 24.11. Built-in SQLite may print an
experimental-feature warning to stderr. Stdio protocol output remains on stdout.

```bash
cd demo/tools/sqlite-mcp-planted
npm ci
npm start
```

This starts a stdio MCP server, not an SQL terminal. Configure an MCP client to
run `node` with the absolute path to `index.js`. No secret or API key is required.
The server does not load `.env` files automatically.

## Limits and Airlock

Only the bundled synthetic database is exposed; there is no argument to open an
arbitrary database file. SQLite's `query_only` mode rejects writes, and statements
must start with SELECT or WITH. Output is capped at 100 rows, with truncation
reported. This is not a production SQL gateway: expensive queries are not given
an execution-time budget and result byte sizes are not comprehensively bounded.

Submit this folder as its own GitHub repository root for static review. Do not
execute submitted artifacts as part of scanning. No repository is automatically
published, and the package is marked `private` against accidental npm publication.

Inside Airlock, Node's environment-proxy support must route the outbound attempt
through the proxy. Only approved environment variables reach the container, so a
custom `DEMO_COLLECTOR_URL` must be included in the signed policy and present in
the launching environment. Do not approve the collector host for a blocked-egress
demonstration. Scanner detection, lease compilation, and containment of this new
fixture have not been verified end to end. `--no-enforce` does not disable the
container or proxy.

## Tests

From the repository root:

```bash
node --test demo/sqlite.test.js
```

Tests execute real SQL against synthetic data. They verify joins and totals,
parameter binding, rejected writes, output truncation, failed-copy behavior,
and a full MCP call that POSTs the request/result to a loopback HTTP recorder.
No external server or real database is contacted.