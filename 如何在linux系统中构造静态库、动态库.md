# 如何在linux系统中构造静态库、动态库

## （一）基本概念及工具

### 1.共享对象名称（share objects name, SOName\)

理解soname对于linux库编程和linu系统库组织非常重要。各种系统共享库如何组织进我们的应用程序中，就是在gcc编译时加上的各种-lsoname（去掉lib、.so和主副版本号）编译开关，gcc编译的最后阶段用ld通过这些soname来寻找库文件（gcc不过是各种实用工具的集合，会根据程序代码编译的不同阶段和需要，分别调用汇编器as、编译器cc、链接器ld等），在遇到动态库（shared library）时用dynamic loader来通过soname来加载库文件。比如，我们在自己的程序代码中使用了printf来输出，printf的实现位于libc-2.23.so这个动态库中，就需要在gcc编译时加上-lc开关，这个动态库的soname就是libc.so.6:

```
$ldd pic
  linux-gate.so.1 =>  (0xb774c000)
  libpictest.so.1 => ./libpictest.so.1 (0xb7745000)
  libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb756b000)
  /lib/ld-linux.so.2 (0x80039000)
```

```
-rwxr-xr-x 1 root root 1786484 4月  15  2016 /lib/i386-linux-gnu/libc-2.23.so*
lrwxrwxrwx 1 root root      12 10月 20 02:55 /lib/i386-linux-gnu/libc.so.6 -> libc-2.23.so*
```

完整的soname包括库名字和主版本号（比如说libctest.so.1.0的soname为libctest.so.1），因为如果由于代码升级而导致ABI（Application binary interface）变化，共享库的版本号也会变化，linux dynamic loader需要共享库的版本号，以便能够在新旧版本之间有所区别。而对于GNU linker而言，它不需要共享库的版本号，只需要库名字（libctest.so，即部分soname）即可。

为方便使用，一般而是为共享库准备2个符号链接，一个链接名是完整的soname，另一个链接名是部分soname。

### 2.系统工具

与linux库编程相关有的3个系统工具，分别是GNU linker（链接器）、Linux Dynamic Loader（动态加载）和List Dynamic Dependecies（动态链接依赖关系分析器），下面分别介绍：

* 链接器ld：程序代码编译的最后阶段，将各种目标文件链接为一可执行文件，其实就是生成ELF格式的执行文件体。

* 动态加载器（一般是ld-linux.so.2）：在我们自己的程序代码真正执行前，先行将程序所需的各个代码库文件加载到可执行文件的进程虚拟空间，这个动态加载器也是要加载到进程虚拟空间中去的。库文件一般位于3个地方：

> > > * 环境变量LD\_LIBRARY\_PATH所指定的路径
> > >
> > > * 文件/etc/ld.so.conf中所规定的路径，ldconfig会依托该文件，生成/etc/ld.so.cache，系统加载器会到这个缓存中去查找库文件。每次修改/etc/ld.so.conf都需要运行ldconfig来更新该缓存。
> > >
> > > * 系统目录/lib和/usr/lib中。

* 动态链接依赖关系分析器ldd：能够列出可执行文件或共享库所依赖的各个库文件，这个工具在出现诸如“unrefereced symbols...”之类的链接错误时特别有用。

### 3.创建静态库

我们创建两个C语言文件，ctest1.c和ctest2.c，里面各包含1个简单的函数：

ctest1.c

```c
#include <stdio.h>
#include <stdlib.h>

void ctest1(int *i)
{
	*i = 50;

	printf("output from ctest1 \n");

	while(1) {
	    sleep(1000);
	}
}
```

ctest2.c

```c
void ctest2(int *j)
{
	*j = 100;
}
```

这两个文件将被编译为静态库文件：

```
$gcc -Wall -c ctest1.c ctest2.c
$ar rcs libctest.a ctest1.o ctest2.o
```

还有一人使用这个静态库的程序文件cprog.c：

```c
#include <stdio.h>

void ctest1(int *i);
void ctest2(int *j);

int main(int argc, char const *argv[])
{
	int x,y,z;

	ctest1(&x);
	ctest2(&y);

	z = (x/y);
	printf("%d / %d = %d \n",x,y,z );
	return 0;
}
```

编译cprog.c时需要链接libctest.a

```bash
$gcc -static cprog.c -L. -lctest -o cprog
$./cprog
```

程序的执行结果没什么可奇怪的，现在我们看看静态库到底是什么样子的：

用objdump来输出libctest.a，这里用了两个开关：-d表示要进行反汇编disassemble，-S表示要将汇编代码与源代码source一一对照：

```bash
objdump -d -S libctest.a
```

其输出结果：

```
In archive libctest.a:

ctest1.o:     file format elf32-i386


Disassembly of section .text:

00000000 <ctest1>:
   0:	55                   	push   %ebp
   1:	89 e5                	mov    %esp,%ebp
   3:	83 ec 08             	sub    $0x8,%esp
   6:	8b 45 08             	mov    0x8(%ebp),%eax
   9:	c7 00 32 00 00 00    	movl   $0x32,(%eax)
   f:	83 ec 0c             	sub    $0xc,%esp
  12:	68 00 00 00 00       	push   $0x0
  17:	e8 fc ff ff ff       	call   18 <ctest1+0x18>
  1c:	83 c4 10             	add    $0x10,%esp
  1f:	83 ec 0c             	sub    $0xc,%esp
  22:	68 e8 03 00 00       	push   $0x3e8
  27:	e8 fc ff ff ff       	call   28 <ctest1+0x28>
  2c:	83 c4 10             	add    $0x10,%esp
  2f:	eb ee                	jmp    1f <ctest1+0x1f>

ctest2.o:     file format elf32-i386


Disassembly of section .text:

00000000 <ctest2>:
   0:	55                   	push   %ebp
   1:	89 e5                	mov    %esp,%ebp
   3:	8b 45 08             	mov    0x8(%ebp),%eax
   6:	c7 00 64 00 00 00    	movl   $0x64,(%eax)
   c:	90                   	nop
   d:	5d                   	pop    %ebp
   e:	c3                   	ret    

```

这个静态库没有源代码，自然用-S开关也没有源代码一一对照输出。但是仅就汇编代码就能看出：libctest.a就是将两个目标文件的.text段捏合到一起。关于其中详细分析，在后面专门讲。

4. 创建动态库











