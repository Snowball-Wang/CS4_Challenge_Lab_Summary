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

.. code-block:: console

    $ wget http://csapp.cs.cmu.edu/3e/cachelab-handout.tar
    $ tar xvf cachelab-handout.tar

进入解压后的实验目录，对于Part A，我们只需要修改 ``csim.c`` 文件。
对于Part B，我们只需要修改 ``trans.c`` 文件。

可通过以下命令编译以上文件：

.. code-block:: console

    $ make clean
    $ make

实验目录里有一个 ``traces`` 子目录，包含着一组由 ``valgrind`` 生成的内存访问记录的文件。

.. code-block:: console

    $ valgrind --log-fd=1 --tool=lackey -v --trace-mem=yes ls -l

上述命令将会按顺序记录 ``ls -l`` 命令每次访存的信息，生成如下所示的内容：

.. code-block:: console

    I 0400d7d4,8
     M 0421c7f0,4
     L 04f6b868,8
     S 7ff0005c8,8

每行访存记录的格式为 ``[space]operation address,size`` 。
其中 ``operation`` 表明访存的类型： ``I`` 表示指令加载（instruction load）， ``L`` 表示数据加载（data load）， ``S`` 表示数据存储（data store）， ``M`` 表示数据修改（data modify），即一个数据存储后接一个数据加载。
``address`` 则表示的是访问的64位地址的十六进制数。
``size`` 访问的数据的大小。
需要注意的时，上述访存记录的格式中，除了指令加载 ``I`` 外，其它类型的开头都有一个空格。这个信息后续对我们实现缓存系统模拟器会有帮助。


0x03. 实验过程及思路说明
-----------------------------

Part A
^^^^^^^^^

实验描述及注意事项
''''''''''''''''''''

本实验中，我们将在 ``csim.c`` 里实现一个缓存模拟器，以 ``valgrind`` 生成的访存记录作为输入，模拟缓存命中或不命中的行为，最后输出总的缓存命中，缓存不命中以及缓存行驱逐的数目。

实验提供了一个名为 ``csim-ref`` 的参考版本的缓存模拟器。当出现缓存行驱逐的情况时，它采用的是LRU(Least-recently used)的策略。

``csim-ref`` 的使用如下所示：

.. code-block:: console

    Usage: ./csim-ref [-hv] -s <s> -E <E> -b <b> -t <tracefile>


* ``-h`` : helper参数，输出 ``csim-ref`` 的使用方法
* ``-v`` : verbose模式，输出每条访存的缓存行为
* ``-s <s>`` : 缓存组索引的比特数（ ``S = 2 ^ s`` 为缓存组数目）
* ``-E <E>`` : 每个缓存组中的 ``cacheline`` 数目
* ``-b <b>`` : cacheline中block的比特数（ ``B = 2 ^ b`` 为block的大小）
* ``-t <tracefile>`` : 输入的访存记录文件

我们可根据上述的描述，运行 ``csim-ref`` 来对其有个直观的感受：

.. code-block:: console

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

在实现 ``csim`` 之前，我们可在gdb中通过指令 ``disassemble main`` 来查看 ``csim-ref`` 中 ``main`` 函数中所调用的函数。
我们的 ``csim`` 缓存模拟器的设计，可从 ``csim-ref`` 中的 ``main`` 函数获取灵感。

.. code-block:: console

    $ gdb ./csim-ref
    (gdb) disassemble main
    ...
    call   0x400a90 <atoi@plt>
    ...
    call   0x401202 <printUsage>
    ...
    call   0x400a80 <getopt@plt>
    ...
    call   0x400be0 <initCache>
    ...
    call   0x40105c <replayTrace>
    ...
    call   0x400d81 <freeCache>
    ...
    call   0x401494 <printSummary>

上述已将不重要的二进制代码剔除，提取出 ``csim-ref`` 中 ``main`` 函数所调用的函数。
由此，我们可以对 ``csim-ref`` 中 ``main`` 函数做的事进行总结：

* 调用 ``getopt`` 函数对命令行参数进行解析
* 当命令行参数包含 ``-h`` 时，调用 ``printUsage`` 函数对用户进行提示
* 基于解析成功的缓存参数，调用 ``initCache`` 函数对缓存进行初始化
* 调用 ``replayTrace`` 函数，对每条valgrind的memory trace进行处理，判断其是缓存命中、不命中还是其它
* 调用 ``freeCache`` 释放创建缓存使用到的内存


实现命令行参数解析和 ``printUsage`` 函数
""""""""""""""""""""""""""""""""""""""""

首先用 ``getopt`` 函数对 ``csim`` 进行命令行参数解析。
需要对 ``Makefile`` 中的 ``CFLAGS`` 参数进行修改，将 ``-std=c99`` 替换成 ``-std=gnu99`` ，这样 ``getopt`` 函数才能从 ``unistd.h`` 中引入 ``getopt.h`` 头文件。
`stackoverflow <https://stackoverflow.com/questions/22575940/getopt-not-included-implicit-declaration-of-function-getopt>`_ 上有关于此问题的回答和介绍。

其次，我们也需要考虑 ``printUsage`` 函数的使用场景。应该包括两个场景：一是命令行中出现 ``-h`` 时，即用户显式地查询 ``csim`` 的使用规则。另一个是当用户输入的命令行参数缺失或者错误时，提示用户使用正确的参数。
我们可以定义一个结构体用来存放命令行参数解析的结果，结构体的定义如下所示：

.. code-block:: c

    typedef struct config{
        unsigned int log_set_num;
        unsigned int lines_per_set;
        unsigned int log_blk_off;
        int verbose;
        char *trace_file;
    } config_t;

在 ``main`` 函数中，我们可以通过调用 ``configInit`` 函数来完成对命令行参数的解析，再通过 ``configCheck`` 函数来检查用户输入的参数是否有效，无效时调用 ``printUsage`` 提示用户。
两个函数的具体实现如下：

.. code-block:: c

    /* parse the argument */
    int configInit(config_t *config, int argc, char *argv[])
    {
        char c = 0;
        while((c = getopt(argc, argv, "s:E:b:t:vh")) != -1)
        {
            switch(c)
            {
                case 's':
                    config->log_set_num = atoi(optarg);
                    break;
                case 'E':
                    config->lines_per_set = atoi(optarg);
                    break;
                case 'b':
                    config->log_blk_off = atoi(optarg);
                    break;
                case 't':
                    config->trace_file = optarg;
                    break;
                case 'v':
                    config->verbose = 1;
                    break;
                case 'h':
                    printUsage();
                    exit(EXIT_SUCCESS);
                default:
                    fprintf(stderr, "unknown options\n");
                    exit(EXIT_FAILURE);
            }
        }
        return 0;
    }

    /* check the mandatory arguments */
    void configCheck(config_t *config)
    {
        /* set number, lines per set, block offset and trace file */
        /* have to be assigned */
        if(config->log_set_num == 0 || 
            config->lines_per_set == 0 || 
            config->log_blk_off == 0 ||
            config->trace_file == NULL)
        {
            printf("./csim: Missing required command line argument\n");
            printUsage();
            exit(EXIT_FAILURE);
        }
    }


使用 ``malloc`` 初始化缓存
"""""""""""""""""""""""""""""

完成了命令行参数的解析后，我们需要根据解析的命令行参数初始化缓存。本质上，我们模拟的缓存是一个二维数组。第一维存放的是每个组（set）的指针，第二维则对应的是每个组的缓存行（cacheline）。
每个缓存行由有效位（valid bit）、标记位（tag bits）和缓存块（block）组成。
考虑到要根据 ``LRU`` （Least-recently used）的规则更新缓存，我们可通过比较缓存行访问的时间戳来实现。并且我们并不需要获取缓存块存储的数据，只是模拟缓存访存的行为，由此我们可将缓存行定义成如下结构体：

.. code-block:: c

    /* Cacheline structure */
    typedef struct cacheline{
        unsigned int valid_bit;
        unsigned long tag_bits;
        clock_t timestamp;
    } cacheline_t;

基于以上定义，我们可在 ``initCache`` 函数利用 ``malloc`` 分配二维数组缓存的内存：

.. code-block:: c

    /* initialize 2d cache memory */
    cacheline_t** initCache(config_t *config)
    {
        int set_num = pow(2, config->log_set_num);
        cacheline_t **cache = (cacheline_t **)malloc(sizeof(cacheline_t *) * set_num);
        for(int i = 0; i < set_num; i++)
        {
            cache[i] = (cacheline_t *)malloc(config->lines_per_set * sizeof(cacheline_t));
            /* zero the cacheline */
            memset(cache[i], 0, config->lines_per_set * sizeof(cacheline_t));
        }

        return cache;
    }



实现 ``replayTrace`` 模拟缓存行为
"""""""""""""""""""""""""""""""""""""""""

完成了命令行参数解析和缓存的创建，接下来便是此实验最为核心的部分，对valgrind生成的每条访存记录（memory trace）软件模拟访存行为。

访存最终有三种结果，缓存命中（cache hit）、缓存不命中（cache miss）和缓存行驱逐（eviction）。我们可将三种结果定义在如下结构体中：

.. code-block:: c

    typedef struct result{
        unsigned int hits;
        unsigned int misses;
        unsigned int evictions;
    } result_t;

在实现 ``replayTrace`` 函数之前，我们先用语言描述一下函数的流程：

1. 逐行读取文件中的访存记录，并对每条访存记录进行解析
2. 从访存记录的指令和地址解析得到访存指令、组索引和标记位
3. 忽略类型为 ``I`` 的指令，处理 ``L`` 、 ``S`` 和 ``M`` 指令
        3.1 遍历当前缓存组中的缓存行
            * 如果当前缓存行的有效位为1，且标记位与当前标记位一致，则缓存命中，同时更新当前缓存行的时间戳，函数返回
            * 如果当前缓存行的有效位为0，则需标记当前缓存组内有空闲缓存行，并且记录第一个空闲缓存行的索引
            * 以上不满足，查询当前缓存组的下一个缓存行
        3.2 如果解析得到的标记位与当前缓存组中的所有缓存行的标记位都不匹配
            * 如果缓存组内有空闲缓存行，则缓存不命中，更新当前缓存行的有效位、标记位和时间戳
            * 如果没有空闲缓存行，则缓存不命中并且要进行缓存行驱逐
                * 遍历组内缓存行，根据LRU找到对应缓存行的时间戳
                * 再次遍历组内缓存行，找到以上时间戳的缓存行，更新其标记位和时间戳

指令 ``L`` 和 ``S`` 的处理逻辑基本一致，指令 ``M`` 等同于指令 ``L`` 和 ``S`` 的组合。
接下来就是用代码来实现我们上述语言表述的逻辑。为了方便，我们定义了以下结构体来表示解析得到的访存记录的信息：

.. code-block:: c

    typedef struct mem_trace{
        char op;
        unsigned long addr;
        unsigned int blk_off;
    } mem_trace_t;

函数 ``replayTrace`` 的实现如下：

.. code-block:: c

    /* implement the cache simulator */
    void replayTrace(config_t *config, cacheline_t **cache, result_t *result)
    {
        mem_trace_t mem_trace = { 0 };
        mem_trace_t *ptrace = &mem_trace;

        char *line = NULL;
        size_t len = 0;
        ssize_t nread = 0;

        unsigned int ci = 0;
        unsigned int ct = 0;

        /* open the trace file */
        FILE *fp = fopen(config->trace_file, "r");
        if(!fp)
        {
            fprintf(stderr, "unable to open trace file!\n");
            exit(EXIT_FAILURE);
        }

        /* read each line and parse it */
        while((nread = getline(&line, &len, fp)) != -1)
        {
            /* skip instruction load operation */
            if(line[0] == 'I')
                continue;

            /* parse the memory trace */
            sscanf(line, " %c %lx,%u", &ptrace->op, &ptrace->addr, &ptrace->blk_off);
            /* calculate cache set index and cache tag bits */
            ci = (ptrace->addr >> config->log_blk_off) & ((1 << config->log_set_num) - 1);
            ct = ptrace->addr >> (config->log_blk_off + config->log_set_num);
            if(ptrace->op == 'L' || ptrace->op == 'S')
            {
                /* do actual work in handleOperation */
                handleOperation(config, cache, ptrace, ci, ct, result);
            }
            else
            {
                /* if op is 'M', it is load + store */
                handleOperation(config, cache, ptrace, ci, ct, result);
                handleOperation(config, cache, ptrace, ci, ct, result);
            }
        }

        free(line);
        fclose(fp);
    }

    /* cache behavior simulation */
    void handleOperation(config_t *config, cacheline_t **cache, mem_trace_t *trace, unsigned int ci, unsigned int ct, result_t *result)
    {
        int empty = 0;
        int empty_index = 0;
        for(int i = 0; i < config->lines_per_set; i++)
        {
            /* cache hit */
            if(cache[ci][i].valid_bit == 1 && cache[ci][i].tag_bits == ct)
            {
                result->hits += 1;
                /* remember to update the timestamp */
                cache[ci][i].timestamp = clock();
                if(config->verbose)
                    printf("%c %lx,%u hit\n", trace->op, trace->addr, trace->blk_off);
                return;
            }
            /* cache miss but empty cachelines are available */
            else if(cache[ci][i].valid_bit == 0)
            {
                /* if empty flag is set, no need to do it again */
                if(empty == 0)
                {
                    empty = 1;
                    empty_index = i;
                }
            }
            else
                continue;
        }

        /* cache miss */
        if(empty)
        {
            cache[ci][empty_index].valid_bit = 1;
            cache[ci][empty_index].tag_bits = ct;
            cache[ci][empty_index].timestamp = clock();
            if(config->verbose)
                printf("%c %lx,%u miss\n", trace->op, trace->addr, trace->blk_off);
            result->misses += 1;
        }
        /* cache miss and eviction */
        else
        {
            /* find the LRU cacheline */
            clock_t t0 = cache[ci][0].timestamp;
            for(int i = 0; i < config->lines_per_set; i++)
            {
                if(cache[ci][i].timestamp != 0 && t0 >= cache[ci][i].timestamp)
                    t0 = cache[ci][i].timestamp;
            }
            /* evict the victim */
            for(int i = 0; i < config->lines_per_set; i++)
            {
                if(cache[ci][i].timestamp == t0)
                {
                    /* update cacheline tag bits and timestamp */
                    cache[ci][i].tag_bits = ct;
                    cache[ci][i].timestamp = clock();
                }
            }
            if(config->verbose)
                printf("%c %lx, %u miss eviction\n", trace->op, trace->addr, trace->blk_off);
            result->misses += 1;
            result->evictions += 1;
        }
    }

释放内存
"""""""""""""""

程序的最后我们需通过 ``free`` 函数对分配的二维数组缓存的内存进行释放：

.. code-block:: c

    /* free 2d cache memory */
    void freeCache(config_t *config, cacheline_t **cache)
    {
        int set_num = pow(2, config->log_set_num);
        /* free each cachelines in the set */
        for(int i = 0; i < set_num; i++)
            free(cache[i]);
        
        /* free the cache */
        free(cache)
    }

最终的 ``main`` 函数的执行流程如下所示：

.. code-block:: c

    int main(int argc, char *argv[])
    {
        config_t config = { 0 };
        result_t result = { 0 };

        /* init configuration */
        configInit(&config, argc, argv);

        /* check mandatory arguments */
        configCheck(&config);

        /* init the cache */
        cacheline_t **cache = initCache(&config);

        /* cache simulator */
        replayTrace(&config, cache, &result);

        /* print the result */
        printSummary(result.hits, result.misses, result.evictions);

        /* free the cache */
        freeCache(&config, cache);

        return 0;
    }

程序运行结果
"""""""""""""

``make csim`` 编译自定义的缓存模拟器，运行 ``./test-csim`` 查看运行结果：

.. code-block:: console

    $ ./test-csim
                            Your simulator     Reference simulator
    Points (s,E,b)    Hits  Misses  Evicts    Hits  Misses  Evicts
        3 (1,1,1)       9       8       6       9       8       6  traces/yi2.trace
        3 (4,2,4)       4       5       2       4       5       2  traces/yi.trace
        3 (2,1,4)       2       3       1       2       3       1  traces/dave.trace
        3 (2,1,3)     167      71      67     167      71      67  traces/trans.trace
        3 (2,2,3)     201      37      29     201      37      29  traces/trans.trace
        3 (2,4,3)     212      26      10     212      26      10  traces/trans.trace
        3 (5,1,5)     231       7       0     231       7       0  traces/trans.trace
        6 (5,1,5)  265189   21775   21743  265189   21775   21743  traces/long.trace
        27

TEST_CSIM_RESULTS=27

至此，Part A部分顺利完成！