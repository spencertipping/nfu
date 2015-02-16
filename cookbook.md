# The nfu Cookbook
An ongoing collection of real-world tasks I ended up using nfu to solve. I
recommend using `nfu --explain ...` and `nfu --expand-code '...'` on the
examples below if you're new to nfu.

## First record within each of N categories
I had a series of JSON records, each of which had a "category" field. I wanted
to get 10000 output records, each in a different category.

```sh
$ nfu records -m  'my $j = jd(%0); row $j.metadata.category, %0' \
              -gA '${%1}[0]' \
              -T10000
```

Initially I also wanted to make sure none of the categories were bogus, so I
previewed with the category keys still in the first column like this:

```sh
$ nfu records -m  'my $j = jd(%0); row $j.metadata.category, %0' \
              -gA 'row $_, ${%1}[0]' \
              -T10000
```

## Comparing field values across different JSON formats
There were two files, one with lines in this format:

```
$ head -n1 file1
{"metadata": {"category": "foo", ...}, "id": 1, "name": "bar", ...}
```

and the other, derived from the first, with lines in this format:

```
$ head -n1 file2
{"category": "foo", "data": [{"id": 1, "record": {"name": "bar", ...}, ...}]}
```

I wanted to count up the number of names that had changed. The second file's
rows weren't in the same order as the first, so I needed to join by ID.

```sh
$ nfu file1 -m 'my $j = jd(%0); row $j.id, $j.name' > file1-by-id
$ nfu file2 \
    -m 'my $j = jd(%0); row $j.data->[0].id, $j.data->[0].record.name' \
    -i0 file1-by-id \
    -m 'row %1 ne %2' \
    -sT+1
```

`file1-by-id` contains a TSV of `id, name`, and we end up doing the same thing
to the contents of `file2` before joining (`-i0 file1-by-id`) to get the
combined inner join, `id, file1-name, file2-name`. Perl's `ne` operator returns
zero for equal strings and 1 for unequal strings, so we use `-s` to sum the 1's
up, taking just the last record to get the total.
