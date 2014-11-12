# nfu: Numeric Fu for your shell
`nfu` gives you a handful of simple operators that do things you'd otherwise do
with shell commands. For example, here are some different ways to add a list of
numbers:

```sh
$ seq 100 | perl -e 'print $x += $_, "\n"' | tail -n1
$ seq 100 | paste -d+ -s | bc
$ seq 100 | nfu -s | tail -n1
$ seq 100 | nfu -s -T+1                 # "+1" means "take last"
$ seq 100 | nfu -sT+1                   # most commands can be combined
```

nfu can do much more than just adding things. (**TODO:** elaborate on this point
a little)
