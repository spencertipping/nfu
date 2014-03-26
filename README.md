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
$ egrep -o '\w+' file | nfu -gcOsf0p 'with lines'
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
$ nfu [options...] [files...]
```

`nfu` will automatically uncompress any files ending in `.xz`, `.bz2`, `.lzo`,
or `.gz`, and will behave like `cat` if you give it multiple files. If you use
`-v`, each file's data transfer rate will be measured using `pv` or `pipemeter`
(whichever is on your system, failing over to `cat` if none). `-v` can be used
anywhere to measure throughput and total transfer through any piece of the
pipeline you're constructing.

Files can also be specified as `[http[s]:]//website/path` (handled with curl)
or `[user@]host:path/file` (handled with `ssh -C`).

`nfu` accepts both long-form and short-form options, and there are a couple of
different ways to write numbers. This is especially relevant if you're using
things like `-S` (`--slice`), which take two numeric arguments:

```sh
$ nfu --slice 10 10
$ nfu -S 10 10                          # same thing
$ nfu -S10,10                           # also the same...
$ nfu -S10,10p                          # ... but more composable
$ nfu -a50p                             # works with any numeric-arg command
```

The other syntactic transformation happens with `-e` (`--eval`), which rewrites
any `%n` into Perl's more verbose `$_[n]`. As a result:

```sh
$ seq 100 | nfu -e 'int log $_[0]'
$ seq 100 | nfu -e 'int log %0'         # same thing
$ seq 100 | nfu -le 'int %0'            # same thing
$ seq 100 | nfu -le int                 # same, but only for one-column input
$ seq 100 | nfu -lq1                    # almost; q rounds, not truncates
```

## Examples
```sh
$ nfu -e length -f0Op 'with lines' nfu  # falloff plot of line lengths of nfu
$ seq 100 | nfu -sp                     # running total of 1 .. 100
$ seq 50 | nfu -e '%0 ** 2' -sp         # running total of 1^2, 2^2, ... 50^2
$ seq 100 | nfu -sssp                   # third-integral of 1 .. 100
$ egrep -o '\w' words | nfu -gcO        # descending letter frequency distribution
$ seq 100 | shuf > data
$ nfu -a5 data                          # sliding average of up to 5 elements
$ nfu -a data                           # running average of all items so far
$ nfu -S10,10dp data                    # remove first/last 10, delta, plot
$ seq 100 | nfu -lq0.01                 # list of logs, two decimal places
$ nfu -F:f6gcO /etc/passwd              # login shells by frequency
```

## Environment variables
Listed here with default values.

```sh
NFU_SORT_BUFFER=256M                    # in-memory sort buffer size
NFU_SORT_PARALLEL=4                     # max parallel mergesorts
NFU_SORT_COMPRESS=                      # program to compress disk temps
```

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
  produce multiple output rows, not columns.
- `-c`, `--count`: Counts adjacent, equivalent items. You should probably use
  `-g` before this unless your data is already grouped or you just want run
  lengths.
- `-d`, `--delta`: The inverse of `sum`; returns the difference between
  successive numbers.
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
- `-j`, `--json`: Same as `eval`, but the first field is automatically parsed
  as JSON and the result stored in `$_`. Also, any `.(\w+)` in the expression
  are converted to `->{'$1'}`, so you can write `$_.foo` instead of
  `$_->{'foo'}`, for instance. Returning multiple values causes them to be
  written as plain text, not re-encoded. Returning zero values with `()` causes
  no output row to be generated.
- `-J`, `--jsonflat`: Just like `--json`, but never re-encodes into JSON, and
  multiple values are placed on separate _lines_, not in separate columns. You
  can use the `row(...)` function to join values with tabs.
- `-l`, `--log`: Log-transforms every value.
- `-L`, `--exp`: Exponent-transforms every value.
- `-o`, `--order`: Orders elements by numeric value.
- `-O`, `--rorder`: Same as `order`, but reverses the sort.
- `-p`, `--plot`: Plots the input data as-is. You may need to reorder or slice
  fields to get gnuplot to work correctly.
- `-P`, `--poll`: Takes an interval in seconds and a command, and runs the
  command forever, sleeping by the interval between runs. You can use this to
  generate a stream of data.
- `-q`, `--quant`: Quantize each number to the nearest x, which defaults to 1.
  x can be any positive value.
- `-r`, `--reduce`: Identical to `eval`, but takes a first argument specifying
  the initial value for `$_`. For example, `nfu -r 0 '$_ += %0'` is the same as
  `nfu -s`.
- `-s`, `--sum`: Takes a running total of the given numbers.
- `-S`, `--slice`: Takes two numbers: #lines to chop from head, #lines to chop
  from tail.
- `-V`, `--variance`: Running variance of the first column.
- `-v`, `--verbose`: Measures data throughput interactively on stderr, emitting
  it untransformed. This can be used anywhere in your pipeline, though
  pipemeter doesn't stack like pv does.

### Long-only options
- `--sd`: Running standard deviation of the first column.
