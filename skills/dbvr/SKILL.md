---
name: dbvr
description: >-
  Drive dbvr, DBeaver's headless command-line client, to run database work from
  the terminal — execute SQL, dump schema/DDL, list databases/schemas/tables,
  and manage connections (datasources), projects, and drivers. Use this skill
  WHENEVER the task involves querying or inspecting a database through dbvr or a
  saved DBeaver connection, even if the user doesn't name dbvr: "run this SQL
  against my Postgres database", "query the postgres datasource", "what tables
  are in <db>", "dump the DDL / schema for <table>", "export this query to CSV",
  "list my DBeaver connections", "connect to <host> and run a SELECT", "headless
  DB query from the CLI". Also trigger on any mention of "dbvr", "DBeaver CLI",
  "DBeaver headless", or a request to run SQL against a database that is reachable
  through an existing DBeaver datasource. Do NOT use this for DBeaver desktop-GUI
  how-to questions unrelated to the CLI, or for writing SQL that will never be
  executed.
---

# dbvr — DBeaver headless CLI

`dbvr` runs database operations from the terminal against the same connections you
have in DBeaver (relational, NoSQL, cloud). Use it to execute SQL, browse
metadata, extract DDL, and manage connections without opening the GUI.

On this machine it is available as the `dbvr` command (aliased to the app binary).
Verify with `dbvr --version` (expect `dbvr 26.x`). Commands start a short-lived JVM,
so each invocation takes a couple of seconds — that's normal, not a hang. Do not
wrap calls in a `timeout` (macOS lacks it by default).

## The one thing to get right: how to connect

Almost every command needs to know **which database**. There are two ways, and
mixing up their flags is the most common failure:

**1. A saved DBeaver datasource (preferred when one exists).** Reference it by name
or ID with `-ds`:

```
dbvr sql -ds="My Database" "SELECT 1"
dbvr datasource list        # see available datasources (name + id + driver)
```

Names with spaces must be quoted. `dbvr datasource list` is the fastest way to
discover what's already configured — start there when the user names a connection.

**2. An ad-hoc / ephemeral connection** built from driver + host + credentials.
Requires `--driver` (get valid ids from `dbvr driver list`). Add `--stateless` so it
isn't persisted into the workspace:

```
dbvr sql --stateless --driver postgres-jdbc --host db.example.com --port 5432 \
  --database appdb -u readonly -p "$PGPASSWORD" "SELECT now()"
```

You can also pass a full JDBC URL with `--url=...` instead of host/port/database, and
supply auth-model-specific properties with `-auth name=value` (see
`dbvr auth-models` for models like `sqlserver_ad_password`, `google_bigquery`).

> Gotcha: `--database` is the *connection* database for ad-hoc mode. The `meta`
> commands take the catalog with `-db`/`--database-name` instead. Passing
> `--database` to a `meta ... -ds=...` call flips it into ad-hoc mode and it will
> demand `--driver`. When you already have `-ds`, use `-db`, `-sn`, `-tn`, `-fn`.

## Running SQL — `dbvr sql`

The query can be a positional argument, piped on stdin, or read from a file with
`-in=script.sql`:

```
dbvr sql -ds="My Database" "SELECT id, name FROM accounts WHERE active = 1"
dbvr sql -ds="My Database" -in=report.sql
echo "SELECT count(*) FROM orders" | dbvr sql -ds="My Database"
```

Key flags (full list in `references/command-reference.md`):

- `-format=csv|xml|json|txt` — output format. Default `txt` renders a Markdown pipe
  table, which is ideal for reading results back to the user. Use `csv`/`json` when
  the user wants machine-readable output or a file.
- `-out=FILE` — write results to a file instead of stdout (pair with `-format`).
  **Use an absolute path.** A relative `-out=report.csv` exits 0 but can silently
  write *no file* (it doesn't resolve against the shell's working directory), so
  always pass a full path like `-out="$PWD/report.csv"` and confirm the file exists
  afterward.
- `-l "[offset,]limit"` — row limit. **Default is 1000**, so large tables are
  silently truncated; set `-l` explicitly (e.g. `-l 50` to sample, `-l 0` behavior
  varies by build — prefer a real number) and mention when you've capped results.
- `--disable-status` — suppress the trailing execution-status line; cleaner output
  when you're going to parse or display the table.
- `--print-queries` — echo each statement before running it (useful for multi-
  statement scripts).
- `-op prop=value,prop2=value2` — exporter options for the chosen format.
- `-u` / `-p` — override the datasource's user/password for this run.

Multiple statements run in sequence; use `-in` for real scripts.

## Reading a query result

`txt` output is a Markdown table:

```
|ping|
|----|
|1   |
```

Read it back to the user as-is for small results, or summarize for large ones.
Prefer `-format=csv -out=file.csv` when they ask to export or the result is wide.

## Exploring structure — `dbvr meta`

Use `meta` instead of hand-writing `information_schema` queries — it works uniformly
across drivers.

```
dbvr meta database list -ds="My Database"                  # catalogs/databases
dbvr meta schema   list -ds="My Database" -db appdb
dbvr meta table    list -ds="My Database" -db appdb
dbvr meta table    ddl  -ds="My Database" -fn appdb.orders
dbvr meta schema   ddl  -ds="My Database" -db appdb -sn public
```

Identify the object either by parts (`-db` catalog, `-sn` schema, `-tn` table) or by
fully-qualified name (`-fn database.schema.table`). MySQL-family drivers treat the
database as the catalog, so `meta schema`/`meta table` need `-db`. Add `--full` to
`meta table ddl` for a more complete definition (indexes, constraints).

`meta` output is a plain newline-delimited list (or raw DDL) — there is **no**
`-format`/`-out` on `meta`. So when the user wants the *result of a listing* as CSV
or JSON, or written to a file, don't hand-assemble it from the plain list; get it
from `dbvr sql` instead, which does have `-format`/`-out`. For example, to export the
database list as CSV, query the catalog and format it:
`dbvr sql -ds="My Database" -format=csv -out="$PWD/databases.csv" "SHOW DATABASES"`
(or `SELECT schema_name FROM information_schema.schemata`).

## Managing connections, projects, drivers

- `dbvr datasource list|view|create|update|delete|move` — manage saved connections.
  Create needs `--driver` plus host/creds or `--url`, and `--name` to label it;
  `--save-password` and `--folder` are optional. Confirm with the user before
  `create`/`update`/`delete` since these mutate their DBeaver workspace.
- `dbvr project list|create|delete|rename|default` — projects group datasources.
  The default is `General`; scope any command to a project with `--project`.
- `dbvr driver list` — valid `--driver` ids (e.g. `postgres-jdbc`, `mysql8`).
- `dbvr auth-models` / `dbvr network-handlers` — auth model ids + their `-auth`
  params, and SSH-tunnel/proxy params passed via `-net name=value`.

See `references/command-reference.md` for every subcommand's flags and worked
examples.

## Working safely

- **Default to read-only.** For "look at / explore / what's in" tasks, use `meta`
  and `SELECT`s. Don't run `INSERT`/`UPDATE`/`DELETE`/DDL against a database unless
  the user explicitly asked to modify data, and say which datasource you're hitting
  before you do.
- **Never invent connection details.** If no datasource is named and none is
  obvious from `datasource list`, ask which one to use rather than guessing a host.
- **Watch the 1000-row default.** When counting or aggregating, do it in SQL
  (`SELECT count(*)`) rather than fetching rows and counting client-side.
- **Passwords:** prefer a saved datasource or an env var (`-p "$DB_PASSWORD"`) over
  a literal password on the command line.
- **Discover, don't assume, flags.** Any subcommand's exact options are one
  `dbvr <command> --help` away; check it when unsure instead of guessing.
