# dbvr command reference

Full flag reference and worked examples for `dbvr` (DBeaver headless CLI, v26.x).
SKILL.md covers the common path; consult this when you need an exact flag, a less
common subcommand, or the authoritative example. When still unsure, run
`dbvr <command> --help` — the CLI is self-documenting.

## Table of contents
- [Global options](#global-options)
- [Connecting: -ds vs ad-hoc](#connecting--ds-vs-ad-hoc)
- [sql](#sql)
- [meta](#meta)
- [datasource](#datasource)
- [project](#project)
- [driver / auth-models / network-handlers](#driver--auth-models--network-handlers)
- [License / secret-manager / cloud / mcp](#not-in-this-build)
- [Driver ids seen on this machine](#driver-ids-seen-on-this-machine)

## Global options

Passed before the subcommand:

- `-data=<path>` — workspace location (where DBeaver stores projects/datasources).
  Default is the DBeaver workspace. Use to target a different workspace.
- `--stateless` — disable state storage; nothing is persisted. Use for one-off
  ad-hoc connections you don't want saved.
- `-nl=<locale>` — default locale.
- `--debug-logs` — verbose logging (add when diagnosing a failure).
- `-dump` — print instance thread dump.
- `-V` / `--version`, `-h` / `--help`.

## Connecting: -ds vs ad-hoc

Every data-touching command (`sql`, all `meta ...`) accepts a connection target one
of three ways — they are mutually exclusive:

1. `-ds=<name-or-id>` — an existing saved datasource. Preferred. Names with spaces
   need quotes: `-ds="My Database"`.
2. `-con=<connectionSpec>` — a datasource specification string.
3. `--driver=<id>` + connection details — ad-hoc. Requires `--driver`. Then either
   `--url=<jdbc>` **or** the parts `--host --port --database --server`. Plus auth:
   `-u/--user`, `-p/--password`, `--auth-model`, `-auth name=value` (repeatable).

Shared connection flags (both modes where applicable):

- `-u`, `--user` / `-p`, `--password` — credentials (override the datasource's).
- `--auth-model=<id>` — see `dbvr auth-models`.
- `-auth name=value` — auth property, repeatable.
- `-prop name=value` — JDBC connection property, repeatable.
- `-ext name=value` — provider/extended property, repeatable.
- `-net name=value` — network handler param (e.g. SSH tunnel), repeatable;
  `-net-save-pwd=<bool>` controls saving its secrets (default true).
- `--folder=<name>`, `--name=<label>`, `--save-password=<bool>` — used when the
  connection is being created/saved.

## sql

`dbvr sql [connection] [options] [=<query>]`

Query source (pick one): positional `"<SQL>"`, stdin (piped), or `-in=<file>`.

Options:
- `-format`, `--output-format=csv|xml|json|txt` — default `txt` (Markdown table).
- `-out`, `--output-file=<file>` — write result to a file. Pass an **absolute**
  path: a relative `-out` can exit 0 yet write nothing (it doesn't resolve against
  the shell cwd). Verify the file exists after the run.
- `-op`, `--output-format-parameters=prop1=v1,prop2=v2` — exporter options for the
  format (comma/space delimited).
- `-l`, `--limit="[offset,]limit"` — fetched rows; **default 1000**. Examples:
  `-l 50` (first 50), `-l "100,50"` (offset 100, next 50).
- `--disable-status` — hide the execution-status footer.
- `--print-queries` — echo each statement before executing.
- `-u`, `-p` — credential overrides.

Examples:
```
# Read a small result as a table
dbvr sql -ds="My Database" --disable-status "SELECT 1 AS ping"

# Export to CSV file (absolute -out), uncapped-ish (raise the limit deliberately)
dbvr sql -ds="My Database" -format=csv -out="$PWD/orders.csv" -l 100000 \
  "SELECT * FROM appdb.orders"

# JSON output for programmatic use
dbvr sql -ds="My Database" -format=json -l 5 "SELECT id, name FROM accounts"

# Run a script file, echoing each statement
dbvr sql -ds="My Database" --print-queries -in=migration_check.sql

# Ad-hoc connection, no saved state
dbvr sql --stateless --driver mysql8 --host 127.0.0.1 --port 3306 \
  --database appdb -u root -p "$MYSQL_PWD" "SHOW TABLES"
```

## meta

Metadata operations, uniform across drivers. Subcommands:

- `meta database list|ddl`
- `meta schema list|ddl`
- `meta table list|ddl`

Object-selection flags (in addition to a connection target):
- `-db`, `--database-name=<catalog>` — database/catalog. (Not `--database`, which is
  the ad-hoc *connection* database and forces driver-mode.)
- `-sn`, `--schema-name=<schema>`
- `-tn`, `--table-name=<table>`
- `-fn`, `--full-name=<database.schema.table>` — fully-qualified alternative to the
  parts above.
- `--full` (on `meta table ddl`) — fuller definition incl. indexes/constraints.

Notes:
- MySQL-family (`mysql8`): the *database* is the catalog. `meta schema list` and
  `meta table list` need `-db <database>`. `meta database list` needs only `-ds`.
- Postgres: catalog = database, schema = `public` etc.; supply `-db` and `-sn`.

Examples:
```
dbvr meta database list -ds="My Database"
dbvr meta schema   list -ds="My Database" -db appdb
dbvr meta table    list -ds="My Database" -db appdb
dbvr meta table    ddl  -ds="My Database" -fn "appdb.accounts"
dbvr meta table    ddl  -ds="My Database" -db appdb -tn accounts --full
```

## datasource

`dbvr datasource <create|update|delete|list|view|move>` — mutates the DBeaver
workspace; confirm with the user before create/update/delete.

- `list [--project=<p>]` — table of ID / NAME / DRIVER.
- `view <target>` — connection details.
- `create --driver=<id> [--url | --host --port --database --server] [-u -p]
  [--name] [--folder] [--auth-model] [-auth n=v] [--save-password=<bool>]
  [--project=<p>]`
- `update <target> [same connection flags]`
- `delete <target>`
- `move <target> --project=<p>` — move to another project.

Example:
```
dbvr datasource create --driver postgres-jdbc --name "Local PG" \
  --host localhost --port 5432 --database postgres \
  -u pg_user -p "$PGPASSWORD" --save-password=true
```

## project

`dbvr project <list|create|delete|rename|default>`. Projects group datasources; the
default is `General`. Scope other commands with `--project=<name-or-id>`.

```
dbvr project list
dbvr project create "Analytics"
dbvr project default "Analytics"
```

## driver / auth-models / network-handlers

- `dbvr driver list` — supported drivers with the ids you pass to `--driver`.
- `dbvr auth-models` — auth model ids (e.g. `sqlserver_database`,
  `sqlserver_ad_password`, `google_bigquery`) and each model's `-auth` params.
- `dbvr network-handlers` — SSH-tunnel/proxy handlers; pass params with
  `-net name=value` and control secret saving with `-net-save-pwd`.

## Not in this build

The published docs also describe `import-license`/`list-license`/`delete-license`,
`secret-manager`, `cloud`, and `mcp` commands. These do **not** appear in this
installation's top-level `--help` (likely edition/license-gated). Before using or
recommending them, confirm they exist here: `dbvr --help`. If a user needs the MCP
server or cloud/secret features and they're absent, that's a licensing/edition
matter, not a syntax problem.

## Driver ids seen on this machine

From `dbvr datasource list` on this workspace: `mysql8`, `postgres-jdbc`. Run
`dbvr driver list` for the full supported set before building an ad-hoc connection.
