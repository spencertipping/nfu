# SQL databases
nfu knows how to talk to PostgreSQL and SQLite 3 using their command-line
interfaces. There are two ways to do this:

- `--sql` (`-Q`) command: populate a table and optionally issue a query
- `sql:` pseudofile: use a database query as TSV data

**WARNING:** `--sql` always first drops and recreates the table before
importing data. The `sql:` pseudofile does not delete or modify anything
(unless you specifically write that into a query).

You indicate postgres vs sqlite using a `P` or `S` prefix on the database name;
otherwise the commands behave identically between databases. For example:

```sh
# import data into an indexed sqlite table
$ nfu /usr/share/dict/words \
      -m 'row %0, length %0' \
      -Q S@:+wordlengths _ _

# length of all words starting with 'a'
$ nfu sql:S@:"%*wordlengths %w f0 LIKE 'a%'"

# nfu explains what's going on here:
$ nfu --expand-sql "%*wordlengths %w f0 LIKE 'a%'"
 select * from wordlengths  where f0 LIKE 'a%'
```

## How this works
The first command generates pairs of `word, length`, which the sqlite3 command
batch-imports into an indexed table called `wordlengths`. Here's how nfu makes
this happen.

`--sql` takes three arguments: `{P|S}dbname:tablename`, `schema`, and `query`,
with the following special cases:

- if `dbname` is `@`, a "default" database is used. For postgres this is your
  username, and for sqlite3 this is `/tmp/nfu-$USER-sqlite.db`.
- if `tablename` begins with `+`, a first-column index is created after the
  data is inserted. If `tablename` is omitted altogether, nfu defaults to `+t`
  -- a table called `t` with a first-field index.
- if `schema` is `_`, nfu looks at the first 20 lines of data and infers one,
  generating column names `f0`, `f1`, ..., `fN-1`.
- if `query` is `_`, nfu prints a `sql:` pseudofile that will read the whole
  table. Otherwise the query is executed and the results printed as TSV.

Queries are subject to SQL shorthand expansion; you can use `--expand-sql` to
see the result. (All shorthands begin with `%`.)
