# nfu: Numeric Fu for your shell
`nfu` is a text data hub and transformation tool with a large set of composable
functions and data source/sink adapters. For example, if you wanted to do a
map-side inner join between a PostgreSQL table, a CSV from the Internet, and
stuff on HDFS and gather the results into a sorted/uniqued text file:

```sh
$ nfu psql:'select * from mytable' \
      -i0 nfu[ http://data.com/csv -F, ] \
      --hadoopcat [ -i0 hdfsjoin:/path/to/hdfs/data ] \
                  [ -gcf1. ] \
      -g \
  > output
```

Then if you wanted to plot a cumulative histogram of the `.metadata.size` JSON
field from the third column values, binned to the nearest 100:

```sh
# long version
$ nfu output --map 'json_decode($_[2]).metadata.size' \
             --quant 100 --order --count --rorder \
             --sum 1 --fields 10 --plot 'with lines'

# equivalent short version
$ nfu output -m 'jd(%2).metadata.size' -q100ocOs1f10p %l
```

- [The Humongous Survival Guide](humongous-survival-guide.md)
- [The nfu Cookbook](cookbook.md)
- [nfu and Hadoop Streaming](hadoop.md)
- [nfu and PostgreSQL](postgres.md)

MIT license as usual.
