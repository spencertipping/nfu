---
title: group
long_flag: --group
short_flag: -g
categories:
- commands
tags:
- common
- easy
---

*Description:*

Sorts ascending alphanumerically.

*Arguments:*

1 optional: Column number (zero-indexed) to sort on.

*Examples:*

```shell
nfu months -g

# Compare this with "nfu n:20 -o"
nfu n:20 -g

# Groups the events by month, alphabetically ascending
nfu events-1900s.tsv -g
```
