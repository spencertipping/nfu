# nfu and Hadoop Streaming
nfu provides tight integration with Hadoop Streaming, preserving its usual
pipe-chain semantics while leveraging HDFS for intermediate data storage. It
does this by providing the `--hadoop` (`-H`) function:

- `--hadoop 'outpath' 'mapper' 'reducer'`: emit data to specified output path,
  printing the output path name as a pseudofile (this lets you say things like
  `nfu $(nfu --hadoop ....) ...`.
- `--hadoop . 'mapper' 'reducer'`: if `outpath` is `.`, then print output data.
- `--hadoop @ 'mapper' 'reducer'`: if `outpath` is `@`, create a temporary path
  and print its pseudofile name.

The "mapper" and "reducer" arguments are arbitrary shell commands that may or
may not involve nfu. "reducer" can be set to `NONE`, `-`, or `_` to run a
map-only job. The mapper, reducer, or both can be `:` as a shorthand for the
identity job. As a result of this and the way nfu handles short options, the
following idioms are supported:

- `-H@`: hadoop and output filename
- `-H.`: hadoop and cat output
- `-H@::`: reshard data, as in preparation for a mapside join
- `-H.: ^gcf1.`: a distributed version of `sort | uniq`

Normally you'd write the mapper and reducer either as external commands, or by
using `nfu --quote ...` or `nfu -Q...`. However, nfu provides two shorthand
notations for quoted forms:

- `[ -gc ]` is the same as `"$(nfu --quote -gc)"` (NB: spaces around brackets
  are required)
- `^gc` is the same as `[ -gc ]`

Quoted nfu jobs can involve `--use` clauses, which are turned into `--run`
before hadoop sends the command to the workers.

Hadoop jobs support some magic to simplify data transfer:

```sh
# upload from stdin
$ seq 100 | nfu --hadoop . "$(nfu --quote -m 'row %0, %0 + 1')" _
$ seq 100 | nfu --hadoop . [ -m 'row %0, %0 + 1' ] _

# upload from pseudofiles
$ nfu sh:'seq 100' --hadoop ...

# use data already on HDFS
$ nfu hdfs:/path/to/data --hadoop ...
```

As a special case, nfu looks for cases where stdin contains lines beginning
with `hdfs:/` and interprets this as a list of HDFS files to process (rather
than being verbatim data). This allows you to chain `--hadoop` jobs without
downloading/uploading all of the intermediate results:

```sh
# two hadoop jobs; intermediate results stay on HDFS and are never downloaded
$ seq 100 | nfu -H@ [ -m 'row %0 % 10, %0 + 1' ] ^gc \
                -H. ^C _
```

nfu detects when it's being run as a hadoop streaming job and changes its
verbose behavior to create hadoop counters. This means you can get the same
kind of throughput statistics by using the `-v` option:

```sh
$ seq 10000 | nfu --hadoop . ^vgc _
```

Because `hdfs:` is a pseudofile prefix, you can also transparently download
HDFS data to process locally:

```sh
$ nfu hdfs:/path/to/data -gc
```

## Mapside joins
You can use nfu's `--index`, `--indexouter`, `--join`, and `--joinouter`
functions to join arbitrary data on HDFS. Because HDFS data is often large and
consistently partitioned, nfu provides a `hdfsjoin:path` pseudofile that
assumes Hadoop default partitioning and expands into a list of partfiles
sufficient to cover all keys that coincide with the current mapper's data.
Here's an example of how you might use it:

```sh
# take 10000 words at random and generate [word, length] in /tmp/nfu-jointest
# NB: you need a reducer here (even though it's a no-op); otherwise Hadoop
# won't partition your mapper outputs.
$ nfu sh:'shuf /usr/share/dict/words' \
  --take 10000 \
  --hadoop /tmp/nfu-jointest [ -vm 'row %0, length %0' ] ^v

# now inner-join against that data
$ nfu /usr/share/dict/words \
  --hadoop . [ -vi0 hdfsjoin:/tmp/nfu-jointest ] _
```

## Examples
### Word count
```sh
# local version:
$ nfu hadoop.md -m  'map row($_, 1), split /\s+/, %0' \
                -gA 'row $_, sum @{%1}'

# process on hadoop, download outputs:
$ nfu hadoop.md -H. [ -m 'map row($_, 1), split /\s+/, %0' ] \
                    [ -A 'row $_, sum @{%1}' ]

# leave on HDFS, download separately using hdfs: pseudofile
$ nfu hadoop.md -H /tmp/nfu-wordcount-outputs \
                   [ -m 'map row($_, 1), split /\s+/, %0' ] \
                   [ -A 'row $_, sum @{%1}' ]
$ nfu hdfs:/tmp/nfu-wordcount-outputs -g
```
