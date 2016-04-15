---
title: rgroup
long_flag: --rgroup
short_flag: -G
categories:
- commands
tags:
- common
- easy
---

*Description:*

Sorts descending alphanumerically.

*Arguments:*

1 optional: Column number (zero-indexed) to sort on.

*Examples:*

```shell
nfu months -G

# Compare this with "nfu n:20 -O"
nfu n:20 -G

# Groups the events by month, alphabetically descending
nfu events-1900s.tsv -G
```
