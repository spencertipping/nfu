# nfu: Numeric Fu for your shell
**NOTE:** nfu is unlikely to receive any more major updates, as I'm currently
working on its successor [ni](https://github.com/spencertipping/ni).

`nfu` is a text data hub and transformation tool with a large set of composable
functions and source/sink adapters. For example, if you wanted to do a map-side
inner join between a PostgreSQL table, a CSV from the Internet, and stuff on
HDFS and gather the results into a sorted/uniqued text file:

```sh
$ nfu sql:P@:'%*mytable' \
      -i0 @[ http://data.com/csv -F , ] \
      -H@::H. [ -i0 hdfsjoin:/path/to/hdfs/data ] ^gcf1. \
      -g \
  > output

# equivalent long version
$ nfu sql:Pdbname:'select * from mytable' \
      --index 0 @[ http://data.com/csv --fieldsplit , ] \
      --hadoop /tmp/temp-resharded-upload-path [ ] [ ] \
      --hadoop . [ --index 0 hdfsjoin:/path/to/hdfs/data ] \
                 [ --group --count --fields 1. ] \
      --group \
  > output
```

Then if you wanted to plot a cumulative histogram of the `.metadata.size` JSON
field from the third column values, binned to the nearest 100:

```sh
$ nfu output -m 'jd(%2).metadata.size' -q100ocOs1f10p %l

# equivalent long version
$ nfu output --map 'json_decode($_[2]).metadata.size' \
             --quant 100 --order --count --rorder \
             --sum 1 --fields 10 --plot 'with lines'
```

## Documentation
- [The Humongous Survival Guide](humongous-survival-guide.md)
- [The nfu Cookbook](cookbook.md)
- [nfu and Hadoop Streaming](hadoop.md)
- [nfu and SQL databases](sql.md)

## Contributors
- [Spencer Tipping](https://github.com/spencertipping)
- [Factual, Inc](https://github.com/Factual)

MIT license as usual.

## Options and stuff
If you invoke `nfu` with no arguments, it will give you the following summary:

```sh
$ nfu
usage: nfu [prefix-commands...] [input-files...] commands...
where each command is one of the following:

  -A|--aggregate  (1) <aggregator fn>
     --append     (1) <pseudofile; appends its contents to current stream>
  -a|--average    (0) -- window size (0 for full average) -- running average
  -c|--count      (0) -- counts by first column value; like uniq -c
  -S|--delta      (0) -- value -> difference from last value
  -D|--drop       (0) -- number of records to drop
     --duplicate  (2) <two shell commands as separate arguments>
  -e|--each       (1) <template; executes with {} set to each value>
     --entropy    (0) -- running entropy of relative probabilities/frequencies
  -E|--every      (1) <n (returns every nth row)>
  -L|--exp        (0) -- optional base (default e)
  -f|--fields     (0) -- string of digits, each a zero-indexed column selector
  -F|--fieldsplit (1) <regexp to use for splitting>
     --fold       (1) <function that returns true when line should be folded>
  -g|--group      (0) -- sorts ascending, takes optional column list
  -H|--hadoop     (3) <hadoop streaming: outpath|.|@, mapper|:, reducer|:|_>
  -i|--index      (2) <field index, unsorted pseudofile to join against>
  -I|--indexouter (2) <field index, unsorted pseudofile to join against>
  -z|--intify     (0) -- convert column to dense integers (linear space)
  -j|--join       (2) <field index, sorted pseudofile to join against>
  -J|--joinouter  (2) <field index, sorted pseudofile to join against>
  -k|--keep       (1) <row filter fn>
  -l|--log        (0) -- optional base (default e)
  -m|--map        (1) <row map fn>
     --mplot      (1) <gnuplot arguments per column, separated by ;>
  -N|--ntiles     (1) <takes N, produces ntiles of numbers>
  -n|--number     (0) -- prepends line number to each line
  -o|--order      (0) -- sorts ascending by general numeric value
     --partition  (2) <partition id fn, shell command (using {})>
     --pipe       (1) <shell command to pipe through>
  -p|--plot       (1) <gnuplot arguments>
  -M|--pmap       (1) <row map fn (executed multiple times in parallel)>
  -P|--poll       (2) <interval in seconds, command whose output to collect>
     --prepend    (1) <pseudofile; prepends its contents to current stream>
     --preview    (0) 
  -q|--quant      (1) <number to round to>
  -r|--read       (0) -- reads pseudofiles from the data stream
  -K|--remove     (1) <inverted row filter fn>
     --repeat     (2) <repeat count, pseudofile to repeat>
  -G|--rgroup     (0) -- sorts descending, takes optional column list
  -O|--rorder     (0) -- sorts descending by general numeric value
     --sample     (1) <row selection probability in [0, 1]>
     --sd         (0) -- running standard deviation
     --splot      (1) <gnuplot arguments>
  -Q|--sql        (3) <create/query SQL table: db[:[+]table], schema|_, query|_>
  -s|--sum        (0) -- value -> total += value
  -T|--take       (0) -- n to take first n, +n to take last n
     --tee        (1) <shell command; duplicates data to stdin of command>
  -C|--uncount    (0) -- the opposite of --count; repeats each row N times
  -V|--variance   (0) -- running variance
  -w|--with       (1) <pseudofile to join column-wise onto input>

and prefix commands are:

  documentation (not used with normal commands):
    --explain           <other-options>
    --expand-pseudofile <filename>
    --expand-code       <code>
    --expand-gnuplot    <gnuplot options>
    --expand-sql        <sql>

  pipeline modifiers:
    --quote     -- quotes args: eval $(nfu --quote ...)
    --use       <file.pl>
    --run       <perl code>

argument bracket preprocessing:

  ^stuff -> [ -stuff ]

   [ ]    nfu as function: [ -gc ]     == "$(nfu --quote -gc)"
  @[ ]    nfu as data:    @[ -gc foo ] == sh:"$(nfu --quote -gc foo)"

pseudofile patterns:

  file.bz2       decompress file with bzip2 -dc
  file.gz        decompress file with gzip -dc
  file.lzo       decompress file with lzop -dc
  file.xz        decompress file with xz -dc
  hdfs:path      read HDFS file(s) with hadoop fs -text
  hdfsjoin:path  mapside join pseudofile (a subset of hdfs:path)
  http[s]://url  retrieve url with curl
  perl:expr      perl -e 'print "$_\n" for (expr)'
  sh:stuff       run sh -c "stuff", take stdout
  sql:db:query   results of query as TSV
  user@host:x    remote data access (x can be a pseudofile)

gnuplot expansions:

  %d -> ' with dots'
  %i -> ' with impulses'
  %l -> ' with lines'
  %t -> ' title '
  %u -> ' using '

SQL expansions:

  %\* -> ' select * from '
  %c -> ' select count(1) from '
  %d -> ' select distinct * from '
  %g -> ' group by '
  %j -> ' inner join '
  %l -> ' outer left join '
  %r -> ' outer right join '
  %w -> ' where '

database prefixes:

  P = PostgreSQL
  S = SQLite 3

environment variables:

  NFU_ALWAYS_VERBOSE    if set, nfu will be verbose all the time
  NFU_HADOOP_COMMAND    hadoop executable; e.g. hadoop jar, hadoop fs -ls
  NFU_HADOOP_OPTIONS    -D options for hadoop streaming jobs
  NFU_HADOOP_STREAMING  absolute location of hadoop-streaming.jar
  NFU_HADOOP_TMPDIR     default /tmp; temp dir for hadoop uploads
  NFU_MAX_FILEHANDLES   default 64; maximum #subprocesses for --partition
  NFU_NO_PAGER          if set, nfu will not use "less" to preview stdout
  NFU_PMAP_PARALLELISM  number of subprocesses for -M
  NFU_SORT_BUFFER       default 256M; size of in-memory sort for -g and -o
  NFU_SORT_COMPRESS     default none; compression program for sort tempfiles
  NFU_SORT_PARALLEL     default 4; number of concurrent sorts to run

see https://github.com/spencertipping/nfu for documentation
```
