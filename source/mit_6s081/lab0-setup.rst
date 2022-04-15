
Lab 0: Environment Setup
==========================================

1. 前言
-------

自己一直想自学操作系统的课程很久了，主要还是因为工作上还是要和操作系统的很多概念打交道。之前断断续续地看过CSAPP，我觉得现在是一个比较合适的契机开启MIT-6.S081课程的学习了。这门课的精华是基于\ ``xv6``\ 内核的Lab，所以想要针对每个Lab的完成过程进行记录。

2. 环境搭建
-----------

正好手上有一台闲置的大学时用的Dell Inspiron N4110，重新装了\ ``Ubuntu 20.04``\ 的系统，系统的信息如下。

.. code-block:: shell

   $ cat /etc/os-release 
   NAME="Ubuntu"
   VERSION="20.04.4 LTS (Focal Fossa)"
   ID=ubuntu
   ID_LIKE=debian
   PRETTY_NAME="Ubuntu 20.04.4 LTS"
   VERSION_ID="20.04"
   HOME_URL="https://www.ubuntu.com/"
   SUPPORT_URL="https://help.ubuntu.com/"
   BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
   PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
   VERSION_CODENAME=focal
   UBUNTU_CODENAME=focal
   $ uname -r
   5.13.0-35-generic

推荐实验环境选用\ ``ubuntu 20.04``\ 及以上版本，用\ ``18.04``\ 版本在后续安装相应\ ``gcc``\ 工具链和\ ``qemu``\ 工具可能会出现版本过低等问题。具体的环境搭建步骤可参照\ `官网 <https://pdos.csail.mit.edu/6.S081/2020/tools.html>`_\ ，也可参照以下我的步骤。

2.1 安装相关工具及工具链
^^^^^^^^^^^^^^^^^^^^^^^^

首先安装\ ``xv6``\ lab中用到的工具和相应的编译工具链。

.. code-block:: shell

   $ sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu gcc-riscv64-unknown-elf

检查工具和工具链是否安装正确。

.. code-block:: shell

   $ riscv64-linux-gnu-gcc --version
   riscv64-linux-gnu-gcc (Ubuntu 9.4.0-1ubuntu1~20.04) 9.4.0
   Copyright (C) 2019 Free Software Foundation, Inc.
   This is free software; see the source for copying conditions.  There is NO
   warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
   $ qemu-system-riscv64 --version
   QEMU emulator version 4.2.1 (Debian 1:4.2-3ubuntu6.21)
   Copyright (c) 2003-2019 Fabrice Bellard and the QEMU Project developers
   $ riscv64-unknown-elf-gcc --version
   riscv64-unknown-elf-gcc () 9.3.0
   Copyright (C) 2019 Free Software Foundation, Inc.
   This is free software; see the source for copying conditions.  There is NO
   warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

对应的\ `官网 <https://pdos.csail.mit.edu/6.S081/2020/tools.html>`_\ 最后检查的\ ``gcc``\ 版本是\ ``10.1.0``\ ，\ ``qemu``\ 版本是\ ``5.1.0``\ ，但是我的版本目前使用下来也没有问题，所以就用原始的安装版本。

2.2 编译\ ``xv6``\ lab
^^^^^^^^^^^^^^^^^^^^^^

下载\ ``xv6``\ lab的实验源码，切换到\ ``util``\ 分支后编译就可以开始第一个lab的实验了，步骤如下。

输入\ ``make qemu``\ 命令后，可以看到一串编译日志，最后的\ ``$``\ 表示\ ``qemu``\ 的命令行。

``xv6``\ 没有\ ``ps``\ 命令，但\ ``Ctrl-p``\ 按键会打印出每个进程的信息。退出\ ``qemu``\ 则是\ ``Ctrl-a x``\ 按键。

.. code-block:: shell

   $ git clone git://g.csail.mit.edu/xv6-labs-2020
   $ cd xv6-labs-2020 # change into lab directory
   $ git branch -a
   * master
     remotes/origin/HEAD -> origin/master
     remotes/origin/cow
     remotes/origin/fs
     remotes/origin/lazy
     remotes/origin/lock
     remotes/origin/master
     remotes/origin/mmap
     remotes/origin/net
     remotes/origin/pgtbl
     remotes/origin/riscv
     remotes/origin/syscall
     remotes/origin/thread
     remotes/origin/traps
     remotes/origin/util
   $ git checkout util
   Branch 'util' set up to track remote branch 'util' from 'origin'.
   Switched to a new branch 'util'
   $ make qemu
   ...
   xv6 kernel is booting

   hart 2 starting
   hart 1 starting
   init: starting sh
   $

3. 参考
-------


#. https://gwzlchn.github.io/202106/6-s081-lab0/
