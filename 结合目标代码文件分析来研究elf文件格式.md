# 目标代码格式研究

目标文件是源代码文件经编译器编译后得到的，其中的内容称为目标代码。目标文件从结构上讲，就是可执行文件格式（在\*NIX系列系统中是ELF格式，在windows系统中是PE格式），只不过其中有些符号和地址（特别是访问外部数据或调用外部函数）还没有被调整，这个调整工作是由“链接器”完成的。

## （一）ELF格式分类

ELF共有4种格式：

* **可重定位文件**。就是linux平台的.o文件、windows平台的.obj文件，这些文件中包含了代码和数据，只需要经过链接就可以成为可执行文件或共享库文件。静态库实际上就是将很多目标文件打包形成的，也可归为这类。

* **可执行文件**，linux平台一般没有扩展名，window平台的.exe文件，可以直接由系统加载执行。

* **共享目标文件（shared object file）**，就是linux平台的.so文件、windows平台的.dll文件，这些文件中包含有代码的数据，可以由链接器与其他共享目标文件、可重定位文件链接，形成新的目标文件，也可以与可执行文件结合，作为进程映像的一部分执行。

* **核心转储文件（core dump）**，进程意外终止，系统将该进程地址空间内容及终止时的系统信息记录下来形成的。

在linux系统中，可以用file命令来查看是什么类型的ELF文件（这个命令虽然看着不起眼，实际上很有用，许多时候还没搞清是什么文件，就急勿勿上手分析）：

```
$file ctest1.o
ctest1.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV),
not stripped

$file libctest.a
libctest.a: current ar archive

$file libctest.so.1.0
libctest.so.1.0: ELF 32-bit LSB shared object, Intel 80386, 
version 1 (SYSV), dynamically linked, BuildID[sha1]=bbd62f0
6ba2cc9bdfc76eb736e01b94642942776, not stripped

$file cprog
cprog: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV),
dynamically linked, interpreter /lib/ld-linux.so.2, 
for GNU/Linux 2.6.32, BuildID[sha1]=f0342b1518b3b6a6693d7c41420c
fc6c08ee5041, not stripped
```

目标文件格式深受操作系统和编译器影响，unix系统最早的可执行文件格式是a.out，其结构很简单，后来由于共享库的出现，其复杂的链接、命名、访问及共享等式问题使得a.out捉襟见肘，所以设计了COFF格式来解决问题。COFF最早是unix system V release 3提出并使用的目标文件及可执行文件格式规范，后来Microsoft基于COFF搞了PE并用于windows NT；而system V release 4也在COFF基础上形成了ELF。

## （二）ELF格式为什么是这样？

ELF文件格式很复杂，但就整体而言，主要是由各种不同的“段”组成（“段”这个概念是COFF的首创），这些可以粗略地分为两大类：程序指令和程序数据，高级语言源代码中执行语句被编译为机器指令形成代码段（称为.text），源代码中的初始化的（通过赋值语句与真实数据绑定的）全局变量和局部静态变量形成数据段（称为.data）、未初始化的全局变量和局部静态变量（没有真实数据，只是占位符）形成.bss段（Block Started by Symbol）。.text算程序指令，.data和.bss算程序数据，当然还有其他各种段，但整体上可以分为这两类。

现在问题来了，ELF为什么要分成各种段？单体（monolithic）结构难道不好吗？

原因一：从现代操作系统内存管理的角度看，可执行文件中各个部分的内存访问属性不同，比如：程序指令是只读但可以执行R+X，程序数据部分是可读写但不可执行R/W（有些数据也是只读的，比如.rodata段），这样将具备不同访问属性的部分分别加载到不同的虚拟内存区域，其读写权限可以分别设置，可以在硬件层次上防止指令篡改或将数据误作指令执行，有利于实现程序执行期保护。

原因二：从现代CPU指令流水线结构角度来看，其着眼于有效利用程序的局部性而设置了不同层次的缓存，而且通常都设计了相互分离的数据缓存和指令缓存，这样程序指令与数据的分离有助于充分利用现代CPU的指令流水线特性，而加快执行速度。

原因三：从可执行程序执行共享的角度来看，当系统中运行着多个程序副本时，内存中只要保存该程序指令部分（包括其他只读数据，比如图标、图片和文本资源等）的一份拷贝，而该程序的数据部分可能是各个进程私有的（极有可能是以copy-on-write方式实现私有），这样可以节省大量内存。



