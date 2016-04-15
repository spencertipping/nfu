---
title: order
long_flag: --order
short_flag: -o
categories:
- commands
tags:
- common
- easy
---

*Description:*

Sorts ascending by numeric value.

*Arguments:*

1 optional: Column number (zero-indexed) to sort on.

*Examples:*

```shell
nfu random-numbers -o

# Compare this with "nfu n:20 -g"
nfu n:20 -o

# Sort events by day of month instead of year
nfu events-1900s.tsv -o 2
```
