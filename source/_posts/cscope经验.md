title: cscope操作经验
author: Joy Xu
date: 2020-02-07 10:12:07
tags: cscope
---

#!/bin/bash

CSCOPE_DIR="$PWD/cscope"

if [ ! -d "$CSCOPE_DIR" ]; then
mkdir "$CSCOPE_DIR"
fi

echo "Finding files ..."
find "$PWD" -name '*.[ch]' \
-o -name '*.java' \
-o -name '*properties' \
-o -name '*.cpp' \
-o -name '*.cc' \
-o -name '*.hpp' \
-o -name '*.py' \
-o -name '*.php' > "$CSCOPE_DIR/cscope.files"

echo "Adding files to cscope db: $PWD/cscope.db ..."
cscope -b -i "$CSCOPE_DIR/cscope.files"

export CSCOPE_DB="$PWD/cscope.out"
echo "Exported CSCOPE_DB to: '$CSCOPE_DB'"


https://www.embeddedarm.com/blog/tag-jumping-in-a-codebase-using-ctags-and-cscope-in-vim/

