# nfu: Numeric Fu for your shell
Documentation:

- [The Humongous Survival Guide](humongous-survival-guide.md)
- [The nfu Cookbook](cookbook.md)
- [nfu and Hadoop Streaming](hadoop.md)

`nfu` gives you a handful of simple operators that do things you'd otherwise do
with shell commands. For example, here are some different ways to add a list of
numbers:

```sh
$ seq 100 | perl -e 'print $x += $_, "\n"' | tail -n1
$ seq 100 | paste -d+ -s | bc
$ seq 100 | nfu -s | tail -n1
$ seq 100 | nfu -s -T+1                 # "+1" means "take last"
$ seq 100 | nfu -sT+1                   # most commands can be combined
```

## Basic usage
You can use `nfu` like a `less` that automatically decompresses stuff, and you
can run it with no arguments to get a list of operations it supports.

```sh
$ seq 1000 | gzip > numbers.gz
$ nfu numbers.gz                        # decompress and less the file
```

nfu also supports some pseudofile syntax for things like shell commands, HTTP
retrieval, SSH retrieval, etc:

```sh
$ nfu http://google.com                 # uses curl to fetch google.com
$ nfu user@host:path                    # tunnels file over ssh
```

## Simple commands
```
-g, --group     sort by given column, or first if unspecified
-G, --rgroup    reverse sort
-o, --order     numeric sort
-O, --rorder    reverse numeric sort
-c, --count     like 'uniq -c', but lets you customize field(s)
```

Commands can be written in long form, short form, or combined:

```sh
$ nfu /usr/share/dict/words --group --count
$ nfu /usr/share/dict/words -g -c
$ nfu /usr/share/dict/words -gc
```

## Numeric commands
```sh
$ seq 100 | nfu -s              # --sum, running total
$ seq 100 | nfu -S              # --delta
$ seq 100 | nfu -a              # --average
$ seq 100 | nfu -V              # --variance
$ seq 100 | nfu -l              # --log, log-transform individual
$ seq 100 | nfu -L              # --exp, inverse of --log
$ seq 100 | nfu -q0,0.1         # --quant, quantize each number to 0.1
$ seq 100 | nfu -q 0 0.1        # same thing
```

These commands are each single-column by default, but you can apply them to one
or more columns by specifying a string of digits:

```sh
$ seq 100 | nfu -s0             # sum column 0
$ seq 100 \
  | perl -ne 'chomp; print "$_\t$_\n"' \
  | nfu -s01                    # sum columns 0 and 1 independently
$ seq 100 \
  | perl -ne 'chomp; print "$_\t$_\n"' \
  | nfu -s0l1                   # sum column 0, log-transform column 1
```

## Row transformation
These commands take Perl expressions that are rewritten to expand some
shorthands. You can use `nfu --expand-code 'command'` to see how the code is
transformed. Columns of input, always split by `/\t/`, are stored in `@_`.

```sh
$ seq 100 | nfu -m '%0 * %0'    # --map; %0 is expanded to $_[0]
$ seq 100 | nfu -k '%0 =~ /0$/' # --keep
$ seq 100 | nfu -K '%0 =~ /0$/' # --remove
```
