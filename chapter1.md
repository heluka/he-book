第一个手工汇编程序：

![](file:///tmp/WizNote/c2b60eeb-d8fa-42fd-b136-ccb0d6601688/index_files/adf96ad9-125c-4db5-8a4e-645b2f88908a.png)

其中用.section命令（这种前面有.的命令称为“伪指令”，assembler directives orpseudo-operations，这种指令不会被翻译为机器代码）来明确汇编程序的“段”，.section .data后面是空的，说明目前数据段是空的  


.section .text表明代码段开始，首先符号\_start放到.global全局符号表段中，这个\_start符号很重要 ，linux系统加载我们的程序，就需要从这个符号中得到程序文件的执行起始地址。

\_start:是汇编语言中的标号label，label是由带有冒号：的符号symbol组成。在使用汇编器（linux中是as）来汇编代码时，会为代码中每个数据项赋值、为每条指令指定内存地址，同时将第每个label的值指定为label后面指令的地址，或是label后面数据的值。

这段代码是实现了程序退出指令，类似执行C语言的exit函数。这个退出是依靠调用system call进行。可以将system call想象为一道门，通过这道门我们可以进行linux系统内核，调用系统功能为我所用，就比如说这个退出指令，实际上是包含了用户程序向linux系统转移cpu控制权的过程，这个过程只能由内核完成，要使用内核中的相应功能，就只能通过system call。

所有的system call都是用int指令，即interrupt中断指令，中断号是0x80，在汇编代码中所有的直接数immediate number都必须使用$符号，不然会被as认定是内存地址。不同的system call的区别就在于eax寄存器中的调用号不同，退出是1号调用。

程序的返回值是放在ebx寄存器中，这是约定，没有什么理由可讲。这个返回值在shell中使用echo $?查看 。



