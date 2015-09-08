# Gcc 编译的背后

-    [前言](#toc_27212_14734_1)
-    [预处理](#toc_27212_14734_2)
    -    [简述](#toc_27212_14734_3)
    -    [打印出预处理之后的结果](#toc_27212_14734_4)
    -    [在命令行定义宏](#toc_27212_14734_5)
-    [编译（翻译）](#toc_27212_14734_6)
    -    [简述](#toc_27212_14734_7)
    -    [语法检查](#toc_27212_14734_8)
    -    [编译器优化](#toc_27212_14734_9)
    -    [生成汇编语言文件](#toc_27212_14734_10)
-    [汇编](#toc_27212_14734_11)
    -    [简述](#toc_27212_14734_12)
    -    [生成目标代码](#toc_27212_14734_13)
    -    [ELF 文件初次接触](#toc_27212_14734_14)
    -    [ELF 文件的结构](#toc_27212_14734_15)
    -    [三种不同类型 ELF 文件比较](#toc_27212_14734_16)
    -    [ELF 主体：节区](#toc_27212_14734_17)
    -    [汇编语言文件中的节区表述](#toc_27212_14734_18)
-    [链接](#toc_27212_14734_19)
    -    [简述](#toc_27212_14734_20)
    -    [可执行文件的段：节区重排](#toc_27212_14734_21)
    -    [链接背后的故事](#toc_27212_14734_22)
    -    [用 ld 完成链接过程](#toc_27212_14734_23)
    -    [C++ 构造与析构：crtbegin.o 和 crtend.o](#toc_27212_14734_24)
    -    [初始化与退出清理：crti.o 和 crtn.o](#toc_27212_14734_25)
    -    [C 语言程序真正的入口](#toc_27212_14734_26)
    -    [链接脚本初次接触](#toc_27212_14734_27)
-    [参考资料](#toc_27212_14734_28)


<span id="toc_27212_14734_1"></span>
## 前言

平时在 Linux 下写代码，直接用 `gcc -o out in.c` 就把代码编译好了，但是这背后到底做了什么呢？

如果学习过《[编译原理](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)》则不难理解，一般高级语言程序编译的过程莫过于：预处理、编译、汇编、链接。

`gcc` 在后台实际上也经历了这几个过程，可以通过 `-v` 参数查看它的编译细节，如果想看某个具体的编译过程，则可以分别使用 `-E`，`-S`，`-c` 和 `-O`，对应的后台工具则分别为 `cpp`，`cc1`，`as`，`ld`。

下面将逐步分析这几个过程以及相关的内容，诸如语法检查、代码调试、汇编语言等。

<span id="toc_27212_14734_2"></span>
## 预处理

<span id="toc_27212_14734_3"></span>
### 简述

预处理是 C 语言程序从源代码变成可执行程序的第一步，主要是 C 语言编译器对各种预处理命令进行处理，包括头文件的包含、宏定义的扩展、条件编译的选择等。

以前没怎么“深入”预处理，脑子对这些东西总是很模糊，只记得在编译的基本过程（词法分析、语法分析）之前还需要对源代码中的宏定义、文件包含、条件编译等命令进行处理。这三类的指令很常见，主要有 `#define`，`#include`和 `#ifdef ... #endif`，要特别地注意它们的用法。

`#define` 除了可以独立使用以便灵活设置一些参数外，还常常和 `#ifdef ... #endif` 结合使用，以便灵活地控制代码块的编译与否，也可以用来避免同一个头文件的多次包含。关于 `#include` 貌似比较简单，通过 `man` 找到某个函数的头文件，复制进去，加上 `<>` 就好。这里虽然只关心一些技巧，不过预处理还是隐藏着很多潜在的陷阱（可参考《[C Traps & Pitfalls](https://en.wikipedia.org/wiki/C_Traps_and_Pitfalls)》）也是需要注意的。下面仅介绍和预处理相关的几个简单内容。

<span id="toc_27212_14734_4"></span>
### 打印出预处理之后的结果

```
$ gcc -E hello.c
```

这样就可以看到源代码中的各种预处理命令是如何被解释的，从而方便理解和查错。

实际上 `gcc` 在这里调用了 `cpp`（虽然通过 `gcc -v` 仅看到 `cc1`)，`cpp` 即 The C Preprocessor，主要用来预处理宏定义、文件包含、条件编译等。下面介绍它的一个比较重要的选项 `-D`。

<span id="toc_27212_14734_5"></span>
### 在命令行定义宏

```
$ gcc -Dmacro hello.c
```

这个等同于在文件的开头定义宏，即 `#define macro`，但是在命令行定义更灵活。例如，在源代码中有这些语句。

```
#ifdef DEBUG
printf("this code is for debugging\n");
#endif
```

如果编译时加上 `-DDEBUG` 选项，那么编译器就会把 `printf` 所在的行编译进目标代码，从而方便地跟踪该位置的某些程序状态。这样 `-DDEBUG` 就可以当作一个调试开关，编译时加上它就可以用来打印调试信息，发布时则可以通过去掉该编译选项把调试信息去掉。

<span id="toc_27212_14734_6"></span>
## 编译（翻译）

<span id="toc_27212_14734_7"></span>
### 简述

编译之前，C 语言编译器会进行词法分析、语法分析，接着会把源代码翻译成中间语言，即汇编语言。如果想看到这个中间结果，可以用 `gcc -S`。需要提到的是，诸如 Shell 等解释语言也会经历一个词法分析和语法分析的阶段，不过之后并不会进行“翻译”，而是“解释”，边解释边执行。

把源代码翻译成汇编语言，实际上是编译的整个过程中的第一个阶段，之后的阶段和汇编语言的开发过程没有什么区别。这个阶段涉及到对源代码的词法分析、语法检查（通过 `-std` 指定遵循哪个标准），并根据优化（`-O`）要求进行翻译成汇编语言的动作。

<span id="toc_27212_14734_8"></span>
### 语法检查

如果仅仅希望进行语法检查，可以用 `gcc` 的 `-fsyntax-only` 选项；如果为了使代码有比较好的可移植性，避免使用 `gcc` 的一些扩展特性，可以结合 `-std` 和 `-pedantic`（或者 `-pedantic-erros` ）选项让源代码遵循某个 C 语言标准的语法。这里演示一个简单的例子：

```
$ cat wrong.c
#include <stdio.h>
int main()
{
	printf("hello, world\n")
	return 0;
}
$ gcc -fsyntax-only wrong.c
wrong.c: In function ‘main’:
wrong.c:5: error: expected ‘;’ before ‘return’
$ vim wrong.c
$ cat wrong.c
#include <stdio.h>
int main()
{
        printf("hello, world\n");
        int i;
        return 0;
}
$ gcc -pedantic-errors wrong.c    #默认情况下，gcc是允许在程序中间声明变量
wrong.c: In function ‘main’:
wrong.c:5: error: ISO C90 forbids mixed declarations and code [-Wpedantic]
```

语法错误是程序开发过程中难以避免的错误（人的大脑在很多情况下都容易开小差），不过编译器往往能够通过语法检查快速发现这些错误，并准确地告知语法错误的大概位置。因此，作为开发人员，要做的事情不是“恐慌”（不知所措），而是认真阅读编译器的提示，根据平时积累的经验（最好总结一份常见语法错误索引，很多资料都提供了常见语法错误列表，如《[C Traps & Pitfalls](https://en.wikipedia.org/wiki/C_Traps_and_Pitfalls)》和编辑器提供的语法检查功能（语法加亮、括号匹配提示等）快速定位语法出错的位置并进行修改。

<span id="toc_27212_14734_9"></span>
### 编译器优化

语法检查之后就是翻译动作，`gcc` 提供了一个优化选项 `-O`，以便根据不同的运行平台和用户要求产生经过优化的汇编代码。例如，

```
$ cat eval.c
int main()
{
    int a = 11, b = 13, c = 5566, j;
    int i, result;
    for (j = 0; j < 10000; j++)
        for (i = 0; i < 10000; i++)
            result = (a * b + i) / (a * c);
    return result;
}
$ gcc -o eval eval.c         # 采用默认选项，不优化
$ gcc -O2 -o eval2 eval.c    # 优化等次是2
$ time ./eval

real    0m0.346s
user    0m0.341s
sys     0m0.000s
$ time ./eval2

real    0m0.001s
user    0m0.000s
sys     0m0.001s
```

根据上面的简单演示，可以看出 `gcc` 有很多不同的优化选项，主要看用户的需求了，目标代码的大小和效率之间貌似存在一个“纠缠”，需要开发人员自己权衡。

<span id="toc_27212_14734_10"></span>
### 生成汇编语言文件

下面通过 `-S` 选项来看看编译出来的中间结果：汇编语言，以 `hello.c` 为例。

```
$ cat hello.c
#include <stdio.h>
int main()
{
    printf("hello, world");
    return 0;
}
$ gcc -S hello.c  # 默认输出是hello.s，可自己指定，输出到屏幕`-o -`，输出到其他文件`-o file`
$ cat hello.s
        .file   "hello.c"
        .section        .rodata
.LC0:
        .string "hello, world"
        .text
        .globl  main
        .type   main, @function
main:
.LFB0:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        movl    $.LC0, %edi
        movl    $0, %eax
        call    printf
        movl    $0, %eax
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   main, .-main
        .ident  "GCC: (Ubuntu 4.8.4-2ubuntu1~14.04) 4.8.4"
        .section        .note.GNU-stack,"",@progbits
```

不知道看出来没？和课堂里学的 intel 的汇编语法不太一样，这里用的是 `AT&T` 语法格式。如果想学习 Linux 下的汇编语言开发，下一节开始的所有章节基本上覆盖了 Linux 下汇编语言开发的一般过程，不过这里不介绍汇编语言语法。

在学习后面的章节之前，建议自学旧金山大学的[微机编程课程 CS 630](http://www.cs.usfca.edu/~cruse/cs630f06/)，该课深入介绍了 Linux/X86 平台下的 `AT&T` 汇编语言开发。如果想在 `Qemu` 上做这个课程里的实验，可以阅读本文作者写的 [CS630: Linux 下通过 Qemu 学习 X86 AT&T 汇编语言](http://www.tinylab.org/cs630-qemu/)。

需要补充的是，在写 C 语言代码时，如果能够对编译器比较熟悉（工作原理和一些细节）的话，可能会很有帮助。包括这里的优化选项（有些优化选项可能在汇编时采用）和可能的优化措施，例如字节对齐、条件分支语句裁减（删除一些明显分支）等。

<span id="toc_27212_14734_11"></span>
## 汇编

<span id="toc_27212_14734_12"></span>
### 简述

汇编实际上还是翻译过程，只不过把作为中间结果的汇编代码翻译成了机器代码，即目标代码，不过它还不可以运行。如果要产生这一中间结果，可用 `gcc -c`，当然，也可通过 `as` 命令处理汇编语言源文件来产生。

汇编是把汇编语言翻译成目标代码的过程，如果有在 Windows 下学习过汇编语言开发，大家应该比较熟悉 `nasm` 汇编工具(支持 Intel 格式的汇编语言)，不过这里主要用 `as` 汇编工具来汇编 `AT&T` 格式的汇编语言，因为 `gcc` 产生的中间代码就是 `AT&T` 格式的。

<span id="toc_27212_14734_13"></span>
### 生成目标代码

下面来演示分别通过 `gcc -c` 选项和 `as` 来产生目标代码。

```
$ file hello.s
hello.s: ASCII assembler program text
$ gcc -c hello.s   #用gcc把汇编语言编译成目标代码
$ file hello.o     #file命令用来查看文件类型，目标代码可重定位的(relocatable)，
                   #需要通过ld进行进一步链接成可执行程序(executable)和共享库(shared)
hello.o: ELF 64-bit LSB  relocatable, x86-64, version 1 (SYSV), not stripped
$ as -o hello.o hello.s        #用as把汇编语言编译成目标代码
$ file hello.o
hello.o: ELF 64-bit LSB  relocatable, x86-64, version 1 (SYSV), not stripped
```

`gcc` 和 `as` 默认产生的目标代码都是 ELF 格式的，因此这里主要讨论ELF格式的目标代码（如果有时间再回顾一下 `a.out` 和 `coff` 格式，当然也可以先了解一下，并结合 `objcopy` 来转换它们，比较异同)。

<span id="toc_27212_14734_14"></span>
### ELF 文件初次接触

目标代码不再是普通的文本格式，无法直接通过文本编辑器浏览，需要一些专门的工具。如果想了解更多目标代码的细节，区分 `relocatable`（可重定位）、`executable`（可执行）、`shared libarary`（共享库）的不同，我们得设法了解目标代码的组织方式和相关的阅读和分析工具。下面主要介绍这部分内容。

> BFD is a package which allows applications to use the same routines to
operate on object files whatever the object file format. A new object file
format can be supported simply by creating a new BFD back end and adding it to
the library.

binutils（GNU Binary Utilities）的很多工具都采用这个库来操作目标文件，这类工具有 `objdump`，`objcopy`，`nm`，`strip` 等（当然，我们也可以利用它。如果深入了解ELF格式，那么通过它来分析和编写 Virus 程序将会更加方便），不过另外一款非常优秀的分析工具 `readelf` 并不是基于这个库，所以也应该可以直接用 `elf.h` 头文件中定义的相关结构来操作 ELF 文件。

下面将通过这些辅助工具（主要是 `readelf` 和 `objdump`），结合 ELF 手册来分析它们。将依次介绍 ELF 文件的结构和三种不同类型 ELF 文件的区别。

<span id="toc_27212_14734_15"></span>
### ELF 文件的结构

```
ELF Header(ELF文件头)
Program Headers Table(程序头表，实际上叫段表好一些，用于描述可执行文件和可共享库)
Section 1
Section 2
Section 3
...
Section Headers Table(节区头部表，用于链接可重定位文件成可执行文件或共享库)
```

对于可重定位文件，程序头是可选的，而对于可执行文件和共享库文件（动态链接库），节区表则是可选的。可以分别通过 `readelf` 文件的 `-h`，`-l` 和 `-S` 参数查看 ELF 文件头（ELF Header）、程序头部表（Program Headers Table，段表）和节区表（Section Headers Table）。

文件头说明了文件的类型，大小，运行平台，节区数目等。

<span id="toc_27212_14734_16"></span>
### 三种不同类型 ELF 文件比较

先来通过文件头看看不同ELF的类型。为了说明问题，先来几段代码吧。

```
/* myprintf.c */

#include <stdio.h>
void myprintf(void)
{
    printf("hello, world!\n");
}
```

```
/* test.h -- myprintf function declaration */

#ifndef _TEST_H_
#define _TEST_H_
void myprintf(void);
#endif
```

```
/* test.c */

#include "test.h"
int main()
{
    myprintf();
    return 0;
}
```


下面通过这几段代码来演示通过 `readelf -h` 参数查看 ELF 的不同类型。期间将演示如何创建动态链接库（即可共享文件）、静态链接库，并比较它们的异同。


编译产生两个目标文件 `myprintf.o` 和 `test.o`，它们都是可重定位文件（REL）：

```
$ gcc -c myprintf.c test.c
$ readelf -h test.o | grep Type
  Type:                              REL (Relocatable file)
$ readelf -h myprintf.o | grep Type
  Type:                              REL (Relocatable file)
```

根据目标代码链接产生可执行文件，这里的文件类型是可执行的(EXEC)：

```
$ gcc -o test myprintf.o test.o
$ readelf -h test | grep Type
  Type:                              EXEC (Executable file)
```

用 `ar` 命令创建一个静态链接库，静态链接库也是可重定位文件（REL）：

```
$ ar rcsv libmyprintf.a myprintf.o
$ readelf -h libmyprintf.a | grep Type
  Type:                              REL (Relocatable file)
```

可见，静态链接库和可重定位文件类型一样，它们之间唯一不同是前者可以是多个可重定位文件的“集合”。

静态链接库可直接链接（只需库名，不要前面的 `lib`），也可用 `-l` 参数，`-L` 指定库搜索路径。

```
$ gcc -o test test.o -lmyprintf -L./
```

编译产生动态链接库，并支持 `major` 和 `minor` 版本号，动态链接库类型为 `DYN`：

```
$ gcc -fPIC -c myprintf.c
$ gcc -Wall myprintf.o -shared -Wl,-soname,libmyprintf.so.0 -o libmyprintf.so.0.0
$ ln -sf libmyprintf.so.0.0 libmyprintf.so.0
$ ln -sf libmyprintf.so.0 libmyprintf.so
$ readelf -h libmyprintf.so | grep Type
  Type:                              DYN (Shared object file)
```

动态链接库编译时和静态链接库类似：

```
$ gcc -o test test.o -lmyprintf -L./
```

但是执行时需要指定动态链接库的搜索路径，把 `LD_LIBRARY_PATH` 设为当前目录，指定 `test` 运行时的动态链接库搜索路径：

```
$ LD_LIBRARY_PATH=./ ./test
$ gcc -static -o test test.o -lmyprintf -L./
```

在不指定 `-static` 时会优先使用动态链接库，指定时则阻止使用动态链接库，这时会把所有静态链接库文件加入到可执行文件中，使得执行文件很大，而且加载到内存以后会浪费内存空间，因此不建议这么做。

经过上面的演示基本可以看出它们之间的不同：

- 可重定位文件本身不可以运行，仅仅是作为可执行文件、静态链接库（也是可重定位文件）、动态链接库的 “组件”。
- 静态链接库和动态链接库本身也不可以执行，作为可执行文件的“组件”，它们两者也不同，前者也是可重定位文件（只不过可能是多个可重定位文件的集合），并且在链接时加入到可执行文件中去。
- 而动态链接库在链接时，库文件本身并没有添加到可执行文件中，只是在可执行文件中加入了该库的名字等信息，以便在可执行文件运行过程中引用库中的函数时由动态链接器去查找相关函数的地址，并调用它们。

从这个意义上说，动态链接库本身也具有可重定位的特征，含有可重定位的信息。对于什么是重定位？如何进行静态符号和动态符号的重定位，我们将在链接部分和[《动态符号链接的细节》][100]一节介绍。

[100]: 02-chapter4.markdown

<span id="toc_27212_14734_17"></span>
### ELF 主体：节区

下面来看看 ELF 文件的主体内容：节区（Section)。

ELF 文件具有很大的灵活性，它通过文件头组织整个文件的总体结构，通过节区表 (Section Headers Table）和程序头（Program Headers Table 或者叫段表）来分别描述可重定位文件和可执行文件。但不管是哪种类型，它们都需要它们的主体，即各种节区。

在可重定位文件中，节区表描述的就是各种节区本身；而在可执行文件中，程序头描述的是由各个节区组成的段（Segment），以便程序运行时动态装载器知道如何对它们进行内存映像，从而方便程序加载和运行。

下面先来看看一些常见的节区，而关于这些节区（Section）如何通过重定位构成不同的段（Segments），以及有哪些常规的段，我们将在链接部分进一步介绍。

可以通过 `readelf -S` 查看 ELF 的节区。（建议一边操作一边看文档，以便加深对 ELF 文件结构的理解）先来看看可重定位文件的节区信息，通过节区表来查看：

默认编译好 `myprintf.c`，将产生一个可重定位的文件 `myprintf.o`，这里通过 `myprintf.o` 的节区表查看节区信息。

```
$ gcc -c myprintf.c
$ readelf -S myprintf.o
There are 13 section headers, starting at offset 0x128:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000010  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000590
       0000000000000030  0000000000000018          11     1     8
  [ 3] .data             PROGBITS         0000000000000000  00000050
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] .bss              NOBITS           0000000000000000  00000050
       0000000000000000  0000000000000000  WA       0     0     1
  [ 5] .rodata           PROGBITS         0000000000000000  00000050
       000000000000000e  0000000000000000   A       0     0     1
...
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```

用 `objdump -d` 可看反编译结果，用 `-j` 选项可指定需要查看的节区：

```
$ objdump -d -j .text   myprintf.o
myprintf.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <myprintf>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	bf 00 00 00 00       	mov    $0x0,%edi
   9:	e8 00 00 00 00       	callq  e <myprintf+0xe>
   e:	5d                   	pop    %rbp
   f:	c3                   	retq
```

用 `-r` 选项可以看到有关重定位的信息，这里有两部分需要重定位：

```
$ readelf -r myprintf.o

Relocation section '.rela.text' at offset 0x590 contains 2 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000005  00050000000a R_X86_64_32       0000000000000000 .rodata + 0
00000000000a  000a00000002 R_X86_64_PC32     0000000000000000 puts - 4
```

`.rodata` 节区包含只读数据，即我们要打印的 `hello, world!`

```
$ readelf -x .rodata myprintf.o

Hex dump of section '.rodata':
  0x00000000 68656c6c 6f2c2077 6f726c64 2100     hello, world!.
```

没有找到 `.data` 节区, 它应该包含一些初始化的数据：

```
$ readelf -x .data myprintf.o

Section '.data' has no data to dump.
```

也没有 `.bss` 节区，它应该包含一些未初始化的数据，程序默认初始为 0：

```
$ readelf -x .bss myprintf.o

Section '.bss' has no data to dump.
```

`.comment` 是一些注释，可以看到是是 `Gcc` 的版本信息

```
$ readelf -x .comment myprintf.o

Hex dump of section '.comment':
  0x00000000 00474343 3a202855 62756e74 7520342e .GCC: (Ubuntu 4.
  0x00000010 382e342d 32756275 6e747531 7e31342e 8.4-2ubuntu1~14.
  0x00000020 30342920 342e382e 3400              04) 4.8.4.
```

`.note.GNU-stack` 这个节区也没有内容：

```
$ readelf -x .note.GNU-stack myprintf.o

Section '.note.GNU-stack' has no data to dump.
```

`.shstrtab` 包括所有节区的名字：

```
$ readelf -x .shstrtab myprintf.o

Hex dump of section '.shstrtab':
  0x00000000 002e7379 6d746162 002e7374 72746162 ..symtab..strtab
  0x00000010 002e7368 73747274 6162002e 72656c61 ..shstrtab..rela
  0x00000020 2e746578 74002e64 61746100 2e627373 .text..data..bss
  0x00000030 002e726f 64617461 002e636f 6d6d656e ..rodata..commen
  0x00000040 74002e6e 6f74652e 474e552d 73746163 t..note.GNU-stac
  0x00000050 6b002e72 656c612e 65685f66 72616d65 k..rela.eh_frame
  0x00000060 00
```

符号表 `.symtab` 包括所有用到的相关符号信息，如函数名、变量名，可用 `readelf` 查看：

```
$ readelf -s myprintf.o

Symbol table '.symtab' contains 11 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS myprintf.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    8
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT    6
     9: 0000000000000000    16 FUNC    GLOBAL DEFAULT    1 myprintf
    10: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND puts
```

字符串表 `.strtab` 包含用到的字符串，包括文件名、函数名、变量名等：

```
$ readelf -x .strtab myprintf.o

Hex dump of section '.strtab':
  0x00000000 006d7970 72696e74 662e6300 6d797072 .myprintf.c.mypr
  0x00000010 696e7466 00707574 7300              intf.puts.
```

从上表可以看出，对于可重定位文件，会包含这些基本节区 `.text`, `.rel.text`, `.data`, `.bss`, `.rodata`, `.comment`, `.note.GNU-stack`, `.shstrtab`, `.symtab` 和 `.strtab`。

<span id="toc_27212_14734_18"></span>
### 汇编语言文件中的节区表述

为了进一步理解这些节区和源代码的关系，这里来看一看 `myprintf.c` 产生的汇编代码。

```
$ gcc -S myprintf.c
$ cat myprintf.s
	.file	"myprintf.c"
	.section	.rodata
.LC0:
	.string	"hello, world!"
	.text
	.globl	myprintf
	.type	myprintf, @function
myprintf:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movl	$.LC0, %edi
	call	puts
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE0:
	.size	myprintf, .-myprintf
	.ident	"GCC: (Ubuntu 4.8.4-2ubuntu1~14.04) 4.8.4"
	.section	.note.GNU-stack,"",@progbits
```

是不是可以从中看出可重定位文件中的那些节区和汇编语言代码之间的关系？在上面的可重定位文件，可以看到有一个可重定位的节区，即 `.rel.text`，它标记了两个需要重定位的项，`.rodata` 和 `puts`。这个节区将告诉编译器这两个信息在链接或者动态链接的过程中需要重定位， 具体如何重定位？将根据重定位项的类型，比如上面的 `R_386_32` 和 `R_386_PC32`。

到这里，对可重定位文件应该有了一个基本的了解，下面将介绍什么是可重定位，可重定位文件到底是如何被链接生成可执行文件和动态链接库的，这个过程除了进行一些符号的重定位外，还进行了哪些工作呢？

<span id="toc_27212_14734_19"></span>
## 链接

<span id="toc_27212_14734_20"></span>
### 简述

重定位是将符号引用与符号定义进行链接的过程。因此链接是处理可重定位文件，把它们的各种符号引用和符号定义转换为可执行文件中的合适信息（一般是虚拟内存地址）的过程。

链接又分为静态链接和动态链接，前者是程序开发阶段程序员用 `ld`（`gcc` 实际上在后台调用了 `ld`）静态链接器手动链接的过程，而动态链接则是程序运行期间系统调用动态链接器（`ld-linux.so`）自动链接的过程。

比如，如果链接到可执行文件中的是静态链接库 `libmyprintf.a`，那么 `.rodata` 节区在链接后需要被重定位到一个绝对的虚拟内存地址，以便程序运行时能够正确访问该节区中的字符串信息。而对于 `puts` 函数，因为它是动态链接库 `libc.so` 中定义的函数，所以会在程序运行时通过动态符号链接找出 `puts` 函数在内存中的地址，以便程序调用该函数。在这里主要讨论静态链接过程，动态链接过程见[《动态符号链接的细节》][100]。

静态链接过程主要是把可重定位文件依次读入，分析各个文件的文件头，进而依次读入各个文件的节区，并计算各个节区的虚拟内存位置，对一些需要重定位的符号进行处理，设定它们的虚拟内存地址等，并最终产生一个可执行文件或者是动态链接库。这个链接过程是通过 `ld` 来完成的，`ld` 在链接时使用了一个链接脚本（`linker script`），该链接脚本处理链接的具体细节。

由于静态符号链接过程非常复杂，特别是计算符号地址的过程，考虑到时间关系，相关细节请参考 ELF 手册。这里主要介绍可重定位文件中的节区（节区表描述的）和可执行文件中段（程序头描述的）的对应关系以及 `gcc` 编译时采用的一些默认链接选项。

<span id="toc_27212_14734_21"></span>
### 可执行文件的段：节区重排

下面先来看看可执行文件的节区信息，通过程序头（段表）来查看，为了比较，先把 `test.o` 的节区表也列出：

```
$ readelf -S test.o
There are 12 section headers, starting at offset 0x118:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000010  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000520
       0000000000000018  0000000000000018          10     1     8
  [ 3] .data             PROGBITS         0000000000000000  00000050
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] .bss              NOBITS           0000000000000000  00000050
       0000000000000000  0000000000000000  WA       0     0     1
...
$ gcc -o test test.o myprintf.o
$ readelf -l test

Elf file type is EXEC (Executable file)
Entry point 0x400440
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040
                 0x00000000000001f8 0x00000000000001f8  R E    8
  INTERP         0x0000000000000238 0x0000000000400238 0x0000000000400238
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x0000000000000734 0x0000000000000734  R E    200000
  LOAD           0x0000000000000e10 0x0000000000600e10 0x0000000000600e10
                 0x0000000000000230 0x0000000000000238  RW     200000
  DYNAMIC        0x0000000000000e28 0x0000000000600e28 0x0000000000600e28
                 0x00000000000001d0 0x00000000000001d0  RW     8
  NOTE           0x0000000000000254 0x0000000000400254 0x0000000000400254
                 0x0000000000000044 0x0000000000000044  R      4
  GNU_EH_FRAME   0x00000000000005e4 0x00000000004005e4 0x00000000004005e4
                 0x000000000000003c 0x000000000000003c  R      4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     10
  GNU_RELRO      0x0000000000000e10 0x0000000000600e10 0x0000000000600e10
                 0x00000000000001f0 0x00000000000001f0  R      1

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame
   03     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss
   04     .dynamic
   05     .note.ABI-tag .note.gnu.build-id 
   06     .eh_frame_hdr
   07
   08     .init_array .fini_array .jcr .dynamic .got
```

可发现，`test` 和 `test.o`，`myprintf.o` 相比，多了很多节区，如 `.interp` 和 `.init` 等。另外，上表也给出了可执行文件的如下几个段（Segment）：

- `PHDR`: 给出了程序表自身的大小和位置，不能出现一次以上。
- `INTERP`: 因为程序中调用了 `puts`（在动态链接库中定义），使用了动态链接库，因此需要动态装载器／链接器（`ld-linux.so`）
- `LOAD`: 包括程序的指令，`.text` 等节区都映射在该段，只读（R）
- `LOAD`: 包括程序的数据，`.data`,`.bss` 等节区都映射在该段，可读写（RW）
- `DYNAMIC`: 动态链接相关的信息，比如包含有引用的动态链接库名字等信息
- `NOTE`: 给出一些附加信息的位置和大小
- `GNU_STACK`: 这里为空，应该是和GNU相关的一些信息

这里的段可能包括之前的一个或者多个节区，也就是说经过链接之后原来的节区被重排了，并映射到了不同的段，这些段将告诉系统应该如何把它加载到内存中。

<span id="toc_27212_14734_22"></span>
### 链接背后的故事

从上表中，通过比较可执行文件 `test` 中拥有的节区和可重定位文件（`test.o` 和 `myprintf.o`）中拥有的节区后发现，链接之后多了一些之前没有的节区，这些新的节区来自哪里？它们的作用是什么呢？先来通过 `gcc -v` 看看它的后台链接过程。

把可重定位文件链接成可执行文件：

```
$ gcc -v -o test test.o myprintf.o 2>&1 | tail -1
 /usr/lib/gcc/x86_64-linux-gnu/4.8/collect2 --sysroot=/ --build-id --eh-frame-hdr -m elf_x86_64 --hash-style=gnu --as-needed -dynamic-linker /lib64/ld-linux-x86-64.so.2 -z relro -o test /usr/lib/gcc/x86_64-linux-gnu/4.8/../../../x86_64-linux-gnu/crt1.o /usr/lib/gcc/x86_64-linux-gnu/4.8/../../../x86_64-linux-gnu/crti.o /usr/lib/gcc/x86_64-linux-gnu/4.8/crtbegin.o -L/usr/lib/gcc/x86_64-linux-gnu/4.8 -L/usr/lib/gcc/x86_64-linux-gnu/4.8/../../../x86_64-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/4.8/../../../../lib -L/lib/x86_64-linux-gnu -L/lib/../lib -L/usr/lib/x86_64-linux-gnu -L/usr/lib/../lib -L/usr/lib/gcc/x86_64-linux-gnu/4.8/../../.. test.o myprintf.o -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed /usr/lib/gcc/x86_64-linux-gnu/4.8/crtend.o /usr/lib/gcc/x86_64-linux-gnu/4.8/../../../x86_64-linux-gnu/crtn.o
```

从上述演示看出，`gcc` 在链接了我们自己的目标文件 `test.o` 和 `myprintf.o` 之外，还链接了 `crt1.o`，`crtbegin.o` 等额外的目标文件，难道那些新的节区就来自这些文件？

<span id="toc_27212_14734_23"></span>
### 用 ld 完成链接过程

另外 `gcc` 在进行了相关配置（`./configure`）后，调用了 `collect2`，却并没有调用 `ld`，通过查找 `gcc` 文档中和 `collect2` 相关的部分发现 `collect2` 在后台实际上还是去寻找 `ld` 命令的。为了理解 `gcc` 默认链接的后台细节，这里直接把 `collect2` 替换成 `ld`，并把一些路径换成绝对路径或者简化，得到如下的 `ld` 命令以及执行的效果。

```
$ ld --eh-frame-hdr \
-m elf_x86_64 \
-dynamic-linker /lib64/ld-linux-x86-64.so.2 \
-o test \
/usr/lib/gcc/x86_64-linux-gnu/4.8/../../../x86_64-linux-gnu/crt1.o \
/usr/lib/gcc/x86_64-linux-gnu/4.8/../../../x86_64-linux-gnu/crti.o \
/usr/lib/gcc/x86_64-linux-gnu/4.8/crtbegin.o \
test.o myprintf.o -lc \
/usr/lib/gcc/x86_64-linux-gnu/4.8/crtend.o /usr/lib/gcc/x86_64-linux-gnu/4.8/../../../x86_64-linux-gnu/crtn.o
$ ./test
hello, world!
```

不出所料，它完美地运行了。下面通过 `ld` 的手册（`man ld`）来分析一下这几个参数：

- `--eh-frame-hdr`

    要求创建一个 `.eh_frame_hdr` 节区(貌似目标文件test中并没有这个节区，所以不关心它)。

- `-m elf_x86_64`

    这里指定不同平台上的链接脚本，可以通过 `--verbose` 命令查看脚本的具体内容，如 `ld -m elf_x86_64 --verbose`，它实际上被存放在一个文件中（`/usr/lib/ldscripts` 目录下），我们可以去修改这个脚本，具体如何做？请参考 `ld` 的手册。在后面我们将简要提到链接脚本中是如何预定义变量的，以及这些预定义变量如何在我们的程序中使用。需要提到的是，如果不是交叉编译，那么无须指定该选项。

- -dynamic-linker /lib64/ld-linux-x86-64.so.2

    指定动态装载器/链接器，即程序中的 `INTERP` 段中的内容。动态装载器/链接器负责链接有可共享库的可执行文件的装载和动态符号链接。

- -o test

    指定输出文件，即可执行文件名的名字

- /usr/lib/gcc/x86_64-linux-gnu/4.8/../../../x86_64-linux-gnu/crt1.o ; /usr/lib/gcc/x86_64-linux-gnu/4.8/../../../x86_64-linux-gnu/crti.o ; /usr/lib/gcc/x86_64-linux-gnu/4.8/crtbegin.o

    链接到 `test` 文件开头的一些内容，这里实际上就包含了 `.init` 等节区。`.init` 节区包含一些可执行代码，在 `main` 函数之前被调用，以便进行一些初始化操作，在 C++ 中完成构造函数功能。

- test.o myprintf.o

    链接我们自己的可重定位文件

- `-lc`

    链接 `libc` 库，后者定义有我们需要的 `puts` 函数

- /usr/lib/gcc/x86_64-linux-gnu/4.8/crtend.o ; /usr/lib/gcc/x86_64-linux-gnu/4.8/../../../x86_64-linux-gnu/crtn.o

    链接到 `test` 文件末尾的一些内容，这里实际上包含了 `.fini` 等节区。`.fini` 节区包含了一些可执行代码，在程序退出时被执行，作一些清理工作，在 C++ 中完成析构造函数功能。我们往往可以通过 `atexit` 来注册那些需要在程序退出时才执行的函数。

<span id="toc_27212_14734_24"></span>
### C++构造与析构：crtbegin.o和crtend.o

对于 `crtbegin.o` 和 `crtend.o` 这两个文件，貌似完全是用来支持 C++ 的构造和析构工作的，所以可以不链接到我们的可执行文件中，链接时把它们去掉看看，

```
$ ld -m elf_x86_64 -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o test \
  /usr/lib/x86_64-linux-gnu/crt1.o /usr/lib/x86_64-linux-gnu/crti.o \
  test.o myprintf.o \
  -lc /usr/lib/x86_64-linux-gnu/crtn.o    #后面发现不用链接libgcc，也不用--eh-frame-hdr参数
$ readelf -l test

Elf file type is EXEC (Executable file)
Entry point 0x4003c0
There are 7 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040
                 0x0000000000000188 0x0000000000000188  R E    8
  INTERP         0x00000000000001c8 0x00000000004001c8 0x00000000004001c8
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
...
 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.ABI-tag .hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame
   03     .dynamic .got .got.plt .data
   04     .dynamic
   05     .note.ABI-tag
   06
$ ./test
hello, world!
```

完全可以工作，而且发现 `.ctors`（保存着程序中全局构造函数的指针数组）, `.dtors`（保存着程序中全局析构函数的指针数组）,`.jcr`（未知）,`.eh_frame` 节区都没有了，所以 `crtbegin.o` 和 `crtend.o` 应该包含了这些节区。

<span id="toc_27212_14734_25"></span>
### 初始化与退出清理：crti.o 和 crtn.o

而对于另外两个文件 `crti.o` 和 `crtn.o`，通过 `readelf -S` 查看后发现它们都有 `.init` 和 `.fini` 节区，如果我们不需要让程序进行一些初始化和清理工作呢？是不是就可以不链接这个两个文件？试试看。

```
$ ld  -m elf_x86_64 -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o test \
      /usr/lib/x86_64-linux-gnu/crt1.o test.o myprintf.o -L/usr/lib/ -lc
/usr/lib/libc_nonshared.a(elf-init.oS): In function `__libc_csu_init':
(.text+0x25): undefined reference to `_init'
```

貌似不行，竟然有人调用了 `__libc_csu_init` 函数，而这个函数引用了 `_init`。这两个符号都在哪里呢？

```
$ readelf -s /usr/lib/crt1.o | grep __libc_csu_init
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND __libc_csu_init
$ readelf -s /usr/lib/crti.o | grep _init
     9: 0000000000000000     0 FUNC    GLOBAL DEFAULT    4 _init
```

竟然是 `crt1.o` 调用了 `__libc_csu_init` 函数，而该函数却引用了我们没有链接的 `crti.o` 文件中定义的 `_init` 符号。这样的话不链接 `crti.o` 和 `crtn.o` 文件就不成了罗？不对吧，要不干脆不用 `crt1.o` 算了，看看 `gcc` 额外链接进去的最后一个文件 `crt1.o` 到底干了个啥子？

```
$ ld -m elf_x86_64 -dynamic-linker /lib64/ld-linux-x86-64.so.2 \
     -o test test.o myprintf.o -lc
ld: warning: cannot find entry symbol _start; defaulting to 0000000000400270
```

这样却说没有找到入口符号 `_start`，难道 `crt1.o` 中定义了这个符号？不过它给默认设置了一个地址，只是个警告，说明 `test` 已经生成，不管怎样先运行看看再说。

```
$ ./test
hello, world!
Segmentation fault (core dumped)
```

貌似程序运行完了，不过结束时冒出个段错误？可能是程序结束时有问题，用 `gdb` 调试看看：

```
$ gcc -g -c test.c myprintf.c #产生目标代码, 非交叉编译，不指定-m也可链接，所以下面可去掉-m
$ ld -dynamic-linker /lib/ld-linux.so.2 -o test \
     test.o myprintf.o -L/usr/lib -lc
ld: warning: cannot find entry symbol _start; defaulting to 0000000000400270
$ ./test
hello, world!
Segmentation fault
$ gdb -q ./test
(gdb) list
1       #include "test.h"
2       int main()
3       {
4               myprintf();
5               return 0;
6       }
(gdb) break 6      #在程序的末尾设置一个断点
Breakpoint 1 at 0x40027e: file test.c, line 6.
(gdb) r            #程序都快结束了都没问题，怎么会到最后出个问题呢？
Starting program: /tmp/test
hello, world!

Breakpoint 1, main () at test.c:6
7       }
(gdb) n        #单步执行看看，怎么下面一条指令是0x0000000000000001，肯定是程序退出以后出了问题
0x00000001 in ?? ()
(gdb) n        #诶，当然找不到边了，都跑到0x0000000000000001了
Cannot find bounds of current function
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x0000000000000001 in ?? ()
```

原来是这么回事，估计是 `return 0` 返回之后出问题了，看看它的汇编去。

```
$ gcc -S test.c #产生汇编代码
$ cat test.s
...
	call	myprintf
	movl	$0, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
...
```

后面就这么几条指令，难不成 `ret` 返回有问题，不让它 `ret` 返回，把 `return` 改成 `_exit` 直接进入内核退出。

```
$ cp test.c test-exit.c vim test-exit.c
$ cat test-exit.c    #就把return语句修改成_exit了。
#include "test.h"
#include <unistd.h> /* _exit */
int main()
{
	myprintf();
	_exit(0);
}
$ gcc -g -c test-exit.c myprintf.c
$ ld -dynamic-linker /lib/ld-linux.so.2 -o test-exit test-exit.o myprintf.o -L/usr/lib -lc
ld: warning: cannot find entry symbol _start; defaulting to 00000000004002c0
$ ./test-exit    #竟然好了，再看看汇编有什么不同
hello, world!
$ gcc -S test-exit.c
$ cat test-exit.s    #貌似就把ret指令替换成了_exit函数调用，直接进入内核，让内核处理了，那为什么ret有问题呢？
...
        call    myprintf
        movl    $0, %edi
        call    _exit
...
$ gdb -q ./test    #把代码改回去（改成return 0;），再调试看看调用main函数返回时的下一条指令地址eip
(gdb) list
warning: Source file is more recent than executable.
1       #include "test.h"
2       int main()
3       {
4               myprintf();
5               return 0;
6       }
(gdb) break 4
Breakpoint 1 at 0x400274: file test.c, line 4.
(gdb) break 6
Breakpoint 2 at 0x400279: file test.c, line 6.
(gdb) r
Starting program: /mnt/hda8/Temp/c/program/test

Breakpoint 1, main () at test.c:5
5               myprintf();
(gdb) x/4x $rsp
0x7fffffffe588:	0x00000000	0x00000000	0x00000001	0x00000000
```

发现 `0x00000001` 刚好是之前调试时看到的程序返回后的位置，即 `rip`，说明程序在初始化时，这个 `eip` 就是错误的。为什么呢？因为根本没有链接进初始化的代码，而是在编译器自己给我们，初始化了程序入口即 `0000000000400270`，也就是说，没有人调用 `main`，`main` 不知道返回哪里去，所以，我们直接让 `main` 结束时进入内核调用 `_exit` 而退出则不会有问题。

通过上面的演示和解释发现只要把return语句修改为_exit语句，程序即使不链接任何额外的目标代码都可以正常运行（原因是不链接那些额外的文件时相当于没有进行初始化操作，如果在程序的最后执行ret汇编指令，程序将无法获得正确的eip，从而无法进行后续的动作）。但是为什么会有“找不到_start符号”的警告呢？通过`readelf -s`查看crt1.o发现里头有这个符号，并且crt1.o引用了main这个符号，是不是意味着会从`_start`进入main呢？是不是程序入口是`_start`，而并非main呢？

<span id="toc_27212_14734_26"></span>
### C 语言程序真正的入口

先来看看刚才提到的链接器的默认链接脚本（`ld -m elf_x86_64 --verbose`），它告诉我们程序的入口（entry）是 `_start`，而一个可执行文件必须有一个入口地址才能运行，所以这就是说明了为什么 `ld` 一定要提示我们 “_start找不到”，找不到以后就给默认设置了一个地址。

```
$ ld --verbose  | grep ^ENTRY    #非交叉编译，可不用-m参数；ld默认找_start入口，并不是main哦！
ENTRY(_start)
```

原来是这样，程序的入口（entry）竟然不是 `main` 函数，而是 `_start`。那干脆把汇编里头的 `main` 给改掉算了，看行不行？

先生成汇编 `test.s`：

```
$ cat test.c
#include "test.h"
#include <unistd.h>     /* _exit */

int main()
{
	myprintf();
	_exit(0);
}
$ gcc -S test.c
```

然后把汇编中的 `main` 改为 `_start`，即改程序入口为 `_start`：

```
$ sed -i -e "s#main#_start#g" test.s
$ gcc -c test.s myprintf.c
```

重新链接，发现果然没问题了：

```
$ ld -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o test test.o myprintf.o -L/usr/lib/ -lc
$ ./test
hello, world!
```

`_start` 竟然是真正的程序入口，那在有 `main` 的情况下呢？为什么在 `_start` 之后能够找到 `main` 呢？这个看看 alert7 大叔的[Before main 分析](http://www.xfocus.net/articles/200109/269.html)吧，这里不再深入介绍。

总之呢，通过修改程序的 `return` 语句为 `_exit(0)` 和修改程序的入口为 `_start`，我们的代码不链接 `gcc` 默认链接的那些额外的文件同样可以工作得很好。并且打破了一个学习 C 语言以来的常识：`main` 函数作为程序的主函数，是程序的入口，实际上则不然。

<span id="toc_27212_14734_27"></span>
### 链接脚本初次接触

再补充一点内容，在 `ld` 的链接脚本中，有一个特别的关键字 `PROVIDE`，由这个关键字定义的符号是 `ld` 的预定义字符，我们可以在 C 语言函数中扩展它们后直接使用。这些特别的符号可以通过下面的方法获取，

```
$ ld --verbose | grep PROVIDE | grep -v HIDDEN
  PROVIDE (__executable_start = SEGMENT_START("text-segment", 0x400000)); . = SEGMENT_START("text-segment", 0x400000) + SIZEOF_HEADERS;
  PROVIDE (__etext = .);
  PROVIDE (_etext = .);
  PROVIDE (etext = .);
  _edata = .; PROVIDE (edata = .);
  _end = .; PROVIDE (end = .);
```

这里面有几个我们比较关心的，第一个是程序的入口地址 `__executable_start`，另外三个是 `etext`，`edata`，`end`，分别对应程序的代码段（text）、初始化数据（data）和未初始化的数据（bss）（可参考`man etext`），如何引用这些变量呢？看看这个例子。

```
/* predefinevalue.c */

#include <stdio.h>
extern int __executable_start, etext, edata, end;

int main(void)
{
	printf ("program entry: 0x%x \n", &__executable_start);
	printf ("etext address(text segment): 0x%x \n", &etext);
	printf ("edata address(initilized data): 0x%x \n", &edata);
	printf ("end address(uninitilized data): 0x%x \n", &end);

	return 0;
}
```

到这里，程序链接过程的一些细节都介绍得差不多了。在[《动态符号链接的细节》][100]中将主要介绍 ELF 文件的动态符号链接过程。

<span id="toc_27212_14734_28"></span>
## 参考资料

- [Linux 汇编语言开发指南](http://www.ibm.com/developerworks/cn/linux/l-assembly/index.html)
- [PowerPC 汇编](http://www.ibm.com/developerworks/cn/linux/hardware/ppc/assembly/index.html)
- [用于 Power 体系结构的汇编语言](http://www.ibm.com/developerworks/cn/linux/l-powasm1.html)
- [Linux 中 x86 的内联汇编](http://www.ibm.com/developerworks/cn/linux/sdk/assemble/inline/index.html)
- Linux Assembly HOWTO
- Linux Assembly Language Programming
- Guide to Assembly Language Programming in Linux
- [An beginners guide to compiling programs under Linux](http://www.luv.asn.au/overheads/compile.html)
- [gcc manual](http://gcc.gnu.org/onlinedocs/gcc-4.2.2/gcc/)
- [A Quick Tour of Compiling, Linking, Loading, and Handling Libraries on Unix](http://efrw01.frascati.enea.it/Software/Unix/IstrFTU/cern-cnl-2001-003-25-link.html)
- [Unix 目标文件初探](http://www.ibm.com/developerworks/cn/aix/library/au-unixtools.html)
- [Before main()分析](http://www.xfocus.net/articles/200109/269.html)
- [A Process Viewing Its Own /proc/<PID>/map Information](http://www.linuxforums.org/forum/linux-kernel/51790-process-viewing-its-own-proc-pid-map-information.html)
- UNIX 环境高级编程
- Linux Kernel Primer
- [Understanding ELF using readelf and objdump](http://www.linuxforums.org/misc/understanding_elf_using_readelf_and_objdump.html)
- [Study of ELF loading and relocs](http://netwinder.osuosl.org/users/p/patb/public_html/elf_relocs.html)
- ELF file format and ABI
    - [\[1\]](http://www.x86.org/ftp/manuals/tools/elf.pdf)
    - [\[2\]](http://www.muppetlabs.com/~breadbox/software/ELF.txt)
- TN05.ELF.Format.Summary.pdf
- [ELF文件格式(中文)](http://www.xfocus.net/articles/200105/174.html)
- 关于 Gcc 方面的论文，请查看历年的会议论文集
    - [2005](http://www.gccsummit.org/2005/2005-GCC-Summit-Proceedings.pdf)
    - [2006](http://www.gccsummit.org/2006/2006-GCC-Summit-Proceedings.pdf)
- [The Linux GCC HOW TO](http://www.faqs.org/docs/Linux-HOWTO/GCC-HOWTO.html)
- [ELF: From The Programmer's Perspective](http://linux.jinr.ru/usoft/WWW/www_debian.org/Documentation/elf/elf.html)
- [C/C++ 程序编译步骤详解](http://www.xxlinux.com/linux/article/development/soft/20070424/8267.html)
- [C 语言常见问题集](http://c-faq-chn.sourceforge.net/ccfaq/index.html)
- [使用 BFD 操作 ELF](http://elfhack.whitecell.org/mydocs/use_bfd.txt)
- [bfd document](http://sourceware.org/binutils/docs/bfd/index.html)
- [UNIX/LINUX 平台可执行文件格式分析](http://blog.chinaunix.net/u/19881/showart_215242.html)
- [Linux 汇编语言快速上手：4大架构一块学](http://www.tinylab.org/linux-assembly-language-quick-start/)
- GNU binutils 小结
