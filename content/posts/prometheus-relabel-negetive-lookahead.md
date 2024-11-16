---
layout: post
title: Negetive Lookahead in Prometheus Relabel Config (sort of)
date: 2024-11-16
tags: [prometheus, regex]
---

Prometheus is written in Go and uses [RE2](https://github.com/google/re2/wiki/Syntax) and that does not support negative lookahead. From convenience perspective, its easier to write human-readable regex using that, but as its not supported, we'll see a workaround for that.

Say for example, you have a label `my_label` and you'd like to relabel all the label values to `custom_value` where it contains `my_value` but only when it does not contain `foobar`.

If negative lookahead was supported, you could have written something like this:

```yaml
- source_labels: [my_label]
  regex: 'my_value(?!_foobar)'
  target_label: my_label
  replacement: 'my_value'
```

So a matching metric would be:
```
my_metric{my_label="my_value"} -> my_metric{my_label="custom_value"}
```

And a non-matching metric would be:
```
my_metric{my_label="my_value_foobar_long_value"} -> my_metric{my_label="my_value_foobar_long_value"}
```

But since its not supported, you'll get : 
```
error parsing regexp: invalid or unsupported Perl syntax: `(?!`
```
### Solution
What you can do is to specify the pattern with each possible suffix that you'd like to exclude. 
So in our case, we'd like to exclude `foobar` so the regex pattern would be:

```yaml

regex: (my_value)_(?:[^f]|f[^o]|fo[^o]|foo[^b]|foob[^a]|fooba[^r]).*
```

Now, this is not as clean as negative lookahead, but this way works, and you can exclude any suffix you'd like, so the relabeling will work as expected.

```yaml
- source_labels: [my_label]
  regex: '(my_value)_(?:[^f]|f[^o]|fo[^o]|foo[^b]|foob[^a]|fooba[^r]).*'
  target_label: my_label
  replacement: 'my_value'
```

You can test this relabeling using https://relabeler.promlabs.com. Here's how it looks : 

![relabel_match.png](/relabel_match.png)

![relabel_no_match.png](/relabel_no_match.png)

### Possible Issues
- As we match each char, the relabeling will not work even if the label contains any char  even if its just `foo` or `fooba` etc as long as it starts with `f` and ends with `r`. So you'll have to be careful with the regex pattern you write.