---
title: "OSX 上的动态链接"
date: 2020-06-09T17:19:51+08:00
draft: true
---

在 OSX 系统中，可执行文件的链接器涉及两种，ld（静态链接） 和 dyld（动态链接）。

静态链接器 ld 混合所有的目标文件，解决内部符号对外部的引用，并重定位这些符号，最后编译出一个复杂的可运行的文件。
静态链接器 ld (和 ld64)负责将源代码中的符号引用转换为间接的符号查找，供 dyld 以后使用。

dyld 动态链接库的工作如下：
1. 内核为进程启动提供了一个非常简单的原始堆栈，动态链接的启动基于这个堆栈 
2. 递归、缓存式的加载执行文件所有依赖的动态库到内存空间，包括环境和可执行的运行路径
3. 通过立即绑定非惰性符号并为惰性绑定设置必要的表，将这些库链接到可执行文件
4. 为可执行文件运行静态初始化器
5. 设置main函数的可执行参数，并调用它
6. 在程序执行过程中，通过符号表，加载惰性绑定的调用；提供运行时动态加载服务（通过 dl()函数）；给gdb或其他的调试工具提供 Hook，来获取关键信息。 
7. 在main函数返回后，运行静态终止路由
8. 一些情况下，在main函数返回后，调用 libSystem's 的 _exit 函数

启动：
一个新进程启动首先执行dyld的代码，内核通过系统调用命令 LC_LOAD_DYLINKER 来调用 __dyld_start ，从而启动 dyld 。__dyld_start 大致做了如下事情：
```
noreturn __dyld_start(stack mach_header *exec_mh, stack int argc, stack char **argv, stack char **envp, stack char **apple, stack char **STRINGS)
    {
        stack push 0 // debugger end of frames marker
        stack align 16 // SSE align stack
        uint64_t slide = __dyld_start - __dyld_start_static;
        void *glue = NULL;
        void *entry = dyldbootstrap::start(exec_mh, argc, argv, slide, ___dso_handle, &glue);
        if (glue)
            push glue // pretend the return address is a glue routine in dyld
        else
            stack restore // undo stack stuff we did before
        goto *entry(argc, argv, envp, apple); // never returns
    }
```