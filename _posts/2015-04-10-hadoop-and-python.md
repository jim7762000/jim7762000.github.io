---
layout: post
title: Hadoop and python
---

A simple log for doing a job of mapreduce in python.

We implement wordcount by using hadoop streaming. New two python script files named mapper.py and reducer.py, respectively.

``` python
#!/usr/bin/env python
## mapper.py
import sys
for line in sys.stdin:
    line = line.strip()
    words = line.split()
    for word in words:
        print '%s\t%s' % (word, 1)
```

``` python
#!/usr/bin/env python
## reducer.py
from operator import itemgetter
import sys
current_word = None
current_count = 0
word = None
for line in sys.stdin:
    line = line.strip()
    word, count = line.split('\t', 1)
    try:
        count = int(count)
    except ValueError:
        continue
    if current_word == word:
        current_count += count
    else:
        if current_word:
            print '%s\t%s' % (current_word, current_count)
        current_count = count
        current_word = word
if current_word == word:
    print '%s\t%s' % (current_word, current_count)
```

Using the example in previous article for hadoop and run hadoop streaming in the terminal:
``` bash
cd ~/Downloads && mkdir testData && cd testData
wget http://www.gutenberg.org/ebooks/5000.txt.utf-8
cd ..
hdfs dfs -copyFromLocal testData/ /user/celest/
hdfs dfs -ls /user/celest/testData/

hadoop jar /usr/local/hadoop/share/hadoop/tools/lib/hadoop-streaming-2.6.0.jar \
-files mapper.py,reducer.py  -mapper "mapper.py -m" \
-reducer "reducer.py -r"  -input /user/celest/testData/* \
-output /user/celest/testData-output

hdfs dfs -cat /user/celest/testData-output/part-00000
```

We can obtain the same result for wordcount.
