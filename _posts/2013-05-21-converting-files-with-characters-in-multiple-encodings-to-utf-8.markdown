---
published: true
title: Converting files with characters in multiple encodings to utf-8
layout: post
---

It's easy to convert a `windows-1252` file into a `utf-8` one, right? Use `iconv`, `enca` or any other tool of your choice and you're done. Even if you have thousands of such files, you can easily automate things using Bash, OS X Automator or anything else.

But what should you do if you have characters both encoded in `utf-8` and something else? <s>Go find author and kick his ass, obviously.</s> We adopted a [snippet](http://stackoverflow.com/questions/10009753/python-dealing-with-mixed-encoding-files) found on StackOverflow and got the solution:

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

import codecs
import sys
from optparse import OptionParser

last_position = -1
source_encoding = "utf-8"


def mixed_decoder(unicode_error):
    global last_position
    global source_encoding
    string = unicode_error[1]
    position = unicode_error.start
    if position <= last_position:
        position = last_position + 1
    last_position = position
    new_char = string[position].decode(source_encoding)
    #new_char = u"_"
    return new_char, position + 1


codecs.register_error("mixed", mixed_decoder)

parser = OptionParser()
parser.add_option("-s", "--source-encoding", dest="source_encoding", default="utf-8")
(options, args) = parser.parse_args()

source_encoding = options.source_encoding
target_file = args[0]

if not args:
    print 'Script should be called with valid file name as an argument. The file will be converted to utf-8.'
    sys.exit(1)

try:
    f = open(target_file, 'r')
except IOError:
    print target_file + " is not a valid file"
    sys.exit(1)

s = f.read()
f.close()

s = s.decode("utf-8", "mixed");

f = open(target_file, 'w')
f.write(s.encode("utf-8"))
```

No one of us is good enough in Python, we feel that code could be better, but it works like a charm.

