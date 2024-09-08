Lab 6: Shell Lab
=================

0x01. 实验介绍
------------------

本实验将帮助我们熟悉并理解进程控制和信号处理的概念。
我们将实现一个支持任务控制的简单的Unix shell。

0x02. 实验环境搭建
------------------

.. code-block:: console

    $ wget http://csapp.cs.cmu.edu/3e/shlab-handout.tar
    $ tar xvf shlab-handout.tar

下载实验压缩包，解压后进入目录。本实验我们只需修改 ``tsh.c`` 文件来实现一个简单的Unix Shell。
实验已经在 ``tsh.c`` 中搭建好了一个简单的Unix Shell的框架，我们只需按照要求实现以下函数：

* ``eval`` 解析 ``tsh`` 的命令行参数
* ``builtin_cmd`` 解析 ``tsh`` 的内置命令参数： ``quit`` ， ``fg`` ， ``bg`` 和 ``jobs``
* ``do_bgfg`` 实现Shell的 ``bg`` 和 ``fg`` 内置命令特性
* ``waitfg`` 实现 ``tsh`` 等待前台任务结束的功能
* ``sigchld_handler`` 捕捉 ``SIGCHLD`` 信号
* ``sigint_handler`` 捕捉 ``SIGINT`` （ctrl-c） 信号
* ``sigtstp_handler`` 捕捉 ``SIGTSTP`` （ctrl-z）信号

实验中包含了一个样本shell ``tshref`` 供我们参考与对比。
与之前的实验一样，我们可通过命令 ``make`` 来编译实验， ``make clean`` 清理实验。

0x03. 实验过程及思路说明
-----------------------------

``tsh.c`` 框架梳理
^^^^^^^^^^^^^^^^^^^^

在动手写代码之前，我们首先需要结合 ``tsh.c`` ，弄清楚要实现的Unix Shell的框架。
在此基础上，理解每个待实现的函数所要完成的功能和特性。

首先看一下 ``main`` 函数。函数首先对传入 ``tsh`` 的参数进行解析。 ``-v`` 是冗余输出， ``-p`` 是不打印 ``tsh>`` 提示。
接下来四个信号的信号处理函数的安装。然后调用 ``initjobs`` 来完成任务的初始化。
再然后读取用户输入 ``tsh`` 里的命令行参数到 ``cmdline`` 中，最后调用 ``eval`` 函数解析 ``cmdline`` 。

.. code-block:: c

    /*
    * main - The shell's main routine 
    */
    int main(int argc, char **argv) 
    {
        char c;
        char cmdline[MAXLINE];
        int emit_prompt = 1; /* emit prompt (default) */

        /* Redirect stderr to stdout (so that driver will get all output
        * on the pipe connected to stdout) */
        dup2(1, 2);

        /* Parse the command line */
        while ((c = getopt(argc, argv, "hvp")) != EOF) {
            switch (c) {
            case 'h':             /* print help message */
                usage();
            break;
            case 'v':             /* emit additional diagnostic info */
                verbose = 1;
            break;
            case 'p':             /* don't print a prompt */
                emit_prompt = 0;  /* handy for automatic testing */
            break;
        default:
                usage();
        }
        }

        /* Install the signal handlers */

        /* These are the ones you will need to implement */
        Signal(SIGINT,  sigint_handler);   /* ctrl-c */
        Signal(SIGTSTP, sigtstp_handler);  /* ctrl-z */
        Signal(SIGCHLD, sigchld_handler);  /* Terminated or stopped child */

        /* This one provides a clean way to kill the shell */
        Signal(SIGQUIT, sigquit_handler); 

        /* Initialize the job list */
        initjobs(jobs);

        /* Execute the shell's read/eval loop */
        while (1) {

        /* Read command line */
        if (emit_prompt) {
            printf("%s", prompt);
            fflush(stdout);
        }
        if ((fgets(cmdline, MAXLINE, stdin) == NULL) && ferror(stdin))
            app_error("fgets error");
        if (feof(stdin)) { /* End of file (ctrl-d) */
            fflush(stdout);
            exit(0);
        }

        /* Evaluate the command line */
        eval(cmdline);
        fflush(stdout);
        fflush(stdout);
        } 

        exit(0); /* control never reaches here */
    }    

上述中涉及到一个概念 --- 任务（job）。任务的定义如下，简单来说，就是shell创建子进程，在子进程中完成用户输入的命令。

    The child processes created as a result of interpreting a single command line are known collectively as a job.
    In general, a job can consist of multiple child processes connected by Unix pipes.

``tsh.c`` 中提供了以下关于任务增添删改的函数供我们使用：

.. code-block:: c

    void clearjob(struct job_t *job);
    void initjobs(struct job_t *jobs);
    int maxjid(struct job_t *jobs); 
    int addjob(struct job_t *jobs, pid_t pid, int state, char *cmdline);
    int deletejob(struct job_t *jobs, pid_t pid); 
    pid_t fgpid(struct job_t *jobs);
    struct job_t *getjobpid(struct job_t *jobs, pid_t pid);
    struct job_t *getjobjid(struct job_t *jobs, int jid); 
    int pid2jid(pid_t pid); 
    void listjobs(struct job_t *jobs);

针对 ``eval`` 函数的实现，我们需要完成以下功能：

* Step 1: 解析 ``cmdline`` 参数，判断命令行参数是内置命令还是用户程序
* Step 2: 如果是内置命令，则直接执行命令；若为用户程序，判断命令为前台命令还是后台命令，然后创建子进程，将当前子进程添加到任务列表
* Step 3: 若命令为前台命令，则当前Shell父进程需等待子进程完成后再处理下一个命令行参数

上述三个步骤，我们将分别调用 ``builtin_cmd`` ， ``do_bgfg`` 以及 ``waitfg`` 来实现相应的功能。
同时，在 ``eval`` 函数的实现过程中，还有一些细节的问题需要我们考虑：

首先是可能出现的同步问题。如同书中8.40的例子，父进程在 ``fork`` 子进程之前，应当使用 ``sigprocmask`` 来屏蔽掉SIGCHLD信号。
当把子进程加入到任务列表时，也需要屏蔽所有的信号，完成后再解除屏蔽。创建的子进程会继承父进程的屏蔽的信号，所以在执行命令前，需解除相应的信号屏蔽。

其次是SIGINT信号处理函数的作用域问题。 ``tsh`` 是作为前台进程运行在系统中的，所创建的子进程默认和 ``tsh`` 归属于一个前台进程组（foreground process group）。
所以当我们敲入 ``ctrl-c` 想要终止运行在 ``tsh`` 的程序时， ``tsh`` 本身也会终止运行。
实验提供了一种规避的方法，即调用 ``setpgid(0,0)`` 将新创建的子进程的进程组ID设置为其本身的进程ID。这样Shell就能够捕捉到 ``SIGINT`` 信号并将其转发到相应的前台进程组。

最后是关于信号处理函数的小细节。当我们实现对应 ``SIGINT`` 和 ``SIGTSTP`` 的信号处理函数时，我们应处理整个前台进程组，即传给 ``kill`` 函数应该是 ``-pid`` ，而不是 ``pid`` 。

理清楚了 ``tsh.c`` 的框架和细节，接下来我们可参照书中的实例代码，完成 ``tsh`` 的实现。

``tsh`` 代码实现
^^^^^^^^^^^^^^^^^^^^

``eval`` 函数
''''''''''''''''''''

参考书中8.24和8.40的实例代码，函数 ``eval`` 的实现如下：

.. code-block:: c

    void eval(char *cmdline) 
    {
        char *argv[MAXARGS];	/* Argument list execve() */
        char buf[MAXLINE];		/* Holds modified command line */
        int bg;					/* Should the job run in bg or fg? */
        pid_t pid;				/* Process id */

        sigset_t mask, mask_all, prev_mask;
        sigfillset(&mask_all);
        sigemptyset(&mask);
        sigaddset(&mask, SIGCHLD);

        strcpy(buf, cmdline);
        bg = parseline(cmdline, argv);
        /* Ignore empty lines */
        if (argv[0] == NULL)
            return;

        /* if command is builtin command ? */
        if (!builtin_cmd(argv))
        {
            /* Block SIGCHLD */
            sigprocmask(SIG_BLOCK, &mask, &prev_mask);
            pid = fork();
            if (pid < 0)
            {
                unix_error("fork error");
                exit(1);
            }
            /* child process*/
            else if (pid == 0)
            {
                /* Unblock SIGCHLD */
                sigprocmask(SIG_SETMASK, &prev_mask, NULL);
                /* Change group ID to its own */
                setpgid(0, 0);
                if (execve(argv[0], argv, environ) < 0)
                {
                    printf("%s: Command not found.\n", argv[0]);
                    exit(0);
                }
            }
            /* parent process */
            else
            {
                if (!bg)
                {
                    /* Block all signals when adding job to the list */
                    sigprocmask(SIG_BLOCK, &mask_all, NULL);
                    addjob(jobs, pid, FG, cmdline);	/* add foreground job */
                    /* Unblock SIGCHLD */
                    sigprocmask(SIG_SETMASK, &prev_mask, NULL);
                    waitfg(pid);
                }
                else
                {
                    /* Block all signals when adding job to the list */
                    sigprocmask(SIG_BLOCK, &mask_all, NULL);
                    addjob(jobs, pid, BG, cmdline);	/* add background job */
                    /* Unblock SIGCHLD */
                    sigprocmask(SIG_SETMASK, &prev_mask, NULL);
                    printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);
                }
            }

        }
        return;
    }

按照之前的逻辑， ``eval`` 函数的实现很直接。只是要注意一些同步问题的细节，比如如书中实例8.40所示，需在执行 ``fork`` 函数前屏蔽掉SIGCHLD信号，确保子进程在添加到任务列表后才可以被回收。

``builtin_cmd`` 函数
''''''''''''''''''''

``builtin_cmd`` 函数的实现也很直接，我们只需要对四个内置命令解析即可：

.. code-block:: c

    int builtin_cmd(char **argv) 
    {
        /* quit command */
        if (!strcmp(argv[0], "quit"))
            exit(0);

        /* jobs command */
        if (!strcmp(argv[0], "jobs"))
        {
            listjobs(jobs);
            return 1;
        }

        /* bg/fg commands */
        if (!strcmp(argv[0], "bg") || !strcmp(argv[0], "fg"))
        {
            do_bgfg(argv);
            return 1;
        }

        return 0;     /* not a builtin command */
    }


``do_bgfg`` 函数
'''''''''''''''''''''''

``bg`` 和 ``fg`` 命令有两种参数，一种是Job ID，另一种是进程ID，所以需针对这两种参数做相应的处理。
另一点需要注意的是，调用 ``kill`` 函数发送SIGCONT信号时，应将信号发送给整个进程组。
对于 ``fg`` 命令而言，需调用 ``waitfg`` 函数等待前台进程结束。
对于 ``bg`` 命令而言，则需打印出相应后台进程的相关信息。

.. code-block:: c

    void do_bgfg(char **argv) 
    {
        struct job_t *jb;
        pid_t pid;
        int jid;
        char *jb_str;

        /* bg/fg <job> */
        jb_str = argv[1];

        /* if <job> is NULL, prompt error msg and return */
        if (jb_str == NULL)
        {
            printf("%s command requires PID or %%jobid argument\n", argv[0]);
            return;
        }

        /* parse <job> argument */
        /* job is a jid number */
        if (sscanf(jb_str, "%%%d", &jid) > 0)
        {
            jb = getjobjid(jobs, jid);
            if (jb == NULL)
            {
                printf("%s: No such job\n", jb_str);
                return;
            }
        }
        /* job is a pid number */
        else if (sscanf(jb_str, "%d", &pid) > 0)
        {
            jb = getjobpid(jobs, pid);
            if (jb == NULL)
            {
                printf("%s: No such process\n", jb_str);
                return;
            }
        }
        /* job is an invalid argument */
        else
        {
            printf("%s: argument must be a PID or %%jobid\n", argv[0]);
            return;
        }

        /* Send SIGCONT signal */
        if (kill(-(jb->pid), SIGCONT))
            unix_error("kill error");

        /* bg or fg command? */
        if (!strcmp("fg", argv[0]))
        {
            jb->state = FG;
            waitfg(jb->pid);
        }
        else
        {
            printf("[%d] (%d) %s", jb->jid, jb->pid, jb->cmdline);
            jb->state = BG;
        }

        return;
    }


``waitfg`` 函数
'''''''''''''''''''''''

根据书中8.42的示例，我们可用 ``sigsuspend`` 函数来实现 ``waitfg`` 函数。
需要注意的时，在调用 ``sigsuspend`` 函数之前，SIGCHLD信号是要被屏蔽的。但 ``sigsuspend`` 会暂时解禁SIGCHLD信号，这样当子进程终结时，就会触发 ``sigsuspend`` 函数退出，从而退出while循环。

.. code-block:: c

    void waitfg(pid_t pid)
    {
        sigset_t mask;
        sigemptyset(&mask);
        while(fgpid(jobs) == pid)
        {
            sigsuspend(&mask);
        }
        return;
    }

信号处理函数
'''''''''''''''''''''''

对于SIGINT和SIGTSTP信号而言，其实现相对简单：

.. code-block:: c

    void sigint_handler(int sig) 
    {
        /* save the error code */
        int olderrno = errno;

        pid_t pid = fgpid(jobs);
        if (pid != 0)
            kill(-pid, sig);

        errno = olderrno;
        return;
    }

    void sigtstp_handler(int sig) 
    {
        /* save the error code */
        int olderrno = errno;

        pid_t pid = fgpid(jobs);
        if (pid != 0)
            kill(-pid, sig);

        errno = olderrno;
        return;
    }

对于SIGCHLD信号而言，情况就稍微复杂一些。我们既要考虑到子进程退出的可能情况，是正常exit退出，还是被 ``ctrl-c`` 或 ``ctrl-z`` 终止或暂停。
同时也要考虑到可能出现的竞争问题，所以要对 ``deletejob`` 进行相应的同步处理。

.. code-block:: c

    void sigchld_handler(int sig) 
    {
        /* 3 cases need to be considered here:
        * 1. the child terminates
        * 2. the child was terminated by ctrl-c
        * 3. the child was stopped by a ctrl-z
        */

        /* save error code */
        int olderrno = errno;

        int status;
        pid_t pid;
        struct job_t *jb;
        sigset_t mask, prev;
        sigfillset(&mask);

        /* WNOHANG | WUNTRACED option
        * return immediately, with a return value of zero
        * if none of the children in the wait set has stopped or terminated,
        * or with a return value equal to the PID of one of the stopped
        * or terminated children. */
        while ((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0)
        {
            /* Block all signals */
            sigprocmask(SIG_BLOCK, &mask, &prev);
            /* case 1 */
            if (WIFEXITED(status))
            {
                deletejob(jobs, pid);
            }
            /* case 2 */
            else if (WIFSIGNALED(status))
            {
                deletejob(jobs, pid);
                /* printf is not an async-safe function, so we shouldn't use them in real world */
                printf("Job [%d] (%d) terminated by signal %d\n", pid2jid(pid), pid, WTERMSIG(status));

            }
            /* case 3 */
            else if (WIFSTOPPED(status))
            {
                jb = getjobpid(jobs, pid);
                jb->state = ST;
                /* printf is not an async-safe function, so we shouldn't use them in real world */
                printf("Job [%d] (%d) stopped by signal %d\n", pid2jid(pid), pid, WSTOPSIG(status));
            }
            /* Unblock all signals */
            sigprocmask(SIG_SETMASK, &prev, NULL);
        }
        errno = olderrno;
        return;
    }


测试程序
'''''''''''''''''''''''

至此，本实验的所以待实现的函数都已完成。参考 `此链接 <https://www.cnblogs.com/liqiuhao/p/8120617.html>`_ ，执行一下 ``test.sh`` 来对我们实现的 ``tsh`` 和参考程序 ``tshref`` 的执行结果进行对比。

.. code-block:: shell

    #! /bin/bash

    mkdir test

    for file in $(ls trace*)
    do
        ./sdriver.pl -t $file -s ./tshref -a "-p" > ./test/tshref.$file
        ./sdriver.pl -t $file -s ./tsh -a "-p" > ./test/tsh.$file
    done

    for file in $(ls trace*)
    do
        diff ./test/tshref.$file ./test/tsh.$file > ./test/diff.$file
    done

    for file in $(ls ./test/diff*)
    do
        echo -e "------------\n"
        echo $file
        cat $file
        echo -e "------------\n"
    done

运行的结果可以发现，除了运行的进程的PID值不同意外，两个程序的运行结果一致，实验通过。


0x04. 总结和评价
----------------

这个实验是参考着网上的教程实现的。可以说进一步加深了我对信号以及信号处理中可能出现的同步问题的理解。
实验过程中曾一度卡在了 ``waitfg`` 的实现，尤其是对 ``sigsuspend`` 函数的理解一开始并不到位。
另外为了图格式化打印的方便 ``sigchld_handler`` 的实现还是用了 ``printf`` 这种非异步安全的函数，但真正的生产代码肯定是不被允许的。