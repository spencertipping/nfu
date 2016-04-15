---
title: rorder
long_flag: --rorder
short_flag: -O
categories:
- commands
tags:
- common
- easy
---

*Description:*

Sorts descending by numeric value.

*Arguments:*

1 optional: Column number (zero-indexed) to sort on.

*Examples:*

```shell
nfu random-numbers -O

# Compare this with "nfu n:20 -G"
nfu n:20 -O

# Sort events by day of month instead of year
nfu events-1900s.tsv -O 2
```
