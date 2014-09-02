---
layout: post
title: 编译levent
---

{{ page.title }}
================
当然，首先需要下载levent，[地址](https://github.com/xjdrew/levent)。

另外，levent使用cmake生成合适的Makefile。编译之前，需要先安装较新版本的cmake。

linux && mac
-------------
*nux系统一般都有免费的编译工具，可以根据情况使用gcc, clang等编译。

有了编译工具后， 两步即可完成编译：

1. 下载[lua 5.2.3](http://www.lua.org/ftp/lua-5.2.3.tar.gz)以上版本, 编译安装在默认路径。
2. 到levent根目录使用```cmake .```生成Makefile，再```make```即可。


windows
-------
windows系统下面有mingw, cygwin等编译环境可选，这里只介绍使用vs的编译步骤。

首先你需要有一个vs，我用的是vs2010，下面都按照vs2010的设定来编译。

1. 编译lua
由于lua语言本身没有提供windows编译支持，需要先下载已经编译的lua library和lua程序（也可以自己编译lua）。[lua.org](http://lua-users.org/wiki/LuaBinaries)上面提供了一些可选的预编译库。我使用了[LuaBinaries](http://luabinaries.sourceforge.net/)，这个项目提供了lua5.1-lua5.2 32/64bit的各个vs版本的预编译文件。根据系统情况，选用合适的[lua librarys](http://sourceforge.net/projects/luabinaries/files/5.2.3/Windows%20Libraries/Dynamic/) 和[lua tools](http://sourceforge.net/projects/luabinaries/files/5.2.3/Tools%20Executables/)，我选择了 lua-5.2.3_Win32_dll10_lib.zip 和 lua-5.2.3_Win32_bin.zip 。把lua-5.2.3_Win32_dll10_lib.zip解压后的所有文件拷贝到levent/deps/lua/win/下面。

2. 生成sln
步骤跟linux类似，到levent根目录下面，```cmake .```，会生成合适的vcproj和sln文件。

3. 编译
根据你的情况微调vcproj，比如64bit，生成文件的目录等。默认会编译出32bit的dll，并放置在Debug目录下。编译成功之后，把Debug目录下生成的levent.dll, struct.dll拷贝到levent根目录下，方便使用。

test
-----
levent提供了tests和examples，可以使用下面两个用例来验证编译成功：

* socket

```
lua tests/test_socket.lua
```

* dns query

```
lua example/dns_mass_resolve.lua
```

feedback
---------
有问题或需要讨论请提[issue](https://github.com/xjdrew/levent/issues)。

