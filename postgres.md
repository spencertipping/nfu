# PostgreSQL functions
nfu uses the `psql` command-line tool to import and export TSV data from
postgres tables. It supports two commands:

- `--psql table 'schema' 'query'`: Create `table` with `schema`, load it up
  from stdin, export SQL `query` results as TSV to stdout, and delete `table`.
- `--psqlinto table 'schema'`: Create `table` with `schema`, load it up from
  stdin, and print table name.

`--psql` is useful when you want to use postgres as a temporary workspace,
whereas `--psqlinto` leaves the table around for later use.

```sh
# trivial example
$ seq 5 | nfu --psql foo 'x integer' 'select * from foo'
1
2
3
4
5
```

In addition to these commands, nfu also supports a `psql:` pseudofile that lets
you read query outputs as data. For example:

```sh
$ seq 10 | nfu -m 'row %0, %0 * %0' \
               --psqlinto squares 'x integer, square integer'
$ nfu psql:'select * from squares' -OT5
10      100
9       81
8       64
7       49
6       36
```

## Schema inference
If you specify `_` as the schema, nfu will scan the first 20 lines of input and
come up with a set of columns whose names are `f0`, `f1`, ..., and whose types
are one of `text`, `integer`, and `real`.

## Indexing
nfu will create a non-unique index on the table's first column if you prepend
`+` to the table name.
