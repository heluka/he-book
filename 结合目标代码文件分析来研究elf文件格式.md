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

### 1.ELF为什么由各种段组成？

ELF文件格式很复杂，但就整体而言，主要是由各种不同的“段”组成（“段”这个概念是COFF的首创），这些可以粗略地分为两大类：程序指令和程序数据，高级语言源代码中执行语句被编译为机器指令形成代码段（称为.text），源代码中的初始化的（通过赋值语句与真实数据绑定的）全局变量和局部静态变量形成数据段（称为.data）、未初始化的全局变量和局部静态变量（没有真实数据，只是占位符）形成.bss段（Block Started by Symbol）。.text算程序指令，.data和.bss算程序数据，当然还有其他各种段，但整体上可以分为这两类。

现在问题来了，ELF为什么要分成各种段？单体（monolithic）结构难道不好吗？

原因一：从现代操作系统内存管理的角度看，可执行文件中各个部分的内存访问属性不同，比如：程序指令是只读但可以执行R+X，程序数据部分是可读写但不可执行R/W（有些数据也是只读的，比如.rodata段），这样将具备不同访问属性的部分分别加载到不同的虚拟内存区域，其读写权限可以分别设置，可以在硬件层次上防止指令篡改或将数据误作指令执行，有利于实现程序执行期保护。

原因二：从现代CPU指令流水线结构角度来看，其着眼于有效利用程序的局部性而设置了不同层次的缓存，而且通常都设计了相互分离的数据缓存和指令缓存，这样程序指令与数据的分离有助于充分利用现代CPU的指令流水线特性，而加快执行速度。

原因三：从可执行程序执行共享的角度来看，当系统中运行着多个程序副本时，内存中只要保存该程序指令部分（包括其他只读数据，比如图标、图片和文本资源等）的一份拷贝，而该程序的数据部分可能是各个进程私有的（极有可能是以copy-on-write方式实现私有），这样可以节省大量内存。

### 2.ELF头部的作用是什么？

当我们从命令行或图形界面中启动某个可执行程序A时，最终是由execve系统调用(system call)来接手负责，当前（父）进程的虚拟内存空间最终要由A可执行文件中的内容所代替，这个过程不仅仅用mmap内存映射将文件内容拷贝到虚存中相应位置即可，ELF是由不同内存访问属性的各种不同段组成的结构化文件，系统加载器的第一步就是要得到ELF文件中各个段的信息。

![](/assets/elf-structure.png)

如图所示，标识出了可重定位文件和可执行文件所涉及的所有结构。其中**程序头表（Program Header Table, PHT）**是可执行文件独有的，**重定位表段（.rel.text和.rel.data）**是可重定位文件独有的结构。右侧标识了对应结构一般的大小（N为非负整数）。

除却通常情况下的ELF基本属性描述功能之外（版本、目标机器型号、程序入口地址等），“ELF头”最重要的作用在于指出“ELF段表”（section header table）的位置，这个“段表”描述了ELF文件中所有段的信息，包括段名、段长、在可执行文件中的偏移量、读写属性等。还有一点要特别提醒注意：“ELF头”是ELF可执行文件格式中唯一一个位置固定的描述性数据结构，即从ELF文件0偏移量开始（从文件头开始）。

####程序入口地址各不相同

首先用readelf工具来观察一下ELF文件头：


```
heluka >>> readelf -h cprog | tee sec-out 
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x80484f0
  Start of program headers:          52 (bytes into file)
  Start of section headers:          7036 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         9
  Size of section headers:           40 (bytes)
  Number of section headers:         36
  Section header string table index: 33
```




