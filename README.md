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
$ nfu [--use file.pl]... [--verbose] [options...] [files...]
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
