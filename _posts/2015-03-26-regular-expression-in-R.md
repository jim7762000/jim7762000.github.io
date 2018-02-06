---
layout: post
title: Regular expression in R
---

Several examples of regular expression in R.

Regular expression is a powerful tool to process strings. First example, we have strings `'q123er', 'a2334bc', 'b78gg'` and we want to let it be `'qer123qer', 'abc2334abc', 'bgg78bgg'`.

``` R
a = c('q123er', 'a2334bc', 'b78gg')
a_out = gsub('([a-z])(\\d*)([a-z]{2})', '\\1\\3\\2\\1\\3', a)
all.equal(c('qer123qer', 'abc2334abc', 'bgg78bgg'), a_out)
# TRUE
```

Second example is that we want add some string in front of the some characters. We want to search parenthesis in string, but parenthesis need escape or it cannot be found.

``` R
b = c('p(478)cer', 'ba(2334)bdc', 'ber(728)gigi')
b2 = gsub("(\\(|\\))", "\\\\\\1", b)
b_out = gsub("(\\(|\\))", "", b)
b_out
# [1] "p478cer"    "ba2334bdc"  "ber728gigi"
```

Third example is that we take the information from strings.

``` R
d = c('p478cer', 'ba2334bdc', 'ber728gigi')
d_pattern = regexec('[a-z]*(\\d*)[a-z]*', d)
d_out = sapply(regmatches(d, d_pattern), function(x) x[2])
d_out
# [1] "478"  "2334" "728"
```

