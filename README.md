# nfu: Numeric Fu for your shell
`nfu` is desgined to do a bunch of common/useful numeric tasks to text-oriented
data. For example, suppose you want to look at the cumulative distribution of
words in a text file, ordered by most common first. In plain shell, you'd
probably write something like this:

```sh
$ egrep -o '\w+' file | sort | uniq -c | sort -rn | \
>   perl -ane 'print $x += $F[0], "\t$F[1]\n"' | \
>   gnuplot -e 'plot "-" with lines' -persist
```

Here's what you'd say with `nfu`:

```sh
$ egrep -o '\w+' file | nfu -gcOsf0p %l
```

- `g` = "group", which sorts things
- `c` = "count", which means `uniq -c`
- `O` = "reverse order", which means `sort -rn`
- `s` = "sum"
- `f0` = "field 0", which is what awk calls `$1`
- `p` = "plot", which uses gnuplot and croaks if you don't have it

Most `nfu` commands operate on the first column, leaving the others untouched.

## Assumptions
- Columns are always tab-delimited, and may be empty. You can use the `-F`
  (`--fieldsplit`) command if you're starting with data that isn't
  tab-separated.

## Syntax
```sh
$ nfu [--use file.pl]... [options...] [files...]
```

`nfu` will automatically uncompress any files ending in `.xz`, `.bz2`, `.lzo`,
or `.gz`, and will behave like `cat` if you give it multiple files. If you use
`-v`, each file's data transfer rate will be measured using `pv` or `pipemeter`
(whichever is on your system, failing over to `cat` if none). `-v` can be used
anywhere to measure throughput and total transfer through any piece of the
pipeline you're constructing.

Files can also be specified as `[http[s]:]//website/path` (handled with curl),
`[user@]host:path/file` (handled with `ssh -C`), or `sh:command [args...]`.

The `--use` option lets you load files with Perl definitions in them. These
definitions are then visible to any code nfu compiles. Note that all `--use`
options must precede filenames and nfu commands.

`-e` (`--eval`) rewrites any `%n` into Perl's more verbose `$_[n]`. As a
result:

```sh
$ seq 100 | nfu -e 'int log $_[0]'
$ seq 100 | nfu -e 'int log %0'         # same thing
$ seq 100 | nfu -le 'int %0'            # same thing
$ seq 100 | nfu -le int                 # same, but only for one-column input
$ seq 100 | nfu -lq1                    # almost; q rounds, not truncates
```

## Examples
```sh
$ nfu -e length -f0Op %l nfu            # falloff plot of line lengths of nfu
$ seq 100 | nfu -sp                     # running total of 1 .. 100
$ seq 50 | nfu -e '%0 ** 2' -sp         # running total of 1^2, 2^2, ... 50^2
$ seq 100 | nfu -sssp                   # third-integral of 1 .. 100
$ egrep -o '\w' words | nfu -gcO        # descending letter frequency distribution
$ seq 100 | shuf > data
$ nfu -a5 data                          # sliding average of up to 5 elements
$ nfu -a data                           # running average of all items so far
$ nfu --slice 10 10 -dp data            # remove first/last 10, delta, plot
$ seq 100 | nfu -lq0.01                 # list of logs, two decimal places
$ nfu -F:f6gcO /etc/passwd              # login shells by frequency
```

### Local map/reduce
`nfu` can also be used to create local map/reduce workflows. The key is the
sequence `-gA`, which lets you process all inputs whose first column has the
same value. For example, suppose you want to find the average word length for
each two-letter suffix:

```
$ nfu -v /usr/share/dict/words \
      -e '($1, length %0) if %0 =~ /(..)$/g' \  # map into (suffix, length)
      -gA 'row $_, sum(@{%1}) / @{%1}'          # reduce into (suffix, avg)
```

Unlike real map/reduce, this collects all grouped records into memory at once
and uses no parallelization. But it may still be useful for debugging and
prototyping when you have a large dataset made of small facets.

(Note that `row ...` is just a shorthand for `join "\t", ...`; `-A` puts
multiple values on multiple lines, so `row` flattens the result into a single
line with multiple columns.)

### gnuplot syntax expansions
`nfu` expands a few shorthands for common gnuplot options:

- `%l` = `with lines`
- `%d` = `with dots`
- `%i` = `with impulses`

So, for instance, `nfu -p %l filename` plots `filename` with lines. Note that
the space after `-p` is required; otherwise `nfu` will have no idea what you're
doing.

## Environment variables
Listed here with default values.

```sh
NFU_SORT_BUFFER=256M                    # in-memory sort buffer size
NFU_SORT_PARALLEL=4                     # max parallel mergesorts
NFU_SORT_COMPRESS=                      # program to compress disk temps
NFU_VERBOSE_COMMAND=                    # path to pipemeter, pv, or similar
```

You can set `NFU_NO_PAGER` to any truthy string value to prevent nfu from using
`less` to preview results sent to a terminal.

## Commands
`nfu` chains commands together just like a shell pipeline. This means that
order matters; `nfu -sc` and `nfu -cs` do two completely different things.

- `-a`, `--average`: Generates a running average of the last N elements. If N =
  0 or is not provided, then generates a running average of all numbers.
- `-A`, `--aggregate`: Allows you to transform groups of rows with a Perl
  expression. Adjacent rows are grouped by their first field value, and your
  expression is invoked with `@_` containing a series of column arrays of
  values (in references). `$_` and `$_[0][0]` both refer to the join key. Note
  that unlike `eval`, any array returned from an aggregation function will
  produce multiple output rows, not columns. You can use the `row` function as
  a shorthand for `join "\t", ...` if you just want a single output row.
- `-c`, `--count`: Counts adjacent, equivalent items. You should probably use
  `-g` before this unless your data is already grouped or you just want run
  lengths.
- `-d`, `--delta`: The inverse of `sum`; returns the difference between
  successive numbers.
- `-D`, `--drop`: Drops the first N lines.
- `-e`, `--eval`: Allows you to transform data with a Perl expression.
  Individual fields are available in `@_`. If you return a single value, then
  it replaces the first column; otherwise your data replaces all values in the
  row. If you return an empty list, no output row is generated. You can store
  state across invocations by writing to `$_`, which is initially the empty
  string.
- `-E`, `--every`: Prints every nth line; this gives you a way to sample large
  datasets.
- `-f`, `--fields`: Allows you to reorder fields arbitrarily, outputting
  tab-delimited data. Takes a single string of digits, each of which is a
  zero-based field index.
- `-F`, `--fieldsplit`: Splits each line of the input on the given Perl regexp,
  which allows you to take in CSV or whitespace-separated data.
- `-g`, `--group`: Pipes data through `sort` to create groups of equivalent
  entries. Assumes lexicographic, not numeric, sort.
- `-G`, `--rgroup`: Same as `group`, but reverses the sort order.
- `-i`, `--index`: Takes a zero-based field number and a shell command, and
  uses `join` to inner-join on the given field. The first column of the shell
  command output is always used, and neither side needs to be sorted initially
  (nfu takes care of it for you).
- `-I`, `--indexouter`: Same as `--index`, but uses an outer left join (where
  the "left" side is the primary nfu data stream).
- `-j`, `--json`: Same as `eval`, but the first field is automatically parsed
  as JSON and the result stored in `$_`. Also, any `.(\w+)` in the expression
  are converted to `->{'$1'}`, so you can write `$_.foo` instead of
  `$_->{'foo'}`, for instance. Returning multiple values causes them to be
  written as plain text, not re-encoded. Returning zero values with `()` causes
  no output row to be generated.
- `-J`, `--jsonflat`: Just like `--json`, but never re-encodes into JSON, and
  multiple values are placed on separate _lines_, not in separate columns. You
  can use the `row(...)` function to join values with tabs.
- `-k`, `--keep`: The opposite of `--remove`.
- `-l`, `--log`: Log-transforms every value.
- `-L`, `--exp`: Exponent-transforms every value.
- `-m`, `--map`: Just like `eval`, but multiple values are emitted on multiple
  lines. If you want multiple columns on a single line, you can use the `row`
  function, which is a shorthand for `join "\t"`.
- `-n`, `--number`: Prefixes each line with a line number, beginning with 1.
- `-o`, `--order`: Orders elements by numeric value.
- `-O`, `--rorder`: Same as `order`, but reverses the sort.
- `-p`, `--plot`: Plots the input data as-is. You may need to reorder or slice
  fields to get gnuplot to work correctly. Takes a string of gnuplot options,
  which are subject to some expansion rules (see "gnuplot syntax expansions"
  above).
- `-P`, `--poll`: Takes an interval in seconds and a command, and runs the
  command forever, sleeping by the interval between runs. You can use this to
  generate a stream of data.
- `-q`, `--quant`: Quantize each number to the nearest x, which defaults to 1.
  x can be any positive value.
- `-r`, `--reduce`: Identical to `eval`, but takes a first argument specifying
  the initial value for `$_`. For example, `nfu -r 0 '$_ += %0'` is the same as
  `nfu -s`.
- `-R`, `--remove`: Analogous to `eval`, but removes any row for which the
  expression returns a truthy value.
- `-s`, `--sum`: Takes a running total of the given numbers.
- `-S`, `--sample`: Takes a probability and returns each record randomly with
  that probability.
- `-T`, `--take`: Takes the first N records, discarding the rest.
- `-V`, `--variance`: Running variance of the first column.
- `-v`, `--verbose`: Measures data throughput interactively on stderr, emitting
  it untransformed. This can be used anywhere in your pipeline, though
  pipemeter doesn't stack like pv does.
- `-w`, `--with`: Takes a quoted shell command and joins line-wise (analogous
  to a list zip).

### Long-only options
- `--sd`: Running standard deviation of the first column.
- `--bits`: Takes the total of a list of numbers, and returns the number of
  bits required to arithmetic-encode any given row, given that the row's
  numbers represent relative frequencies. You can use this to compute the
  entropy of a statistical distribution.
- `--splot`: `splot` command for gnuplot. Takes the same options as `--plot`,
  and applies gnuplot-specific expansions (see "gnuplot syntax expansions"
  above).
- `--slice`: Takes two numbers: #lines to chop from head, #lines to chop from
  tail.
