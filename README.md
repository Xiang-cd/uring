# 基于用户态中断和io_uring的异步IO用户程序

## 项目简介

本项目是`uintr+io_uring`异步IO的用户程序仓库, 程序的运行需要在我们修改过的[内核](https://github.com/OS-F-4/uintr-linux-kernel/tree/uring)_上, 并且操作系统运行在支持用户态中断的硬件环境中。

文件树:

```
.
├── apic
├── bench
├── go-cat
├── io_uring-by-example
└── liburing
```

其中`apic`目录下是一个简单的测试程序, 仅仅测试我们能否收到io_uring完成的用户态中断通知, 程序输出`in handler`代表收到中断。

`bench`是使用不同方法测试完成同样的计算和IO任务所需用的总用时以及IO的延时, 我们的结果表明基于用户态中断和io_uring的方法即能实现计算和io的重叠, 使得完成任务的总用时更小, 同时由于中断的通知机制, IO的延迟方面也比单纯用io_uring要更出色。

`go-cat`是采用Cgo编程的样例程序, 为了使得中断处理函数更快的退出, 从而不会因为中断不能嵌套从而导致部分中断的延迟较高, 我们希望在中断函数中调用线程或者协程去处理IO, 启用线程的开销比协程更大, 我们期望使用协程, 但是在C语言中还没有对协程的支持, 所以我们采用Cgo编程, 将go语言的特性嵌入C语言中, 从而达到更加高效的中断处理。

`io_uring-by-example`包含了几个io_uring的简单用户程序, 能够帮助新手更快的了解io_uring的使用以及liburing的使用。在其中我们主要修改了`03_cat_liburing`, 使得改用户程序能够支持用户态中断来处理IO的完成, 这个版本的实现是线程版本。

`liburing`是经过我们适配的liburing库, 在编译和链接以上的用户程序时, 需要链接次文件夹中对应的头文件。

## 使用方法

```shell
cd liburing
make  # 先进入liburing 目录, 编译后其他仓库的代码才能成功编译和链接
cd xxx # 进入其他目录进行编译
make
```

每个目录下均有`Makefile`, 一般情况下直接make即可, 由于不同系统版本用户态中断的编号可能会改变, 使用时需要查看内核对应的系统调佣编号是否正确, 可以通过定义宏来改变, 或者手动修改源代码。 

注意在链接阶段必须链接本仓库内的liburing, 必要时修改Makefile中liburing的路径。

## bench 结果

我们在执行200个IO任务和1000个计算任务的情况下获得了bench的测试结果, 罗列如下:

其中normal是使用普通同步IO方式来进行IO的读写, uring是单独使用io_uring的情况, uintr是使用io_uring以及用户态中断的情况。本测试中, 单纯的计算用时3410806us。一下是不同大小的IO的延时和总的任务完成时间的情况。

| 方法   | 总用时(us) | 4K 延时 | 4M 延时 | 40M 延时 |
| ------ | ---------- | ------- | ------- | -------- |
| normal | 4163422    | 259     | 38753   | 698487   |
| uring  | 3546226    | 205714  | 269079  | 1283943  |
| uintr  | 3591993    | 360     | 41431   | 727977   |

更多细致的分析见我们的的[技术分享报告](https://github.com/OS-F-4/usr-intr/blob/main/%E6%8A%80%E6%9C%AF%E4%BA%A4%E6%B5%81%E6%8A%A5%E5%91%8A.pptx)。以及[最终报告](https://github.com/OS-F-4/usr-intr/blob/main/%E6%9C%80%E7%BB%88%E6%8A%A5%E5%91%8A.md)。i
