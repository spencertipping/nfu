# nfu: Numeric Fu for your shell
`nfu` is desgined to do a bunch of common/useful numeric tasks to text-oriented
data. For example, suppose you want to look at the cumulative distribution of
words in a text file, ordered by most common first. In plain shell, you'd write
this:

```sh
$ egrep -o \w+ file | sort | uniq -c | sort -rn | \
    perl -ane 'print $x += $F[0], "\t$F[1]\n"' | \
    { echo 'plot "-" with lines'; cat; } | \
    gnuplot -persist
```

And quite frankly, that's ridiculous. Here's what you'd say in `nfu`:

```sh
$ egrep -o \w+ file | nfu -gcOs --plot 'with lines'
```

Here's a list of operations supported by `nfu`:

- `-c`, `--count`: Pipes data through `uniq -c` to count equivalent items.
- `-d`, `--delta`: The inverse of `sum`; returns the difference between
  successive numbers.
- `-e`, `--eval`: Allows you to transform data with a Perl expression.
  Individual fields are available in `$_`. If you return a single value, then
  it replaces the first column; otherwise your data replaces all values in the
  row.
- `-f`, `--fields`: Allows you to reorder fields arbitrarily, outputting
  tab-delimited data.
- `-g`, `--group`: Pipes data through `sort` to create groups of equivalent
  entries. Assumes lexicographic, not numeric, sort.
- `-G`, `--rgroup`: Same as `group`, but reverses the sort order.
- `-o`, `--order`: Orders elements by numeric value.
- `-O`, `--rorder`: Same as `order`, but reverses the sort.
- `-s`, `--sum`: Takes a running total of the given numbers.
