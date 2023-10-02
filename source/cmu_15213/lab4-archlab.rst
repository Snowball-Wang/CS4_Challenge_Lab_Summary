Lab 4: Architecture Lab
=======================

0x01. 实验介绍
--------------

在本实验中，我们将学习一个流水线化的Y86-64处理器的设计与实现，并设法优化此处理器和基准测试程序以最大化性能。
完成此实验后，我们将对软硬件之间的相互作用对程序性能的影响，有更加深刻的认识。

本实验分为三个部分。在Part A中，我们将通过写一些Y86-64的汇编程序，来熟悉相关的Y86-64工具。
在Part B中，我们将对Y86-64处理器的顺序实现（SEQ simulator）添加一条新指令。
基于以上两个实验，我们来到本实验的核心Part C，此部分我们将对Y86-64的基准程序和处理器设计进行优化。

总之，通过此实验，我们将了解CPU设计的基本流程和规范，同时也对如何能够写出高性能的优化代码有更深的理解。

0x02. 实验环境搭建
------------------

实验的源代码可通过以下命令下载解压：

.. code-block:: console

    $ wget http://csapp.cs.cmu.edu/3e/archlab-handout.tar
    $ tar xvf archlab-handout.tar

进入实验目录，再通过以下命令对 ``sim.tar`` 压缩包进行解压：

.. code-block:: console

    $ cd archlab-handout/
    $ tar xvf sim.tar

首先查看一下本实验包含的文件。
在 ``archlab-handout`` 目录下，有以下文件和目录：

* ``archlab.pdf`` 是本实验的实验要求。
* ``Makefile`` 用于实验代码提交，我们属于自学，当然就用不着了。
* ``README`` 是对实验顶级目录下各个文件或文件夹的说明。
* ``sim`` 是Y86-64的工具合集，也是我们实际编写代码，运行程序的地方。
* ``simguide.pdf`` 说明Y86-64中相关工具的使用。

进入到解压缩后的 ``sim`` 目录里，有以下文件和目录：

* ``Makefile`` 用来编译Y86-64工具链。
* ``README`` 是对此目录下各个文件或文件夹的说明。
* ``misc`` 包含Y86-64的汇编器yas，指令模拟器yis，ISA源文件以及hcl2c和hcl2v的源代码。
* ``seq`` 包含SEQ和SEQ+处理器的源代码。
* ``pipe`` 包含PIPE处理器的源代码。
* ``y86-code`` 包含书本中的 ``.ys`` 例子和相应处理器基准测试脚本。
* ``ptest`` 包含测试处理器设计的自动回归脚本。

理清了实验中包含的文件以及每个文件的用途，我们先编译实验，熟悉实验提供的工具：

.. code-block:: console

    $ cd sim/
    $ make
    (cd misc; make all)
    make[1]: Entering directory '/home/jiewan01/CS4_Challenge/csapp_lab/test/archlab-handout/sim/misc'
    gcc -Wall -O1 -g -c yis.c
    gcc -Wall -O1 -g -c isa.c
    gcc -Wall -O1 -g yis.o isa.o -o yis
    gcc -Wall -O1 -g -c yas.c
    gcc -O1 -c yas-grammar.c
    gcc -Wall -O1 -g yas-grammar.o yas.o isa.o -lfl -o yas
    gcc -O1 node.c lex.yy.c hcl.tab.c outgen.c -o hcl2c
    make[1]: Leaving directory '/home/jiewan01/CS4_Challenge/csapp_lab/test/archlab-handout/sim/misc'
    (cd pipe; make all GUIMODE= TKLIBS="-L/usr/lib -ltk -ltcl" TKINC="-isystem /usr/include/tcl8.5")
    make[1]: Entering directory '/home/jiewan01/CS4_Challenge/csapp_lab/test/archlab-handout/sim/pipe'
    # Building the pipe-std.hcl version of PIPE
    ../misc/hcl2c -n pipe-std.hcl < pipe-std.hcl > pipe-std.c
    gcc -Wall -O2 -isystem /usr/include/tcl8.5 -I../misc  -o psim psim.c pipe-std.c \
            ../misc/isa.c -L/usr/lib -ltk -ltcl -lm
    /usr/bin/ld: cannot find -ltk
    /usr/bin/ld: cannot find -ltcl
    collect2: error: ld returned 1 exit status
    Makefile:42: recipe for target 'psim' failed
    make[1]: *** [psim] Error 1
    make[1]: Leaving directory '/home/jiewan01/CS4_Challenge/csapp_lab/test/archlab-handout/sim/pipe'
    Makefile:26: recipe for target 'all' failed
    make: *** [all] Error 2

编译错误，原因是 ``ld`` 链接器找不到对应的 ``tk`` 和 ``tcl`` 库。
我使用的是Ubuntu 20.04系统，可通过 ``sudo apt-get install tk-dev tcl-dev`` 命令下载安装上述库。
然后修改 ``Makefile`` 文件，指定链接 ``tk`` 和 ``tck`` 头文件的版本为8.6。

.. code-block:: makefile

    TKINC=-system /usr/include/tcl8.6

再次敲入 ``make`` 命令编译实验，此时编译通过。
需要注意的是，Y86-64处理器模拟器有两种模式，分别是 ``TTY mode`` 和 ``GUI mode`` 。
``TTY mode`` 则是把所有的结果输出到终端上，而 ``GUI mode`` 则是有图形界面对运行结果进行可视化。
``GUI mode`` 需要安装Tcl/Tk库，同时要在 ``Makefile`` 打开开关 ``GUIMODE=-DHAS_GUI`` 。

因为本实验设计的时间有一定年限了，在编译 ``GUI mode`` 时，会遇到诸如 ``‘Tcl_Interp’ has no member named ‘result’`` 和 ``undefined reference to `matherr'`` 等错误，这是由于Tcl库版本兼容的问题导致的，可通过以下patch解决。

.. code-block:: console

    Author: Jieqiang Wang <wangjieqiang123@163.com>
    Date:   Tue Jul 11 10:15:20 2023 +0800

        archlab: fix build issues for GUI mode

    diff --git a/sim/Makefile b/sim/Makefile
    index 7fd8f06..887fe84 100644
    --- a/sim/Makefile
    +++ b/sim/Makefile
    @@ -1,19 +1,19 @@
    # Comment this out if you don't have Tcl/Tk on your system

    -#GUIMODE=-DHAS_GUI
    +GUIMODE=-DHAS_GUI

    # Modify the following line so that gcc can find the libtcl.so and
    # libtk.so libraries on your system. You may need to use the -L option
    # to tell gcc which directory to look in. Comment this out if you
    # don't have Tcl/Tk.

    -TKLIBS=-L/usr/lib -ltk -ltcl
    +TKLIBS=-L/usr/lib -ltk8.6 -ltcl8.6

    # Modify the following line so that gcc can find the tcl.h and tk.h
    # header files on your system. Comment this out if you don't have
    # Tcl/Tk.

    -TKINC=-isystem /usr/include/tcl8.5
    +TKINC=-isystem /usr/include/tcl8.6

    ##################################################
    # You shouldn't need to modify anything below here
    diff --git a/sim/pipe/Makefile b/sim/pipe/Makefile
    index ca4607e..81839fc 100644
    --- a/sim/pipe/Makefile
    +++ b/sim/pipe/Makefile
    @@ -17,7 +17,7 @@ TKLIBS=-L/usr/lib -ltk -ltcl
    # header files on your system. Comment this out if you don't have
    # Tcl/Tk.

    -TKINC=-isystem /usr/include/tcl8.5
    +TKINC=-isystem /usr/include/tcl8.6

    # Modify these two lines to choose your compiler and compile time
    # flags.
    @@ -25,6 +25,9 @@ TKINC=-isystem /usr/include/tcl8.5
    CC=gcc
    CFLAGS=-Wall -O2

    +# Add following flags to suppress building error due to tcl tools
    +CPPFLAGS=-DUSE_INTERP_RESULT
    +
    ##################################################
    # You shouldn't need to modify anything below here
    ##################################################
    @@ -41,7 +44,7 @@ all: psim drivers
    psim: psim.c sim.h pipe-$(VERSION).hcl $(MISCDIR)/isa.c $(MISCDIR)/isa.h
            # Building the pipe-$(VERSION).hcl version of PIPE
            $(HCL2C) -n pipe-$(VERSION).hcl < pipe-$(VERSION).hcl > pipe-$(VERSION).c
    -       $(CC) $(CFLAGS) $(INC) -o psim psim.c pipe-$(VERSION).c \
    +       $(CC) $(CPPFLAGS) $(CFLAGS) $(INC) -o psim psim.c pipe-$(VERSION).c \
                    $(MISCDIR)/isa.c $(LIBS)

    # This rule builds driver programs for Part C of the Architecture Lab
    diff --git a/sim/pipe/psim.c b/sim/pipe/psim.c
    index c08508e..28b9642 100644
    --- a/sim/pipe/psim.c
    +++ b/sim/pipe/psim.c
    @@ -803,9 +803,10 @@ void sim_log( const char *format, ... ) {
    **********************/

    /* Hack for SunOS */
    +/*
    extern int matherr();
    int *tclDummyMathPtr = (int *) matherr;
    -
    +*/
    static char tcl_msg[256];

    /* Keep track of the TCL Interpreter */
    diff --git a/sim/seq/Makefile b/sim/seq/Makefile
    index 0c71aae..9cbd4b9 100644
    --- a/sim/seq/Makefile
    +++ b/sim/seq/Makefile
    @@ -17,7 +17,7 @@ TKLIBS=-L/usr/lib -ltk -ltcl
    # header files on your system. Comment this out if you don't have
    # Tcl/Tk.

    -TKINC=-isystem /usr/include/tcl8.5
    +TKINC=-isystem /usr/include/tcl8.6

    # Modify these two lines to choose your compiler and compile time
    # flags.
    @@ -25,6 +25,7 @@ TKINC=-isystem /usr/include/tcl8.5
    CC=gcc
    CFLAGS=-Wall -O2

    +CPPFLAGS=-DUSE_INTERP_RESULT
    ##################################################
    # You shouldn't need to modify anything below here
    ##################################################
    @@ -41,14 +42,14 @@ all: ssim
    ssim: seq-$(VERSION).hcl ssim.c  sim.h $(MISCDIR)/isa.c $(MISCDIR)/isa.h
            # Building the seq-$(VERSION).hcl version of SEQ
            $(HCL2C) -n seq-$(VERSION).hcl <seq-$(VERSION).hcl >seq-$(VERSION).c
    -       $(CC) $(CFLAGS) $(INC) -o ssim \
    +       $(CC) $(CPPFLAGS) $(CFLAGS) $(INC) -o ssim \
                    seq-$(VERSION).c ssim.c $(MISCDIR)/isa.c $(LIBS)

    # This rule builds the SEQ+ simulator (ssim+)
    ssim+: seq+-std.hcl ssim.c sim.h $(MISCDIR)/isa.c $(MISCDIR)/isa.h
            # Building the seq+-std.hcl version of SEQ+
            $(HCL2C) -n seq+-std.hcl <seq+-std.hcl >seq+-std.c
    -       $(CC) $(CFLAGS) $(INC) -o ssim+ \
    +       $(CC) $(CPPFLAGS) $(CFLAGS) $(INC) -o ssim+ \
                    seq+-std.c ssim.c $(MISCDIR)/isa.c $(LIBS)

    # These are implicit rules for assembling .yo files from .ys files.
    diff --git a/sim/seq/ssim.c b/sim/seq/ssim.c
    index 4cae5a9..eecc07d 100644
    --- a/sim/seq/ssim.c
    +++ b/sim/seq/ssim.c
    @@ -841,9 +841,10 @@ void sim_log( const char *format, ... ) {
    **********************/

    /* Hack for SunOS */
    +/*
    extern int matherr();
    int *tclDummyMathPtr = (int *) matherr;
    -
    +*/
    static char tcl_msg[256];

    /* Keep track of the TCL Interpreter */

这个patch的核心修改就是通过添加 ``CPPFLAGS=-DUSE_INTERP_RESULT`` 来绕过因为Tcl库8.6版本问题导致的结构体 ``Tcl_Interp`` 无成员变量 ``result`` 的问题。
同时注释掉 ``matherr`` 函数的使用，这种用法已经过时了。
再次编译实验，这次对应 ``GUI mode`` 的实验代码即可编译成功。

我们可以运行 ``y86-code`` 里的一个例子，来验证实验编译成功。
``asum.yo`` 是由 ``asum.ys`` 源代码生成，取自于书中数组求和的例子。
可以看到，通过 ``yis`` Y86-64指令模拟器运行程序，各个寄存器和内存的变化，最终函数返回的结果存在寄存器 ``%rax`` 中。
其值为 ``0xabcdabcdabcd`` ，同预期一致。

.. code-block:: console

    $ cd y86-code
    $ ../misc/yis asum.yo
    Stopped in 34 steps at PC = 0x13.  Status 'HLT', CC Z=1 S=0 O=0
    Changes to registers:
    %rax:   0x0000000000000000      0x0000abcdabcdabcd
    %rsp:   0x0000000000000000      0x0000000000000200
    %rdi:   0x0000000000000000      0x0000000000000038
    %r8:    0x0000000000000000      0x0000000000000008
    %r9:    0x0000000000000000      0x0000000000000001
    %r10:   0x0000000000000000      0x0000a000a000a000

    Changes to memory:
    0x01f0: 0x0000000000000000      0x0000000000000055
    0x01f8: 0x0000000000000000      0x0000000000000013

至此，我们成功地编译了实验，并熟悉了该实验相关的工具链。接下来开始我们的实验部分内容。

0x03. 实验过程及思路说明
-----------------------

Part A
^^^^^^^^

本部分的实验都是在 ``sim/misc`` 目录下进行。
实验的主要任务是将 ``examples.c`` 里三个C函数通过Y86-64汇编代码实现。
然后用实验提供的 ``yas`` 汇编器生成Y86-64二进制代码，再使用 ``yis`` 指令模拟器运行生成的Y86-64程序。

sum.ys
''''''''

本实验是将 ``examples.c`` 中的 ``sum_list`` 函数通过Y86-64汇编语言实现。
首先看C代码的实现：

.. code-block:: c

    /* linked list element */
    typedef struct ELE {
        long val;
        struct ELE *next;
    } *list_ptr;

    /* sum_list - Sum the elements of a linked list */
    long sum_list(list_ptr ls)
    {
        long val = 0;
        while (ls) {
            val += ls->val;
            ls = ls->next;
        }
        return val;
    }

在将上述C代码转换成Y86-64汇编语言之前，我们可先将其转换成X86-64汇编语言。
`此网站 <https://godbolt.org/>`_ 可在线将C代码转换成X86-64汇编代码，如下所示：

.. code-block:: asm

    sum_list(ELE*):
        movl    $0, %eax
        jmp     .L2
    .L3:
        addq    (%rdi), %rax
        movq    8(%rdi), %rdi
    .L2:
        testq   %rdi, %rdi
        jne     .L3
        ret

基于此X86-64汇编代码，再结合 ``misc/asum.ys`` 例子，我们可完成 ``sum.ys`` 的实现。

.. code-block:: asm

    # Execution begins at address 0
        .pos 0
        irmovq stack, %rsp      # Set up stack pointer
        call main               # Execute main program
        halt                    # Terminate program

    # Sample linked list
    .align 8
    ele1:
            .quad 0x00a
            .quad ele2
    ele2:
            .quad 0x0b0
            .quad ele3
    ele3:
            .quad 0xc00
            .quad 0

    main:
            irmovq ele1,%rdi
            call sum_list        # sum_list(list_ptr ls)
            ret

    # long sum_list(list_ptr ls)
    # ls in %rdi
    sum_list:
            xorq %rax,%rax       # sum = 0
            jmp     test         # Goto test
    loop:
            mrmovq (%rdi),%r10   # Get ls->val
            addq %r10,%rax       # Add to sum
            mrmovq 8(%rdi),%rdi  # ls = ls->next
    test:
            andq %rdi,%rdi       # Is ls NULL?
            jne    loop          # Stop when 0
            ret                  # Return

    # Stack starts here and grows to lower addresses
            .pos 0x200
    stack:

再通过 ``yas`` 生成二进制程序， ``yis`` 运行程序。
程序最终返回 ``0xcba`` ，其结果保存在 ``%rax`` 寄存器中。
同原C函数的语义保持一致， ``sum.ys`` 完成。

.. code-block:: console

    $ ../misc/yas sum.ys # output sum.yo
    $ ../misc/yis sum.yo
    Stopped in 26 steps at PC = 0x13.  Status 'HLT', CC Z=1 S=0 O=0
    Changes to registers:
    %rax:   0x0000000000000000      0x0000000000000cba
    %rsp:   0x0000000000000000      0x0000000000000200
    %r10:   0x0000000000000000      0x0000000000000c00

    Changes to memory:
    0x01f0: 0x0000000000000000      0x000000000000005b
    0x01f8: 0x0000000000000000      0x0000000000000013


rsum.ys
''''''''

本实验的任务是将上述 ``sum.ys`` 遍历链表的方式，改用递归的方法实现。
先看C函数 ``rsum_list`` 的实现：

.. code-block:: c

    /* rsum_list - Recursive version of sum_list */
    long rsum_list(list_ptr ls)
    {
        if (!ls)
            return 0;
        else {
            long val = ls->val;
            long rest = rsum_list(ls->next);
            return val + rest;
        }
    }

同样的套路，我们先把上述C代码转换成X86-64的汇编代码：

.. code-block:: asm

    rsum_list(ELE*):
            testq   %rdi, %rdi
            je      .L3
            pushq   %rbx
            movq    (%rdi), %rbx
            movq    8(%rdi), %rdi
            call    rsum_list(ELE*)
            addq    %rbx, %rax
            popq    %rbx
            ret
    .L3:
            movl    $0, %eax
            ret

``rsum.ys`` 的实现的框架基本同 ``sum.ys`` 相同，只需将其中的 ``sum_list`` 替换成 ``rsum_list`` ，并作些许修改即可。

.. code-block:: asm

    # Execution begins at address 0
            .pos 0
            irmovq stack, %rsp      # Set up stack pointer
            call main               # Execute main program
            halt                    # Terminate program

    # Sample linked list
    .align 8
    ele1:
            .quad 0x00a
            .quad ele2
    ele2:
            .quad 0x0b0
            .quad ele3
    ele3:
            .quad 0xc00
            .quad 0

    main:
            irmovq ele1,%rdi
            call rsum_list       # rsum_list(list_ptr ls)
            ret

    # long rsum_list(list_ptr ls)
    # ls in %rdi
    rsum_list:
            andq %rdi,%rdi       # Is ls NULL?
            je return            # If ls is NULL, return 0
            pushq %rbx           # Save callee-saved register
            mrmovq (%rdi),%rbx   # Get ls->val
            mrmovq 8(%rdi),%rdi  # ls->next
            call rsum_list       # Call rsum_list recursively
            addq %rbx,%rax       # Add to sum
            popq %rbx            # Pop callee-saved register
            ret
    return:
            irmovq $0,%rax       # Set return val to 0
            ret                  # Return

    # Stack starts here and grows to lower addresses
            .pos 0x200
    stack:

``rsum.ys`` 的编译和运行结果如下所示：

.. code-block:: console

    $ ../misc/yas rsum.ys # output rsum.yo
    $ ../misc/yis rsum.yo
    Stopped in 37 steps at PC = 0x13.  Status 'HLT', CC Z=0 S=0 O=0
    Changes to registers:
    %rax:   0x0000000000000000      0x0000000000000cba
    %rsp:   0x0000000000000000      0x0000000000000200

    Changes to memory:
    0x01c0: 0x0000000000000000      0x0000000000000086
    0x01c8: 0x0000000000000000      0x00000000000000b0
    0x01d0: 0x0000000000000000      0x0000000000000086
    0x01d8: 0x0000000000000000      0x000000000000000a
    0x01e0: 0x0000000000000000      0x0000000000000086
    0x01f0: 0x0000000000000000      0x000000000000005b
    0x01f8: 0x0000000000000000      0x0000000000000013

可以看到，递归的实现方式最终的结果，也就是 ``%rax`` 值同遍历链表的方法得到的结果是一致的。
但是运行结果显示，递归需要与内存发生更多的交互，因为存在着递归函数出栈压栈的操作。

copy.ys
''''''''

本实验的任务是编写 ``copy.ys`` Y86-64汇编代码，实现将一个位置的内存块的内容，拷贝到另一个内存块位置，并计算出被拷贝的内存内容的校正码。
查看C函数 ``copy_block`` 的实现：

.. code-block:: c

    /* copy_block - Copy src to dest and return xor checksum of src */
    long copy_block(long *src, long *dest, long len)
    {
        long result = 0;
        while (len > 0) {
            long val = *src++;
            *dest++ = val;
            result ^= val;
            len--;
        }
        return result;
    }

先将上述C代码转换成X86-64汇编代码：

.. code-block:: asm

    copy_block(long*, long*, long):
            movl    $0, %ecx
            jmp     .L2
    .L3:
            movq    (%rdi), %rax
            movq    %rax, (%rsi)
            xorq    %rax, %rcx
            subq    $1, %rdx
            leaq    8(%rsi), %rsi
            leaq    8(%rdi), %rdi
    .L2:
            testq   %rdx, %rdx
            jg      .L3
            movq    %rcx, %rax
            ret

基于上述X86-64汇编代码，我们完成 ``copy.ys`` 的实现：

.. code-block:: asm

    # Execution begins at address 0
            .pos 0
            irmovq stack, %rsp      # Set up stack pointer
            call main               # Execute main program
            halt                    # Terminate program

    .align 8
    # Source block
    src:
            .quad 0x00a
            .quad 0x0b0
            .quad 0xc00

    # Destination block
    dest:
            .quad 0x111
            .quad 0x222
            .quad 0x333

    main:
            irmovq src,%rdi
            irmovq dest,%rsi
            irmovq $3,%rdx
            call copy_block      # copy_block(src, dest, len)
            ret

    # long copy_block(long *src, long *dest, long len)
    # src in %rdi, dest in %rsi, len in %rdx
    copy_block:
            irmovq $0, %rcx      # Init result
            irmovq $8, %r8       # Constant 8
            irmovq $1, %r9       # Constant 1
            jmp     test         # Goto test
    loop:
            mrmovq (%rdi),%rax   # Get *src
            rmmovq %rax,(%rsi)   # Set *dest
            xorq %rax, %rcx      # Compute checksum
            addq %r8, %rdi       # src++
            addq %r8, %rsi       # dest++
            subq %r9, %rdx       # len--
    test:
            andq %rdx,%rdx       # Is len > 0 ?
            jg    loop           # Stop when 0
            rrmovq %rcx, %rax    # Return checksum
            ret                  # Return

    # Stack starts here and grows to lower addresses
            .pos 0x200
    stack:

编译运行 ``copy.ys`` ：

.. code-block:: console

    $ ../misc/yas copy.ys # output copy.yo
    $ ../misc/yis copy.yo
    Stopped in 40 steps at PC = 0x13.  Status 'HLT', CC Z=1 S=0 O=0
    Changes to registers:
    %rax:   0x0000000000000000      0x0000000000000cba
    %rcx:   0x0000000000000000      0x0000000000000cba
    %rsp:   0x0000000000000000      0x0000000000000200
    %rsi:   0x0000000000000000      0x0000000000000048
    %rdi:   0x0000000000000000      0x0000000000000030
    %r8:    0x0000000000000000      0x0000000000000008
    %r9:    0x0000000000000000      0x0000000000000001

    Changes to memory:
    0x0030: 0x0000000000000111      0x000000000000000a
    0x0038: 0x0000000000000222      0x00000000000000b0
    0x0040: 0x0000000000000333      0x0000000000000c00
    0x01f0: 0x0000000000000000      0x000000000000006f
    0x01f8: 0x0000000000000000      0x0000000000000013
   
可以看到，原先位于 ``dest`` 的内存值，都被修改成 ``src`` 数组中的值。

Part B
^^^^^^^^

本部分的实验是在 ``sim/seq`` 目录下完成的。
实验的主要任务是根据课后习题4.51和4.52，修改Y84-64处理器的顺序实现的HCL文件 ``seq-full.hcl`` ，添加一条新的指令 ``iaddq`` 。

修改 ``seq-full.hcl``
''''''''''''''''''''''

根据练习习题4.3的说明，指令 ``iaddq`` 的编码格式如下所示。

.. image:: ./../_images/csapp/iaddq.png

再仿照 ``OPq`` 和 ``irmovq`` 指令的计算步骤，我们可以写出新添加的指令 ``iaddq`` 的计算步骤。

.. image:: ./../_images/csapp/iaddq_computation_steps.png 

明确了指令 ``iaddq`` 的在每一个阶段中的操作，我们就可以对文件 ``seq-full.hcl`` 进行相应的修改。

修改的patch如下所示。首先， ``seq-full.hcl`` 中已经为我们定义好了 ``iaddq`` 命令，但是并没有将其加入到 ``instr_valid`` 中。
所以我们首先把指令 ``iaddq`` 加入到其中。 
在取指阶段中， ``iaddq`` 指令既需要读取寄存器 ``rB`` 的值，也需要读取一个常量，所以同时需要 ``need_regids`` 和 ``need_valC`` 控制逻辑。
``iaddq`` 会将读取的寄存器 ``rB`` 的值存放在 ``srcB`` 中，并将计算过后的值存放在 ``dstE`` 中。
在执行阶段， ``iaddq`` 指令的两个输入 ``aluA`` 和 ``aluB`` 分别是 ``valC`` 和 ``valB`` ， ``alufun`` 计算功能保持默认加的操作。
同 ``addq`` 指令一样， ``iaddq`` 指令我们也需要根据指令的计算结果对条件码置位，即把 ``iaddq`` 添加到 ``set_cc`` 的控制逻辑里。

.. code-block:: console

        $ git diff seq/seq-full.hcl
        diff --git a/sim/seq/seq-full.hcl b/sim/seq/seq-full.hcl
        index 0c946dd..c71a82c 100644
        --- a/sim/seq/seq-full.hcl
        +++ b/sim/seq/seq-full.hcl
        @@ -106,16 +106,16 @@ word ifun = [

        bool instr_valid = icode in
                { INOP, IHALT, IRRMOVQ, IIRMOVQ, IRMMOVQ, IMRMOVQ,
        -              IOPQ, IJXX, ICALL, IRET, IPUSHQ, IPOPQ };
        +              IOPQ, IJXX, ICALL, IRET, IPUSHQ, IPOPQ, IIADDQ };

        # Does fetched instruction require a regid byte?
        bool need_regids =
                icode in { IRRMOVQ, IOPQ, IPUSHQ, IPOPQ,
        -                    IIRMOVQ, IRMMOVQ, IMRMOVQ };
        +                    IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ };

        # Does fetched instruction require a constant word?
        bool need_valC =
        -       icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IJXX, ICALL };
        +       icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IJXX, ICALL, IIADDQ };

        ################ Decode Stage    ###################################

        @@ -128,7 +128,7 @@ word srcA = [

        ## What register should be used as the B source?
        word srcB = [
        -       icode in { IOPQ, IRMMOVQ, IMRMOVQ  } : rB;
        +       icode in { IOPQ, IRMMOVQ, IMRMOVQ, IIADDQ  } : rB;
                icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
                1 : RNONE;  # Don't need register
        ];
        @@ -136,7 +136,7 @@ word srcB = [
        ## What register should be used as the E destination?
        word dstE = [
                icode in { IRRMOVQ } && Cnd : rB;
        -       icode in { IIRMOVQ, IOPQ} : rB;
        +       icode in { IIRMOVQ, IOPQ, IIADDQ} : rB;
                icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
                1 : RNONE;  # Don't write any register
        ];
        @@ -152,7 +152,7 @@ word dstM = [
        ## Select input A to ALU
        word aluA = [
                icode in { IRRMOVQ, IOPQ } : valA;
        -       icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ } : valC;
        +       icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ } : valC;
                icode in { ICALL, IPUSHQ } : -8;
                icode in { IRET, IPOPQ } : 8;
                # Other instructions don't need ALU
        @@ -161,7 +161,7 @@ word aluA = [
        ## Select input B to ALU
        word aluB = [
                icode in { IRMMOVQ, IMRMOVQ, IOPQ, ICALL,
        -                     IPUSHQ, IRET, IPOPQ } : valB;
        +                     IPUSHQ, IRET, IPOPQ, IIADDQ } : valB;
                icode in { IRRMOVQ, IIRMOVQ } : 0;
                # Other instructions don't need ALU
        ];
        @@ -173,7 +173,7 @@ word alufun = [
        ];

        ## Should the condition codes be updated?
        -bool set_cc = icode in { IOPQ };
        +bool set_cc = icode in { IOPQ, IIADDQ };

        ################ Memory Stage    ###################################


完成对硬件描述文件 ``seq-full.hcl`` 的修改，我们可通过以下命令编译生成 ``seq-full`` 版本的Y86-64处理器。

.. code-block:: console

        $ make VERSION=full 

我们可用新生成的 ``ssim`` 测试一个包含 ``iaddq`` 指令的Y86-64小程序。可以看到，程序 ``asumi.yo`` 运行成功。

.. code-block:: console

        $ ./ssim -t ../y86-code/asumi.yo
        Y86-64 Processor: seq-full.hcl
        137 bytes of code read
        IF: Fetched irmovq at 0x0.  ra=----, rb=%rsp, valC = 0x100
        IF: Fetched call at 0xa.  ra=----, rb=----, valC = 0x38
        Wrote 0x13 to address 0xf8
        IF: Fetched irmovq at 0x38.  ra=----, rb=%rdi, valC = 0x18
        IF: Fetched irmovq at 0x42.  ra=----, rb=%rsi, valC = 0x4
        IF: Fetched call at 0x4c.  ra=----, rb=----, valC = 0x56
        Wrote 0x55 to address 0xf0
        IF: Fetched xorq at 0x56.  ra=%rax, rb=%rax, valC = 0x0
        IF: Fetched andq at 0x58.  ra=%rsi, rb=%rsi, valC = 0x0
        IF: Fetched jmp at 0x5a.  ra=----, rb=----, valC = 0x83
        IF: Fetched jne at 0x83.  ra=----, rb=----, valC = 0x63
        IF: Fetched mrmovq at 0x63.  ra=%r10, rb=%rdi, valC = 0x0
        IF: Fetched addq at 0x6d.  ra=%r10, rb=%rax, valC = 0x0
        IF: Fetched iaddq at 0x6f.  ra=----, rb=%rdi, valC = 0x8
        IF: Fetched iaddq at 0x79.  ra=----, rb=%rsi, valC = 0xffffffffffffffff
        IF: Fetched jne at 0x83.  ra=----, rb=----, valC = 0x63
        IF: Fetched mrmovq at 0x63.  ra=%r10, rb=%rdi, valC = 0x0
        IF: Fetched addq at 0x6d.  ra=%r10, rb=%rax, valC = 0x0
        IF: Fetched iaddq at 0x6f.  ra=----, rb=%rdi, valC = 0x8
        IF: Fetched iaddq at 0x79.  ra=----, rb=%rsi, valC = 0xffffffffffffffff
        IF: Fetched jne at 0x83.  ra=----, rb=----, valC = 0x63
        IF: Fetched mrmovq at 0x63.  ra=%r10, rb=%rdi, valC = 0x0
        IF: Fetched addq at 0x6d.  ra=%r10, rb=%rax, valC = 0x0
        IF: Fetched iaddq at 0x6f.  ra=----, rb=%rdi, valC = 0x8
        IF: Fetched iaddq at 0x79.  ra=----, rb=%rsi, valC = 0xffffffffffffffff
        IF: Fetched jne at 0x83.  ra=----, rb=----, valC = 0x63
        IF: Fetched mrmovq at 0x63.  ra=%r10, rb=%rdi, valC = 0x0
        IF: Fetched addq at 0x6d.  ra=%r10, rb=%rax, valC = 0x0
        IF: Fetched iaddq at 0x6f.  ra=----, rb=%rdi, valC = 0x8
        IF: Fetched iaddq at 0x79.  ra=----, rb=%rsi, valC = 0xffffffffffffffff
        IF: Fetched jne at 0x83.  ra=----, rb=----, valC = 0x63
        IF: Fetched ret at 0x8c.  ra=----, rb=----, valC = 0x0
        IF: Fetched ret at 0x55.  ra=----, rb=----, valC = 0x0
        IF: Fetched halt at 0x13.  ra=----, rb=----, valC = 0x0
        32 instructions executed
        Status = HLT
        Condition Codes: Z=1 S=0 O=0
        Changed Register State:
        %rax:   0x0000000000000000      0x0000abcdabcdabcd
        %rsp:   0x0000000000000000      0x0000000000000100
        %rdi:   0x0000000000000000      0x0000000000000038
        %r10:   0x0000000000000000      0x0000a000a000a000
        Changed Memory State:
        0x00f0: 0x0000000000000000      0x0000000000000055
        0x00f8: 0x0000000000000000      0x0000000000000013
        ISA Check Succeeds

我们再用 ``ssim`` 运行Y84-64的基准测试程序，来确保我们新添加的 ``iaddq`` 指令不引入任何错误。
可以看到，所有的Y86-64基准测试程序运行无误。

.. code-block:: console

        $ cd ../y86-code
        $ make testssim
        ../seq/ssim -t asum.yo > asum.seq
        ../seq/ssim -t asumr.yo > asumr.seq
        ../seq/ssim -t cjr.yo > cjr.seq
        ../seq/ssim -t j-cc.yo > j-cc.seq
        ../seq/ssim -t poptest.yo > poptest.seq
        ../seq/ssim -t pushquestion.yo > pushquestion.seq
        ../seq/ssim -t pushtest.yo > pushtest.seq
        ../seq/ssim -t prog1.yo > prog1.seq
        ../seq/ssim -t prog2.yo > prog2.seq
        ../seq/ssim -t prog3.yo > prog3.seq
        ../seq/ssim -t prog4.yo > prog4.seq
        ../seq/ssim -t prog5.yo > prog5.seq
        ../seq/ssim -t prog6.yo > prog6.seq
        ../seq/ssim -t prog7.yo > prog7.seq
        ../seq/ssim -t prog8.yo > prog8.seq
        ../seq/ssim -t ret-hazard.yo > ret-hazard.seq
        grep "ISA Check" *.seq
        asumr.seq:ISA Check Succeeds
        asum.seq:ISA Check Succeeds
        cjr.seq:ISA Check Succeeds
        j-cc.seq:ISA Check Succeeds
        poptest.seq:ISA Check Succeeds
        prog1.seq:ISA Check Succeeds
        prog2.seq:ISA Check Succeeds
        prog3.seq:ISA Check Succeeds
        prog4.seq:ISA Check Succeeds
        prog5.seq:ISA Check Succeeds
        prog6.seq:ISA Check Succeeds
        prog7.seq:ISA Check Succeeds
        prog8.seq:ISA Check Succeeds
        pushquestion.seq:ISA Check Succeeds
        pushtest.seq:ISA Check Succeeds
        ret-hazard.seq:ISA Check Succeeds
        rm asum.seq asumr.seq cjr.seq j-cc.seq poptest.seq pushquestion.seq pushtest.seq prog1.seq prog2.seq prog3.seq prog4.seq prog5.seq prog6.seq prog7.seq prog8.seq ret-hazard.seq


最后我们再进行最终的回归测试，首先对除去 ``iaddq`` 指令的 ``ssim`` 进行测试。测试结果如下所示，600条ISA检查通过。

.. code-block:: console

        $ cd ../ptest
        $ make SIM=../seq/ssim
        ./optest.pl -s ../seq/ssim
        Simulating with ../seq/ssim
        All 49 ISA Checks Succeed
        ./jtest.pl -s ../seq/ssim
        Simulating with ../seq/ssim
        All 64 ISA Checks Succeed
        ./ctest.pl -s ../seq/ssim
        Simulating with ../seq/ssim
        All 22 ISA Checks Succeed
        ./htest.pl -s ../seq/ssim
        Simulating with ../seq/ssim
        All 600 ISA Checks Succeed


然后再单独对 ``iaddq`` 进行回归测试。测试结果如下所示，756条ISA检查通过。

.. code-block:: console

        $ cd ../ptest
        $ make SIM=../seq/ssim TFLAGS=-i
        ./optest.pl -s ../seq/ssim -i
        Simulating with ../seq/ssim
        All 58 ISA Checks Succeed
        ./jtest.pl -s ../seq/ssim -i
        Simulating with ../seq/ssim
        All 96 ISA Checks Succeed
        ./ctest.pl -s ../seq/ssim -i
        Simulating with ../seq/ssim
        All 22 ISA Checks Succeed
        ./htest.pl -s ../seq/ssim -i
        Simulating with ../seq/ssim
        All 756 ISA Checks Succeed

至此，Part B实验完成。