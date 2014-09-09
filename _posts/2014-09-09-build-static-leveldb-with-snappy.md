---
layout: post
title: 编译leveldb+snappy静态库
---

{{ page.title }}
================

1. 获取最新代码[leveldb](https://github.com/google/leveldb), [snappy](https://github.com/google/snappy), 

2. 编译snappy
```
cd ./snappy
./autogen.sh
./configure --disable-shared --with-pic
make
```
成功编译之后，会生成文件```.libs/libsnappy.a```。

3. 把snappy放到编译环境变量中
```
SNAPPY_PATH=`cd ./snappy; pwd`
export LIBRARY_PATH=${SNAPPY_PATH}/.libs
export C_INCLUDE_PATH=${SNAPPY_PATH}
export CPLUS_INCLUDE_PATH=${SNAPPY_PATH}
```

4. 编译leveldb
```
cd ./leveldb
make
```
成功编译之后，会生成文件```libleveldb.a```文件

5. 使用
把libleveldb.a, libsnappy.a拷贝到需要使用的工程，添加链接选项```-lleveldb -lsnappy -lstdc++```，编译即可
