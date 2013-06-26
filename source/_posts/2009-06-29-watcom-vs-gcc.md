---
layout: post
title: watcom vs gcc
date: 2009-06-29
comments: true 
---


Для удовлетворения любопытства, скачал openwatcom 1.8 (такой бывший когда то
известным компилятор c/c++), исходники под линукс. собрал, и попробовал
тестовую задачку, [вот эту вот](http://shootout.alioth.debian.org/u32q/benchmark.php?test=binarytrees&lang=gcc&box=1)

Резуьтаты:
<!-- more -->
GCC
{%blockquote%}
walrus@home:/home/walrus/sandbox$ gcc btr.c -O3 -o btr_gcc -lm
walrus@home:/home/walrus/sandbox$ time ./btr_gcc 20
stretch tree of depth 21  check: -1
2097152  trees of depth 4  check: -2097152
524288 trees of depth 6  check: -524288
131072  trees of depth 8  check: -131072
32768  trees of depth 10  check: -32768
8192  trees of depth 12  check: -8192
2048  trees of depth 14  check: -2048
512  trees of depth 16  check: -512
128  trees of depth 18  check: -128
32  trees of depth 20  check: -32
long lived tree of depth 20  check: -1

real 1m2.138s
user 1m1.852s
sys 0m0.120s
{%endblockquote%}

Watcom

{%blockquote%}
walrus@home:/home/walrus/sandbox$ PATH=$PATH:/opt/watcom/binl WATCOM=/opt/watcom owcc -I/opt/watcom/h  -O3 -obtr_wc  btr.c 
walrus@home:/home/walrus/sandbox$ time ./btr_wc 20
stretch tree of depth 21  check: -1
2097152  trees of depth 4  check: -2097152
524288 trees of depth 6  check: -524288
131072  trees of depth 8  check: -131072
32768  trees of depth 10  check: -32768
8192  trees of depth 12  check: -8192
2048  trees of depth 14  check: -2048
512  trees of depth 16  check: -512
128  trees of depth 18  check: -128
32  trees of depth 20  check: -32
long lived tree of depth 20  check: -1

real 0m47.934s
user 0m47.675s
sys 0m0.136s
{%endblockquote%}

62 секунды против 48 в пользу watcom-а. Ай да старик watcom!! Вот это
да!

**UPD**  Да, забыл - ubuntu 9.04 32 AMD, 4000+, gcc gcc версия 4.3.3 (Ubuntu 4.3.3-5ubuntu4), watcom :

> owcc -v
> Open Watcom C/C++32 Compiler Driver Program Version 1.8beta1 LA
> Portions Copyright (c) 1988-2002 Sybase, Inc. All Rights Reserved.
> Source code is available under the Sybase Open Watcom Public License.
> See http://www.openwatcom.org/ for details.


