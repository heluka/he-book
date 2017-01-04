一个完整的开发环境主要包括四个部分：标准C库、头文件、工具链、编辑器、帮助文档，下面依次介绍。

## （一）标准C库glibc

glibc是gnu发布的libc库，即c运行库。glibc是linux系统中内核之上最底层的api，几乎其它任何的运行库都会倚赖于glibc。glibc除了封装linux操作系统所提供的系统服务外，它本身也提供了许多其它一些必要功能服务的实现，主要的如下：

* string，字符串处理

* signal，信号处理

* dlfcn，管理共享库的动态加载

* direct，文件目录操作

* elf，共享库的动态加载器，即interpreter

* iconv，不同字符集的编码转换

* inet，socket接口的实现

* intl，国际化，即gettext的实现

* io，基本IO操作

* linuxthreads，线程

* locale，本地化

* login，虚拟终端设备的管理，及系统的安全访问

* malloc，动态内存的分配与管理

* nis stdlib，其它基本功能

基于glibc库的重要性，已经装好的linux系统，基本上都会有glibc库。可以通过如下命令查看系统目前已经安装的glibc库及相关套件

`dpkg -l | grep 'GNU C Library'`

## （二）头文件

许多新手选择自己定制软件包来安装linux，往往会忘记安装头文件等相关的软件包，导致编译时无法通过，报类似下面的错误:

`error: stdio.h: No such file or directory`

可以通过下面的命令安装头文件以及开发库

`apt-get install libc6-dev`

## （三）工具链

从理论上说编译一个程序依次需要下面几个工具：C预处理器--&gt;词法分析器--&gt;代码生成器--&gt;优化器--&gt;汇编程序--&gt;链接器。linux下有两个软件包binutils、gcc包括了上面的所有工具。

### 1. binutils工具

Binutils 是一组很重要的开发工具，包括链接器\(ld\)、汇编器\(as\)、反汇编器\(objdump\)和其他用于目标文件和档案的工具\(ar\)，也是gcc的依赖项。

可以通过下面的命令安装

`apt-get install binutils`

### 2. gcc\(gnu collect compiler\)

gcc完成了从"C预处理器"到"优化器"的工作，并提供了与编译器紧密相关的运行库的支持，如libgcc\_s.so、libstdc++.so等。

其重要性不必再说，没它什么都干不了

`apt-get install gcc`

### 3. make

跟gcc一样大名鼎鼎，也一样重要  
`apt-get install make`

其实上面三项可以通过一个软件包全部安装了，命令是

`apt-get install build-essential`

\(此命令还会将g++等工具也安装进来\)

### 4. auto工具

make相关工具，也非常有必要

`apt-get install autoconf automake`

### 5. 调试器gdb

又是一个必备工具

`apt-get install gdb`

### 6. 其他实用工具

#### 针对源代码

> > indent   C程序美化器
> >
> > ctags    创建标签文件，供vi编辑器使用。可以加快检查源文件的速度
> >
> > lint         C程序检查器

针对可执行文件

> > dis         反汇编工具
> >
> > ldd         打印文件所需的动态连接信息
> >
> > nm         打印目标文件中的符号表
> >
> > strings  查看签入二进制文件中字符串
> >
> > sum      打印文件的校验和与程序块计数
> >
> > size       打印可执行文件中的各个段
> >
> > readelf  分析elf格式的可执行文件\(包括在binutils中\)
> >
> > strip      去除调试信息和符号\(包括在binutils中\)

#### 帮助调试工具

> > strace      打印可执行文件中的系统调用
> >
> > ps          显示进程信息
> >
> > file         检测文件格式

#### 性能优化辅助工具

> > gprof
> >
> > prof
> >
> > time

### 编辑器

选择一个好的编辑器将会大大提高效率，vi是个不错的选择

## （四）帮助文档

帮助文档非常有用，强烈建议大家装上

### 安装C/C++man帮助手册

`apt-get install manpages-dev`

之后就可以查看函式的原型、具体参数说明、include头文件说明等

例如:  ` man 3 printf `查看printf函数的介绍

### 其他文档

`apt-get install binutils-doc gcc-doc glibc-doc `

`apt-get install cpp-doc stl-manual cpp-4.0-doc libstdc++6-4.0-doc`

