# The Humongous nfu Survival Guide
## Introduction
nfu is all about tab-delimited text data. It does a number of things to make
this data easier to work with; for example:

```sh
$ git clone git://github.com/spencertipping/nfu
$ cd nfu
$ ./nfu README.md               # behaves like 'less'
$ gzip README.md
$ ./nfu README.md.gz            # transparent decompression (+ xz, bz2, lzo)
```

Now let's do some basic word counting. We can get a word list by using nfu's
`-m` operator, which takes a snippet of Perl code and executes it once for each
line. Then we sort (`-g`, or `--group`), count-distinct (`-c`), and
reverse-numeric-sort (`-O`, or `--rorder`) to get a histogram descending by
frequency:

```sh
$ nfu README.md -m 'split /\W+/, %0' -gcO
48	
28	nfu
20	seq
19	100
...
$
```

`%0` is shorthand for `$_[0]`, which is how you access the first element of
Perl's function-arguments (`@_`) variable. Any Perl code you give to nfu will
be run inside a subroutine, and the arguments are usually tab-separated field
values.

Commands you issue to nfu are chained together using shell pipes. This means
that the following are equivalent:

```sh
$ nfu README.md -m 'split /\W+/, %0' -gcO
$ nfu README.md | nfu -m 'split /\W+/, %0' \
                | nfu -g \
                | nfu -c \
                | nfu -O
```

nfu uses a number of shorthands whose semantics may become confusing. To see
what's going on, you can use its documentation options:

```sh
$ nfu --expand-code 'split /\W+/, %0'
split /\W+/, $_[0]
$ nfu --explain README.md -m 'split /\W+/, %0' -gcO
file	README.md
--map	'split /\W+/, %0'
--group
--count
--rorder
--preview
$
```

You can also run nfu with no arguments to see a usage summary.

## Basic idioms
### Extracting data
- `-m 'split /\W+/, %0'`: convert text file to one word per line
- `-m 'map {split /\W+/} @_'`: same thing for text files with tabs
- `-F '\W+'`: convert file to one word per column, preserving lines
- `-m '@_'`: reshape to a single column, flattening into rows
- `seq 10 | tr '\n' '\t'`: reshape to a single row, flattening into columns

The `-F` operator resplits lines by the regexp you provide. So to parse
/etc/passwd, for example, you'd say `nfu -F : /etc/passwd ...`.

### Generating data
- `-P 5 'cat /proc/loadavg'`: run 'cat /proc/loadavg' every five seconds,
  collecting stdout
- `--repeat 10 README.md`: read README.md 10 times in a row (this is more
  useful than it looks; see "Pipelines, Combination, and Quotation" below)

### Basic transformations
- `-n`: prepend line numbers as first column
- `-m 'row @_, %0 * 2'`: keep all existing columns, appending `%0 * 2` as a new
  one
- `-m '%1 =~ s/foo/bar/g; row @_'`: transform second column by replacing 'foo'
  with 'bar'
- `-m 'row %0, %1 =~ s/foo/bar/gr, @_[2..$#_]'`: same thing, but without
  in-place modification of `%1`

`-M` is a variant of `-m` that runs a pool of parallel subprocesses (by default
16). This doesn't preserve row ordering, but can be useful if you're doing
something latency-bound like fetching web documents:

```sh
$ nfu url-list -M 'row %0, qx(curl %0)'
```

In this example, Perl's `qx()` operator could easily produce a string
containing newlines; in fact most shell commands are written this way. Because
of this, nfu's `row()` function strips the newlines from each of its input
strings. This guarantees that `row()` will produce exactly one line of output.

### Filtering
- `-k '%2 eq "nfu"'`: keep any row whose third column is the text "nfu"
- `-k '%0 < 10'`: keep any row whose first column parses to a number < 10
- `-k '@_ < 5'`: keep any row with fewer than five columns
- `-K '@_ < 5'`: reject any row with fewer than five columns (`-K` vs `-k`)
- `-k 'length %0 < 10'`
- `-k '%0 eq -+-%0'`: keep every row whose first column is numeric

### Row slicing
- `-T5`: take the first 5 lines
- `-T+5`: take the last 5 lines (drop all others)
- `-D5`: drop the first 5 lines
- `--sample 0.01`: take 1% of rows randomly
- `-E100`: take every 100th row deterministically

### Column slicing
- `-f012`: keep the first three columns (fields) in their original order
- `-f10`: swap the first two columns, drop the others
- `-f00.`: duplicate the first column, pushing others to the right
- `-f10.`: swap the first two columns, keep the others in their original order
- `-m 'row(reverse @_)'`: reverse the fields within each row (`row()` is a
  function that keeps an array on one row; otherwise you'd flatten the columns
  across multiple rows)
- `-m 'row(grep /^-/, @_)'`: keep fields beginning with `-`

### Histograms (group, count)
- `-gcO`: descending histogram of most frequent values
- `-gcOl`: descending histogram of most frequent values, log-scaled
- `-gcOs`: cumulative histogram, largest values first
- `-gcf1.`: list of unique values (group, count, fields 1..n)

Sorting and counting operators support field selection:

- `-g1`: sort by second column
- `-c0`: count unique values of field 0
- `-c01`: count unique combinations of fields 0 and 1 jointly

### Common numeric operations
- `-q0.05`: round (quantize) each number to the nearest 0.05
- `-q10`: quantize each number to the nearest 10
- `-s`: running sum
- `-S`: delta (inverse of `-s`)
- `-l`: log-transform each number, base e
- `-L`: inverse log-transform (exponentiate) each number
- `-a`: running average
- `-V`: running variance
- `--sd`: running sample standard deviation

Each of these operations can be applied to a specified set of columns. For
example:

- `seq 10 | nfu -f00s1`: first column is 1..10, second is running sum of first
- `seq 10 | nfu -f00a1`: first column is 1..10, second is running mean of first

Some of these commands take an optional argument; for example, you can get a
windowed average if you specify a second argument to `-a`:

- `seq 10 | nfu -f00a1,5`: second column is a 5-value sliding average
- `seq 10 | nfu -f00q1,5`: second column quantized to 5
- `seq 10 | nfu -f00l1,5`: second column log base-5
- `seq 10 | nfu -f00L1,5`: second column 5<sup>x</sup>

Multiple-digit fields are interpreted as multiple single-digit fields:

- `seq 10 | nfu -f00a01,5`: calculate 5-average of fields 0 and 1 independently

The only ambiguous case happens when you specify only one argument: should it
be interpreted as a column selector, or as a numeric parameter? nfu resolves
this by using it as a parameter if the function requires an argument (e.g.
`-q`), otherwise treating it as a column selector.

### Plotting
Note: all plotting requires that `gnuplot` be in your `$PATH`.

- `seq 100 | nfu -p`: 2D plot; input values are Y coordinates
- `seq 100 | nfu -m 'row @_, %0 * %0' -p`: 2D plot; first column is X, second
  is Y
- `seq 100 | nfu -p %l`: plot with lines
- `seq 100 | nfu -m 'row %0, sin(%0), cos(%0)' --splot`: 3D plot

```sh
$ seq 1000 | nfu -m '%0 * 0.1' \
                 -m 'row %0, sin(%0), cos(%0)' \
                 --splot %l
```

You can use `nfu --expand-gnuplot '%l'`, for example, to see how nfu is
transforming your gnuplot options. (There's also a list of these shorthands in
nfu's usage documentation.)

### Progress reporting
If you're doing something with a large amount of data, it's sometimes hard to
know whether it's worth hitting `^C` and optimizing stuff. To help with this,
nfu has a `--verbose` (`-v`) option that activates throughput metrics for each
operation in the pipeline. For example:

```sh
$ seq 100000000 | nfu -o                # this might take a while
$ seq 100000000 | nfu -v -o             # keep track of lines and kb
```

## Advanced usage (assumes some Perl knowledge)
### JSON
nfu provides two functions, `jd` (or `json_decode`) and `je`/`json_encode`,
that are available within any code you write:

```sh
$ ip_addrs=$(seq 10 | tr '\n' '\r' | nfu -m 'join ",", map "%0.4.4.4", @_')
$ query_url="www.datasciencetoolkit.org/ip2coordinates/$ip_addrs"
$ curl "$query_url" \
  | nfu -m 'my $json = jd(%0);
            map row($_, ${$json}{$_}.locality), keys %$json'
```

This code uses another shorthand, `.locality`, which expands to a Perl hash
dereference `->{"locality"}`. There isn't a similar shorthand for arrays, which
means you need to explicitly dereference those:

```sh
$ echo '[1,2,3]' | nfu -m 'jd(%0)[0]'           # won't work!
$ echo '[1,2,3]' | nfu -m '${jd(%0)}[0]'
```

### Multi-plotting
You can setup a multiplot by creating multiple columns of data. gnuplot then
lets you refer to these with its `using N` construct, which nfu lets you write
as `%uN`:

```sh
$ seq 1000 | nfu -m '%0 * 0.01' | gzip > numbers.gz
$ nfu numbers.gz -m 'row sin(%0), cos(%0)' \
                 --mplot '%u1%l%t"sin(x)"; %u2%l%t"cos(x)"'
$ nfu numbers.gz -m 'sin %0' \
                 -f00a1 \
                 --mplot '%u1%l%t"sin(x)"; %u2%l%t"average(sin(x))"'
$ nfu numbers.gz -m 'sin %0' \
                 -f00a1 \
                 -m 'row @_, %1-%0' \
                 --mplot '%u1%l%t"sin(x)";
                          %u2%l%t"average(sin(x))";
                          %u3%l%t"difference"'
```

The semicolon notation is something nfu requires. It works this way because
internally nfu scripts gnuplot like this:

```
plot "tempfile-name" using 1 with lines title "sin(x)"
plot "tempfile-name" using 2 with lines title "average(sin(x))"
plot "tempfile-name" using 3 with lines title "difference"
```

### Local map-reduce
nfu provides an aggregation operator for sorted data. This groups adjacent rows
by their first column and hands you a series of array references, one for each
column's values within that group. For example, here's word-frequency again,
this time using `-A`:

```sh
$ nfu README.md -m 'split /\W+/, %0' \
                -m 'row %0, 1' \
                -gA 'row $_, sum @{%1}'
```

A couple of things are happening here. First, the current group key is stored
in `$_`; this allows you to avoid the more cumbersome (but equivalent)
`${%0}[0]`. Second, `%1` is now an array reference containing the second field
of all grouped rows. `sum` is provided by nfu and does what you'd expect.

In addition to map/reduce functions, nfu also gives you `--partition`, which
you can use to send groups of records to different files. For example:

```sh
$ nfu README.md -m 'split /\W+/, %0' \
                --partition 'substr(%0, 0, 1)' \
                            'cat > words-starting-with-{}'
```

`--partition` will keep up to 256 subprocesses running; if you have more groups
than that, it will close and reopen pipes as necessary, which will cause your
subprocesses to be restarted. (For this reason, `cat > ...` isn't a great
subprocess; `cat >> ...` is better.)

### Loading Perl code
nfu provides a few utility functions:

- `sum @array`
- `mean @array`
- `uniq @array`
- `frequencies @array`
- `read_file "filename"`: returns a string
- `read_lines "filename"`: returns an array of chomped strings

But sometimes you'll need more definitions to write application-specific code.
For this nfu gives you two options, `--use` and `--run`:

```sh
$ nfu --use myfile.pl ...
$ nfu --run 'sub foo {...}' ...
```

Any definitions will be available inside `-m`, `-A`, and other code-evaluating
operators.

A common case where you'd use `--run` is to precompute some kind of data
structure before using it within a row function. For example, to count up all
words that never appear at the beginning of a line:

```sh
$ nfu README.md -F '\s+' -f0 > first-words
$ nfu --run '$::seen{$_} = 1 for read_lines "first-words"' \
      -m 'split /\W+/, %0' \
      -K '$::seen{%0}'
```

Notice that we're package-scoping `%::seen`. This is required because while row
functions reside in the same package as `--run` and `--use` code, they're in a
different lexical scope. This means that any `my` or `our` variables are
invisible and will trigger compile-time errors if you try to refer to them from
other compiled code.

### Pseudofiles
Gzipped data is uncompressed automatically by an abstraction that nfu calls a
pseudofile. In addition to uncompressing things, several other pseudofile forms
are recognized:

```sh
$ nfu http://factual.com                # uses stdout from curl
$ nfu sh:ls                             # uses stdout from a command
$ nfu user@host:other-file              # pipe file over ssh -C
```

nfu supports pseudofiles everywhere it expects a filename, including in
`read_file` and `read_lines`.

### Pipelines, combination, and quotation
nfu gives you several commands that let you gather data from other sources. For
example:

```sh
$ nfu README.md -m 'split /\W+/, %0' --prepend README.md
$ nfu README.md -m 'split /\W+/, %0' --append README.md
$ nfu README.md --with sh:'tac README.md'
$ nfu --repeat 10 README.md
$ nfu README.md --pipe tac
$ nfu README.md --tee 'cat > README2.md'
$ nfu README.md --duplicate 'cat > README2.md' 'tac > README-reverse.md'
```

Here's what these things do:

- `--prepend`: prepends a pseudofile's contents to the current data
- `--append`: appends a pseudofile
- `--with`: joins a pseudofile column-wise, ending when either side runs out of
  rows
- `--repeat`: repeats a pseudofile the specified number of times, forever if n
  = 0; ignores any prior data
- `--pipe`: same thing as a shell pipe, but doesn't lose nfu state
- `--tee`: duplicates data to a shell process, collecting its _stdout into your
  data stream_ (you can avoid this by using `> /dev/null`)
- `--duplicate`: sends your data to two shell processes, combining their
  stdouts

Sometimes you'll want to use nfu itself as a shell command, but this can become
difficult due to nested quotation. To get around this, nfu provides the
`--quote` operator, which generates a properly quoted command line:

```sh
$ nfu --repeat 10 sh:"$(nfu --quote README.md -m 'split /\W+/, %0')"
```

### Keyed joins
This works on sorted data, and behaves like SQL's JOIN construct. Under the
hood, nfu takes care of the sorting and the voodoo associated with getting
`sort` and `join` to work together, so you can write something simple like
this:

```sh
$ nfu /usr/share/dict/words -m 'row %0, length %0' > bytes-per-word
$ nfu README.md -m 'split /\W+/, %0' \
                -I0 bytes-per-word \
                -m 'row %0, %1 // 0' \
                -gA 'row $_, sum @{%1}'
```

Here's what's going on:

- `-I0 bytes-per-word`: outer left join using field 0 from the data, adjoining
  all columns after the key field from the pseudofile 'bytes-per-word'
- `-m 'row %0, %1 // 0'`: when we didn't get any join data, default to 0 (`//`
  is Perl's defined-or-else operator)
- `-gA 'row $_, sum @{%1}'`: reduce by word, summing total bytes

We could sidestep all nonexistent words by using `-i0` for an inner join
instead. This drops all rows with no corresponding entry in the lookup table.
