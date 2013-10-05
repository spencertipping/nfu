# nfu: Numeric Fu for your shell
`nfu` is desgined to do a bunch of common/useful numeric tasks to text-oriented
data. For example, suppose you want to look at the cumulative distribution of
words in a text file, ordered by most common first. In plain shell, you'd write
this:

```sh
$ egrep -o '\w+' file | sort | uniq -c | sort -rn | \
    perl -ane 'print $x += $F[0], "\t$F[1]\n"' | \
    gnuplot -e 'plot "-" with lines' -persist
```

And quite frankly, that's ridiculous. Here's what you'd say in `nfu`:

```sh
$ egrep -o '\w+' file | nfu -gcOsf 0 --plot 'with lines'
```

## Examples
```sh
$ seq 100 | nfu -sp                     # running total of 1 .. 100
$ seq 50 | nfu -e '$_[0] ** 2' -sp      # running total of 1^2, 2^2, ... 50^2
$ seq 100 | nfu -sssp                   # third-integral of 1 .. 100
$ egrep -o '\w' words | nfu -gcO        # letter frequency distribution
```

Here's a list of operations supported by `nfu`:

- `-c`, `--count`: Pipes data through `uniq -c` to count adjacent, equivalent
  items. You should probably use `-g` before this unless your data is already
  grouped.
- `-d`, `--delta`: The inverse of `sum`; returns the difference between
  successive numbers.
- `-e`, `--eval`: Allows you to transform data with a Perl expression.
  Individual fields are available in `@_`. If you return a single value, then
  it replaces the first column; otherwise your data replaces all values in the
  row.
- `-f`, `--fields`: Allows you to reorder fields arbitrarily, outputting
  tab-delimited data. Takes a single string of digits, each of which is a
  zero-based field index.
- `-g`, `--group`: Pipes data through `sort` to create groups of equivalent
  entries. Assumes lexicographic, not numeric, sort.
- `-G`, `--rgroup`: Same as `group`, but reverses the sort order.
- `-o`, `--order`: Orders elements by numeric value.
- `-O`, `--rorder`: Same as `order`, but reverses the sort.
- `-s`, `--sum`: Takes a running total of the given numbers.
- `-p`, `--plot`: Plots the input data as-is. You may need to reorder or slice
  fields to get gnuplot to work correctly.
