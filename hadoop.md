# nfu and Hadoop Streaming
nfu provides tight integration with Hadoop Streaming, preserving its usual
pipe-chain semantics while leveraging HDFS for intermediate data storage. It
does this by providing three Hadoop-related commands:

- `--hadoop 'mapper' 'reducer'`: process data, emitting list of partfiles
- `--hadoopcat 'mapper' 'reducer'`: same as `--hadoop`, but cat output data
- `--hadoopinto 'mapper' 'reducer' 'outpath'`: same as `--hadoop`, but with a
  specified destination (rather than a tempfile)

The "mapper" and "reducer" arguments are arbitrary shell commands that may or
may not involve nfu. "reducer" can be set to `NONE` to run a map-only job.

Hadoop jobs support some magic to simplify data transfer:

```sh
# upload from stdin
$ seq 100 | nfu --hadoopcat "$(nfu --quote -m 'row %0, %0 + 1')" NONE

# upload from pseudofiles
$ nfu sh:'seq 100' --hadoopcat ...

# use data already on HDFS
$ nfu hdfs:/path/to/data --hadoopcat ...
```

As a special case, nfu looks for cases where stdin contains lines beginning
with `hdfs:/` and interprets this as a list of HDFS files to process (rather
than being verbatim data). This allows you to chain `--hadoop` jobs without
downloading/uploading all of the intermediate results:

```sh
# two hadoop jobs; intermediate results stay on HDFS and are never downloaded
$ seq 100 | nfu --hadoop "$(nfu -Qm 'row %0 % 10, %0 + 1')" \
                         "$(nfu -Qgc)" \
                --hadoopcat "$(nfu -C)" NONE
```

nfu detects when it's being run as a hadoop streaming job and changes its
verbose behavior to create hadoop counters. This means you can get the same
kind of throughput statistics by using the `-v` option:

```sh
$ seq 10000 | nfu --hadoopcat "$(nfu -Qvgc)" NONE
```

Because `hdfs:` is a pseudofile prefix, you can also transparently download
HDFS data to process locally:

```sh
$ nfu hdfs:/path/to/data -gc
```

## Examples
### Word count
```sh
# local version:
$ nfu hadoop.md -m  'map row($_, 1), split /\s+/, %0' \
                -gA 'row $_, sum @{%1}'

# process on hadoop, download outputs:
$ nfu hadoop.md --hadoopcat "$(nfu -Qm 'map row($_, 1), split /\s+/, %0')" \
                            "$(nfu -QA 'row $_, sum @{%1}')"

# leave on HDFS, download separately using hdfs: pseudofile
$ nfu hadoop.md --hadoopinto "$(nfu -Qm 'map row($_, 1), split /\s+/, %0')" \
                             "$(nfu -QA 'row $_, sum @{%1}')" \
                             /tmp/nfu-wordcount-outputs
$ nfu hdfs:/tmp/nfu-wordcount-outputs -g
```
