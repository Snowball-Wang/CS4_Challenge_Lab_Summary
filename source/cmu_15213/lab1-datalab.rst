
Lab 1: Data Lab
===============

0x01. 实验介绍
--------------

``datalab``\ 实验的目的是让程序员从\ **位**\ （或者叫\ **比特**\ ）的视角理解如何表示整型数和浮点数。实验总共有13个谜题，每个谜题只能使用限定的操作符（如\ ``!``\ 、\ ``~``\ 等）来实现特定的高级功能。13个谜题分别对应13个函数，如下表所示。

.. list-table::
   :header-rows: 1

   * - 函数名
     - 描述
     - 评分
     - 最多可使用操作符数量
   * - \ ``bitXor(x, y)``\
     - 只用\ ``&``\ 和\ ``~``\ 实现x \ ``||``\ y
     - 1
     - 14
   * - \ ``tmin()``\
     - 实现最小的补码数
     - 1
     - 4
   * - \ ``isTmax(x)``\
     - 判断x是不是最大的补码数，若是，返回True
     - 1
     - 10
   * - \ ``allOddBits(x)``\
     - 判断x的奇位是不是都为1
     - 2
     - 12
   * - \ ``negate(x)``\
     - 不使用\ ``-``\ 号得到\ ``-x``\
     - 2
     - 5
   * - \ ``isAsciiDigit(x)``\
     - 判断x是否是字符0~9
     - 3
     - 15
   * - \ ``conditional``\
     - 实现\ ``x ? y : z``\
     - 3
     - 16
   * - \ ``isLessOrEqual(x, y)``\
     - 判断x是否小于等于y
     - 3
     - 24
   * - \ ``LogicalNeg(x)``\
     - 不使用\ ``!``\ 号得到\ ``!x``\
     - 4
     - 12
   * - \ ``howManyBits(x)``\
     - 用补码表示x需要的最小比特数
     - 4
     - 90
   * - \ ``floatScale2(uf)``\
     - 返回\ ``2*f``\ 的位表示
     - 4
     - 30
   * - \ ``floatFloat2Int(uf)``\
     - 返回\ ``(int)f``\ 的位表示
     - 4
     - 30
   * - \ ``floatPower2(uf)``\
     - 返回\ ``2.0^x``\ 的位表示
     - 4
     - 30


此外，对于整型数谜题，实验有如下要求：


* 只可以使用0~255(\ ``0xFF``\ )范围内的数字常量
* 只可使用局部变量和函数传入参数（不可以使用全局变量）
* 只可使用一元操作符\ ``!``\ ,\ ``~``\ 和二元操作符\ ``&``\ ,\ ``^``\ ,\ ``|``\ ,\ ``+``\ ,\ ``<<``\ ,\ ``>>``
* 不可使用\ ``if``\ 、\ ``while``\ 等控制语句
* 不可定义或使用任何宏或者除\ ``int``\ 类型外的其它数据类型（数组，联合体等）
* 不可定义或调用其它任何函数
* 不可任何形式的类型转换

对于浮点数谜题，实验的要求相比整型数谜题宽松一下，体现在：


* 可使用循环和条件判断语句
* 可使用\ ``int``\ 和\ ``unsigned``\ 两种数据类型，并且不限制数字常量的使用
* 可使用算数、逻辑和比较运算符

0x02. 实验环境搭建
------------------

我的Linux环境是桌面版的\ ``Ubuntu 20.04``\ 系统，实验直接在当前环境下编译运行。当然也可以按照网上的教程用Windows自带的WSL或者Linux环境下的Docker去做实验。

实验的源代码可通过以下命令下载解压：

.. code-block:: bash

   $ wget http://csapp.cs.cmu.edu/3e/datalab-handout.tar
   $ tar xvf datalab-handout.tar

进入实验目录，我们可以先编译实验并且运行一下评分脚本：

.. code-block:: bash

   $ cd datalab-handout && make && ./driver.pl
   make: Nothing to be done for 'all'.
   1. Running './dlc -z' to identify coding rules violations.

   2. Compiling and running './btest -g' to determine correctness score.
   gcc -O -Wall -m32 -lm -o btest bits.c btest.c decl.c tests.c

   3. Running './dlc -Z' to identify operator count violations.

   4. Compiling and running './btest -g -r 2' to determine performance score.
   gcc -O -Wall -m32 -lm -o btest bits.c btest.c decl.c tests.c


   5. Running './dlc -e' to get operator count of each function.

   Correctness Results     Perf Results
   Points  Rating  Errors  Points  Ops     Puzzle
   0       1       1       0       0       bitXor
   0       1       1       0       0       tmin
   0       1       1       0       0       isTmax
   0       2       1       0       0       allOddBits
   0       2       1       0       0       negate
   0       3       1       0       0       isAsciiDigit
   0       3       1       0       0       conditional
   0       3       1       0       0       isLessOrEqual
   0       4       1       0       0       logicalNeg
   0       4       1       0       0       howManyBits
   0       4       1       0       0       floatScale2
   0       4       1       0       0       floatFloat2Int
   0       4       1       0       0       floatPower2

   Score = 0/62 [0/36 Corr + 0/26 Perf] (0 total operators)

可以看到输出显示此时得分为0，表明实验环境已准备就绪，可以开始解题得分。

0x03. 实验代码实现及思路说明
----------------------------

``bitXor(x, y)``
^^^^^^^^^^^^^^^^^^^^

**代码实现**\ ：

.. code-block:: c

   /*
    * bitXor - x^y using only ~ and &
    *   Example: bitXor(4, 5) = 1
    *   Legal ops: ~ &
    *   Max ops: 14
    *   Rating: 1
    */
   int bitXor(int x, int y) {
     return (~(x & y)) & (~(~x & ~y));
   }

**思路说明**\ ：

从简入手，设想x和y都只有1bit，对应的\ ``xor``\ 操作的值是：

.. list-table::
   :header-rows: 1

   * - x
     - y
     - x ^ y
   * - 0
     - 0
     - 0
   * - 0
     - 1
     - 1
   * - 1
     - 0
     - 1
   * - 1
     - 1
     - 0


当x与y相同时，\ ``x & y``\ 的值与\ ``~x & ~y``\ 的值一定是相反的，一个0一个1。而当x与y不相同时，\ ``x & y``\ 和\ ``~x & ~y``\ 的值都为0。而为了构建上表的亦或关系，我们可以对\ ``x & y``\ 与\ ``~x & ~y``\ 的值再进行一次\ ``~``\ 操作，这样对于x和y相同的情况，上述操作得到的结果与之前一直，还是一个0一个1。而x与y不同的时候，则上述操作两个表达式都为1。最后我们给出的答案是\ ``(~(x & y)) & (~(~x & ~y))``\ 。

``tmin()``
^^^^^^^^^^^^^^

**代码实现**\ ：

.. code-block:: c

   /*
    * tmin - return minimum two's complement integer
    *   Legal ops: ! ~ & ^ | + << >>
    *   Max ops: 4
    *   Rating: 1
    */
   int tmin(void) {
     return 1 << 31;

   }

**思路说明**\ ：

最小的补码数也就是\ ``-2^31``\ ，对应可通过1左移31位获得。

``isTmax(x)``
^^^^^^^^^^^^^^^^^

**代码实现**\ ：

.. code-block:: c

   /*
    * isTmax - returns 1 if x is the maximum, two's complement number,
    *     and 0 otherwise
    *   Legal ops: ! ~ & ^ | +
    *   Max ops: 10
    *   Rating: 1
    */
   int isTmax(int x) {
     /* if x is Tmax, x + 1 will be Tmin, x ^ (x + 1) will be -1, negate -1 will be zero.
      * Meanwhile, -1 also has the same property, so it has to be excluded.
      */
     return !(~(x ^ (x + 1)) | !(x + 1));
   }

**思路说明**\ ：

如果x是\ ``Tmax``\ ，那么\ ``x+1``\ 就会使\ ``Tmin``\ ，对应\ ``x ^ (x + 1)``\ 就是-1，求非则就是0。但是要排除x是-1的这种特殊情况，即\ ``!(x + 1)``\ 在x不是-1的情况下都为0。

``allOddBits(x)``
^^^^^^^^^^^^^^^^^^^^^

**代码实现**\ ：

.. code-block:: c

   /*
    * allOddBits - return 1 if all odd-numbered bits in word set to 1
    *   where bits are numbered from 0 (least significant) to 31 (most significant)
    *   Examples allOddBits(0xFFFFFFFD) = 0, allOddBits(0xAAAAAAAA) = 1
    *   Legal ops: ! ~ & ^ | + << >>
    *   Max ops: 12
    *   Rating: 2
    */
   int allOddBits(int x) {
     /* construct 0xAAAAAAAA using << operator.
      * x & 0xAAAAAAAA ^ 0xAAAAAAAA will be zero if x's odd-numbered bits are all 1.
      */
     int oddNum = 0xAA | (0xAA << 8) | (0xAA << 16) | (0xAA << 24);
     int result = !((x & oddNum) ^ oddNum);
     return result;
   }

**思路说明**\ ：

因为实验限制只能使用0~255的整数常量，所以我们必须首先构建奇数比特位全为1的32位常量值。通过对\ ``0xAA``\ 的8位增量左移操作构建出\ ``0xAAAAAAA``\ ，然后将此值与x相与，得到的值再和32位奇数比特位1值进行亦或操作，即可判断x的奇数比特位是否全为1。若全为1，则上述过程的计算结果为0，反之为1。最后对结果取非即实现了我们想要的功能。

``negate(x)``
^^^^^^^^^^^^^^^^^

**代码实现**\ ：

.. code-block:: c

   /*
    * negate - return -x
    *   Example: negate(1) = -1.
    *   Legal ops: ! ~ & ^ | + << >>
    *   Max ops: 5
    *   Rating: 2
    */
   int negate(int x) {
     /* negate(x) = ~x + 1 */
     return (~x + 1);
   }

**思路说明**\ ：

这是根据补码的特性决定的。补码的取负数即为原比特表示的取反加1。

``isAsciiDigit(x)``
^^^^^^^^^^^^^^^^^^^^^^^

**代码实现**\ ：

.. code-block:: c

   /*
    * isAsciiDigit - return 1 if 0x30 <= x <= 0x39 (ASCII codes for characters '0' to '9')
    *   Example: isAsciiDigit(0x35) = 1.
    *            isAsciiDigit(0x3a) = 0.
    *            isAsciiDigit(0x05) = 0.
    *   Legal ops: ! ~ & ^ | + << >>
    *   Max ops: 15
    *   Rating: 3
    */
   int isAsciiDigit(int x) {
     /* cond1: the second byte should be 0x3.
      * cond2: the first byte + 6 should be less than 0x10.
      * x will be [0x30, 0x39] if both condition satisfy.
      */
     int cond1 = !((x >> 4) ^ 0x3);
     int cond2 = !(((x & 0xF) + 0x6) & 0x10);
     int result = cond1 & cond2;
     return result;
   }

**思路说明**\ ：

x如果是ASCII码数字的话，对应的值是0x30~0x39区间内。所以\ ``cond1``\ 先判断x的第2个半字节是不是0x3。然后\ ``cond2``\ 判断x的第1个半字节是不是在0x0~0x9的范围内，这个我们可以通过对这个半字加上6来判断是否有进位实现。当两个条件都成立时，对应的x是ASCII码数字。

``conditional(x, y, z)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**代码实现**\ ：

.. code-block:: c

   /*
    * conditional - same as x ? y : z
    *   Example: conditional(2,4,5) = 4
    *   Legal ops: ! ~ & ^ | + << >>
    *   Max ops: 16
    *   Rating: 3
    */
   int conditional(int x, int y, int z) {
     /* if x is zero, cond will be zero. Otherwise cond will extend to 0xFFFFFFFF.
      * result will be y or z depending on the value of cond.
      */
     int cond = !!(x);
     cond = ~cond + 1;
     return (y & cond) | (z & ~cond);
   }

**思路说明**\ ：

实现三元符号\ ``? :``\ ，第一直觉是构建0和0xFFFFFFFF，然后根据x的值来对y和z与0和0xFFFFFFFF进行组合来实现功能。所以\ ``cond``\ 首先判断x是否为0。若为1，则应返回y，对应y应和0xFFFFFFFF相与，z与0相与置为0，两个结果或后得到y。反之亦然。

``isLessOrEqual(x, y)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^

**代码实现**\ ：

.. code-block:: c

   /*
    * isLessOrEqual - if x <= y  then return 1, else return 0
    *   Example: isLessOrEqual(4,5) = 1.
    *   Legal ops: ! ~ & ^ | + << >>
    *   Max ops: 24
    *   Rating: 3
    */
   int isLessOrEqual(int x, int y) {
     /* x >= 0 and y >= 0, or x < 0 and y < 0, cond1 satisfy.
      * x >= 0 and y < 0, cond4 satisfy and should be excluded.
      * x < 0 and y >= 0, cond2 satisfy.
      * x == y, cond3 satisfy.
      */
     int x_msb = x >> 31;
     int y_msb = y >> 31;
     /* cond1: MSB for negate(x) + y should be 1 when x,y > 0 or x, y < 0
      * Meanwhile, y cannot be 0.
      */
     int cond1 = !((~x + 1 + y) >> 31) & !!(y ^ 0);
     /* cond2: x < 0 and y > 0 */
     int cond2 = (x_msb & (!y_msb));
     /* cond3: x == y */
     int cond3 = !(x ^ y);
     /* cond4: x > 0 and y < 0 should be excluded */
     int cond4 = !((!x_msb) & y_msb);

     int result = (cond1 | cond2 | cond3 ) & cond4;
     return result;
   }

**思路说明**\ ：

首先要明确，不能通过简单的\ ``x - y``\ 的值来判断，因为结果存在溢出。那就分情况处理。\ ``cond1``\ 是x和y同符号的情况下，可以通过\ ``negate(x) + y``\ 的值来判断大小。同时这里还要排除y为0的情况。\ ``cond2``\ 是x为负数，y为正数的情况。\ ``cond3``\ 是x和y相等的情况。以上三组条件满足的情况下还要排除x是正数，y是负数的情况，因为这种情况下\ ``cond1``\ 也满足，所以通过\ ``cond4``\ 可以把上述情况排除掉。最后四个条件组合即可。

``logicalNeg(x)``
^^^^^^^^^^^^^^^^^^^^^

**代码实现**\ ：

.. code-block:: c

   /*
    * logicalNeg - implement the ! operator, using all of
    *              the legal operators except !
    *   Examples: logicalNeg(3) = 0, logicalNeg(0) = 1
    *   Legal ops: ~ & ^ | + << >>
    *   Max ops: 12
    *   Rating: 4
    */
   int logicalNeg(int x) {
     /* MSB should be different for x and negate(x) except 0x0 and 0x80000000. */
     int x_msb = x >> 31;
     int neg_x_msb = (~x + 1) >> 31;
     /* flag will be 0xFFFFFFFF if x != (0x0 or 0x80000000),
      * and will be zero if x == (0x0 or 0x80000000).
      */
     int flag = ~(x_msb ^ neg_x_msb);
     /* exclude 0x80000000 */
     flag = flag & (~x_msb);
     return flag & 0x1;
   }

**思路说明**\ ：

本题要实现的是逻辑非，即x不为0时，运算后值为0；x为0时，运算后值为1。对于除了0和\ ``Tmin``\ (0x80000000)的其它补码数，x和negate(x)的最高有效位必然是不同的。两者抑或后取非，对于x不为0或\ ``Tmin``\ ，得到的是0xFFFFFFFF；x为0或\ ``Tmin``\ ，则得到的是0。最后排除当x为\ ``Tmin``\ ，返回值为1的情况。

``howManyBits(x)``
^^^^^^^^^^^^^^^^^^^^^^

**代码实现：**

.. code-block:: c

   /* howManyBits - return the minimum number of bits required to represent x in
    *             two's complement
    *  Examples: howManyBits(12) = 5
    *            howManyBits(298) = 10
    *            howManyBits(-5) = 4
    *            howManyBits(0)  = 1
    *            howManyBits(-1) = 1
    *            howManyBits(0x80000000) = 32
    *  Legal ops: ! ~ & ^ | + << >>
    *  Max ops: 90
    *  Rating: 4
    */
   int howManyBits(int x) {
     /* declare corresponding bit variables. */
     int bit16, bit8, bit4, bit2, bit1, bit0, bit_num;

     /* Step 1: if x > 0, keep x; x < 0, bit invert x; */
     int x_msb = x >> 31;
     x = (x & ~x_msb) | (~x & x_msb);

     /* Step 2: compute the bit number in binary way. */
     bit16 = !!(x >> 16) << 4;
     x = x >> bit16;

     bit8 = !!(x >> 8) << 3;
     x = x >> bit8;

     bit4 = !!(x >> 4) << 2;
     x = x >> bit4;

     bit2 = !!(x >> 2) << 1;
     x = x >> bit2;

     bit1 = !!(x >> 1);
     x = x >> bit1;

     bit0 = !!(x);

     bit_num = bit16 + bit8 + bit4 + bit2 + bit1 + bit0 + 1;

     return bit_num;
   }

**思路说明**\ ：

这道题首先要理解题目的意思。判断一个数用补码表示所需要的最少位数，应判断这个数落在对应N位补码表示的数的区间，这个区间范围是\ ``-2^(N-1) ~ 2^(N-1) -1``\ 。结合注释中给出的例子，以12和5为例，12落在区间\ ``-16 ~ 15``\ ，用最少的补码表示应该是\ ``01100``\ ，所以至少需要5位比特数来表示补码。同理-5落在区间\ ```-8 ~ 7``\ ，用最少的补码表示应该是\ ``1011``\ ，所以至少需要4位比特数来表示补码。

理解了题目的意思，我们就考虑如何实现此功能。直觉告诉我们不断对x右移，并计算不断右移为1的数量总和再加上1，即是我们需要表示补码的最少比特数。对于负数而言，我们可以对其取反，取反后得到的正数也和负数落在同一个区间内。但是总共32位比特我们不可能一个一个右移32次，这样子大概率会超过90这个允许使用的操作符上限。那我们可针对32位比特采用二分法的方法。若右移N位得到的值不为0，则表明高N位有1，保留高N位比特，并且把高N位比特数计入总和中；若右移N位得到的值位0，则表明高N位全为0，保留低N位比特，比特数不计入总和。上述循环继续执行直到32位比特数遍历完。

``floatScale2(uf)``
^^^^^^^^^^^^^^^^^^^^^^^

**代码实现：**

.. code-block:: c

   //float
   /*
    * floatScale2 - Return bit-level equivalent of expression 2*f for
    *   floating point argument f.
    *   Both the argument and result are passed as unsigned int's, but
    *   they are to be interpreted as the bit-level representation of
    *   single-precision floating point values.
    *   When argument is NaN, return argument
    *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
    *   Max ops: 30
    *   Rating: 4
    */
   unsigned floatScale2(unsigned uf) {
     /* declare variables */
     unsigned sign_bit, exp, frac, result;
     sign_bit = uf & (1 << 31);
     exp = uf >> 23 & 0xFF;
     frac = uf & 0x7FFFFF;

     /* if exp is all ones, uf is infinity or NaN. */
     if(!(exp ^ 0xFF))
       return uf;

     /* if frac and exp are all zeros, +0/-0 means result is same as uf. */
     if(!frac && !exp)
       return uf;
     /* if uf is a normalized value, meaning exp is not zero,
      * simply add 1 to exp field.
      * if uf is a denormalized value, meaning exp is zero,
      * left shift by 1 to frac and add the value to exp field.
      */
     if(exp)
       exp += 1;
     else
       frac = frac << 1;
     result = sign_bit | ((exp << 23) + frac);

     return result;
   }

**思路说明：**

首先把\ ``uf``\ 表示的浮点数的三个部分提取出来，\ ``sign_bit``\ 对应符号位，\ ``exp``\ 对应阶数域，\ ``frac``\ 对应尾数域。\ ``exp``\ 全为1则表示浮点数是无穷大或者\ ``NaN``\ ，则直接返回原始值。如果\ ``exp``\ 和\ ``frac``\ 都为0，则浮点数是\ ``+0``\ 或\ ``-0``\ ，也直接返回原始值。对于正规化的数（normalized value），\ ``2*f``\ 意味着\ ``exp``\ 值加1。对于非正规化的数（denormalized value），\ ``2*f``\ 意味着\ ``frac``\ 左移一位。最后将三部分重新组合在一起。考虑到\ ``frac``\ 左移一位可能会进位，所以对应\ ``exp``\ 和\ ``frac``\ 是相加关系。

``floatFloat2Int(uf)``
^^^^^^^^^^^^^^^^^^^^^^^^^^

**代码实现**\ ：

.. code-block:: c

   /*
    * floatFloat2Int - Return bit-level equivalent of expression (int) f
    *   for floating point argument f.
    *   Argument is passed as unsigned int, but
    *   it is to be interpreted as the bit-level representation of a
    *   single-precision floating point value.
    *   Anything out of range (including NaN and infinity) should return
    *   0x80000000u.
    *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
    *   Max ops: 30
    *   Rating: 4
    */
   int floatFloat2Int(unsigned uf) {
     /* declare variables */
     unsigned sign_bit, exp, mantissa;
     int result, e;
     /* get sign bit, exp and frac field of FP */
     sign_bit = uf >> 31 & 0x1;
     exp = uf >> 23 & 0xFF;
     mantissa = (uf & 0x7FFFFF) | 0x800000;
     e = exp - 0x7F;

     /* if uf is NaN or infinity or out of range for int */
     if(e > 31)
       return 0x80000000u;

     /* if exp < 128, uf as float is 0.xxx, just return 0 */
     if(e < 0)
       return 0;

     /* shift according to e */
     if(e > 23)
       result = mantissa << (e - 23);
     else
       result = mantissa >> (23 - e);
     /* convert negative to postive */
     if(sign_bit)
       result = ~result + 1;
     return result;
   }

**思路说明**\ ：

和上一题一样，提取出\ ``uf``\ 的三个部分，不过对应的\ ``frac``\ 被\ ``mantissa``\ 取代。\ ``mantissa``\ 即是以2为底的计数法的尾数。举个例子，浮点数15213.0的二进制表示是\ ``11101101101101``\ ，对应的以2为底奇数的写法是\ ``1.1101101101101 x 2^13``\ ，对应的\ ``mantissa``\ 即是\ ``1.1101101101101``\ 。

因为\ ``int``\ 类型的取值范围是\ ``-2^31 ~ 2^31 -1``\ ，所以\ ``float``\ 类型转换成\ ``int``\ 类型时，需根据\ ``exp``\ 域做不同的变换，也就是根据阶数\ ``e``\ 来构建浮点数转换的整型数：


* ``e > 31``\ 时，浮点数表示的值超过了整型数的范围，返回0x80000000
* ``e < 0``\ 时，浮点数为\ ``0.xxxx``\ ，返回0
* ``23 < e < 31``\ 时，此时表明阶数较大，则\ ``mantissa``\ 相应地左移\ ``23 - e``\ 位
* ``0 < e <= 23``\ 是，此时表明阶数较小，则\ ``mantissa``\ 相应地右移\ ``e - 23``\ 位，去除\ ``frac``\ 补充的0

最后，如果结果为负数，对值进行\ ``negate``\ 操作。

``floatPower2(x)``
^^^^^^^^^^^^^^^^^^^^^^

**代码实现**\ ：

.. code-block:: c

   /*
    * floatPower2 - Return bit-level equivalent of the expression 2.0^x
    *   (2.0 raised to the power x) for any 32-bit integer x.
    *
    *   The unsigned value that is returned should have the identical bit
    *   representation as the single-precision floating-point number 2.0^x.
    *   If the result is too small to be represented as a denorm, return
    *   0. If too large, return +INF.
    *
    *   Legal ops: Any integer/unsigned operations incl. ||, &&. Also if, while
    *   Max ops: 30
    *   Rating: 4
    */
   unsigned floatPower2(int x) {
     /* if x > 127, x too large, return +INF */
     if(x > 127)
       return 0x7F800000u;
     /* if -126 <= x <= 127, just add x by Bias and left shift 23 bits */
     else if(x <= 127 && x >= -126)
       return (x + 127) << 23;
     /* if -149 <= x <= -127 */
     else if(x <= -127 && x >= -149)
       return 1 << (x + 149);
     else
       return 0;
   }

**思路说明**\ ：

此题求解\ ``2.0^x``\ 的浮点数二进制表示。根据x值的不同，做以下操作：


* ``x > 127``\ ，超出了浮点数可表示的最大值，返回\ ``+INF``
* ``-126 <= x <= 127``\ ，此时x作为阶数在浮点数正规化数的表示范围，则返回\ ``(x + 127) << 23``
* ``-149 <= x <= -127``\ ，此时x作为阶数，属于\ ``exp``\ 全为0情况下极小的小数的表示，这个表示最多只能有\ ``frac``\ 域的23位，对应返回\ ``1 <<(x + 149)``
* ``x < -149``\ ，超出了浮点数克表示的最小值，返回0

0x04. 总结和评价
----------------

**实验成绩**\ ：

.. code-block:: bash

   $ ./driver.pl
   1. Running './dlc -z' to identify coding rules violations.

   2. Compiling and running './btest -g' to determine correctness score.
   gcc -O -Wall -m32 -lm -o btest bits.c btest.c decl.c tests.c

   3. Running './dlc -Z' to identify operator count violations.

   4. Compiling and running './btest -g -r 2' to determine performance score.
   gcc -O -Wall -m32 -lm -o btest bits.c btest.c decl.c tests.c


   5. Running './dlc -e' to get operator count of each function.

   Correctness Results     Perf Results
   Points  Rating  Errors  Points  Ops     Puzzle
   1       1       0       2       7       bitXor
   1       1       0       2       1       tmin
   1       1       0       2       7       isTmax
   2       2       0       2       9       allOddBits
   2       2       0       2       2       negate
   3       3       0       2       8       isAsciiDigit
   3       3       0       2       8       conditional
   3       3       0       2       21      isLessOrEqual
   4       4       0       2       9       logicalNeg
   4       4       0       2       38      howManyBits
   4       4       0       2       15      floatScale2
   4       4       0       2       16      floatFloat2Int
   4       4       0       2       14      floatPower2

   Score = 62/62 [36/36 Corr + 26/26 Perf] (155 total operators)

``datalab``\ 实验主要考察对补码，有符号数无符号数和IEEE 754浮点数的理解。整个实验中除\ ``howManyBits``\ 函数外，其它谜题基本都是自己推导出来，自我感觉要比第一次做实验的效果好很多。
