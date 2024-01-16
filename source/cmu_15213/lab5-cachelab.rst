Lab 5: Cache Lab
=================

0x01. 实验介绍
------------------

本实验将帮助我们理解缓存系统对于C程序性能的影响。

本实验将分为两个部分。在Part A中，我们将要写一个200-300行的C程序，用来模拟缓存系统的工作流程。
在Part B中，我们将对一个矩阵转置函数进行优化，从而使其的缓存不命中的数目降到最低。

总之，通过此实验，我们将会加深对计算机缓存系统工作原理的理解，同时对缓存系统如何影响程序性能有更直观的认识。


0x02. 实验环境搭建
-------------------

.. code-block: console

    $ wget http://csapp.cs.cmu.edu/3e/cachelab-handout.tar
    $ tar xvf cachelab-handout.tar

进入解压后的实验目录，对于Part A，我们只需要修改 ``csim.c`` 文件。
对于Part B，我们只需要修改 ``trans.c`` 文件。

可通过以下命令编译以上文件：

.. code-block: console

    $ make clean
    $ make

实验目录里有一个 ``traces`` 子目录，包含着一组由 ``valgrind`` 生成的内存访问记录的文件。

.. code-block: console

    $ valgrind --log-fd=1 --tool=lackey -v --trace-mem=yes ls -l

上述命令将会顺序记录 ``ls -l`` 命令每次访存的信息，生成如下所示的内容：

.. code-block: console

    I 0400d7d4,8
     M 0421c7f0,4
     L 04f6b868,8
     S 7ff0005c8,8

每行访存记录的格式为 ``[space]operation address,size`` 。
其中 ``operation`` 表明访存的类型： ``I`` 表示指令加载（instruction load）， ``L`` 表示数据加载（data load）， ``S`` 表示数据存储（data store）， ``M`` 表示数据修改（data modify），即一个数据存储后借一个数据加载。
``address`` 则表示的是访问的64位地址的十六进制数。
``size`` 访问的数据的大小。
需要注意的时，上述访存记录的格式中，除了指令加载 ``I`` 外，其它类型的开头都有一个空格。这个信息在后续对我们实现缓存系统模拟器会有帮助。


0x03. 实验过程及思路说明
-----------------------------

Part A
^^^^^^^^^

实验描述及注意事项
''''''''''''''''''''

本实验中，我们将在 ``csim.c`` 里实现一个缓存模拟器，以 ``valgrind`` 生成的访存记录作为输入，模拟缓存命中或不命中的行为，最后输出总的缓存命中，缓存不命中以及缓存驱逐的数目。

实验提供了一个名为 ``csim-ref`` 的参考版本的缓存模拟器。当出现缓存驱逐的情况时，它采用的是LRU(Least-recently used)的策略。

``csim-ref`` 的使用如下所示：

.. code-block: console

    Usage: ./csim-ref [-hv] -s <s> -E <E> -b <b> -t <tracefile>


* ``-h`` : helper参数，输出 ``csim-ref`` 的使用方法
* ``-v`` : verbose模式，输出每条访存的缓存行为
* ``-s <s>`` : 缓存组索引的比特数（ ``S = 2 ^ s`` 为缓存组数目）
* ``-E <E>`` : 每个缓存组中的 ``cacheline`` 数目
* ``-b <b>`` : cacheline中block的比特数（ ``B = 2 ^ b`` 为block的大小）
* ``-t <tracefile>`` : 输入的访存记录文件

我们可根据上述的描述，运行 ``csim-ref`` 来对其有个直观的感受：

.. code-block: console

    $ ./csim-ref -s 4 -E 1 -b 4 -t traces/yi.trace
    hits:4 misses:5 evictions:3
    $ ./csim-ref -s 4 -E 1 -b 4 -t traces/yi.trace -v
    L 10,1 miss
    M 20,1 miss hit
    L 22,1 hit
    S 18,1 hit
    L 110,1 miss eviction
    L 210,1 miss eviction
    M 12,1 miss eviction hit
    hits:4 misses:5 evictions:3

上述 ``csim-ref`` 运行的结果，即是我们在 ``csim.c`` 所要实现的最终效果。

对于Part A，实验有一些前提和编程规则上的要求：

* 我们实现的缓存模拟器必须对任意的 ``s`` ， ``E`` 和 ``b`` 都适用，因此我们需用 ``malloc`` 函数来对相关数据结构的内存进行分配
* 本实验中我们只关心数据缓存的性能，所以对访存记录文件中的 ``I`` 指令访存记录不做处理
* ``main`` 函数最后必须调用函数 ``printSummary()`` 来输出缓存模拟器的命中，不命中和驱逐数目
* 实验中的访存记录中的内存地址都是对齐的，所以记录中的 ``size`` 可以在处理中被忽略


``csim`` 实现
''''''''''''''''

