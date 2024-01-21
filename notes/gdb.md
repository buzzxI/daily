# 写在前面

不得不承认 gdb 熟练了之后确实很好用, 在 vistual studio 中的各种 debug 操作基本上在 gdb 中都有等效的用法, 但因为其 command line 的设计, 上手起来并没有很顺心

我承认官方文档是最好的学习资料, 不过我日常使用的最多的还是 chatgpt (chatgpt 并不是万能的, 有的时候还是需要查文档), 这里仅仅起到 chatgpt cache 的作用

**使用 gdb 调试程序时，要将编译的程序添加标志位 -g**

# breaks

打断点是对于 debugger 最基础的要求了:

*   文件断点: `(gdb) break filename.c:42`, 这个平时自己写 demo 的时候用的多, 因为原文件已知, 行号已知
*   地址断点: `(gdb) break *0x12345678`, 汇编和 c 同时出现的时候给某个地址打断点可能更方便
*   函数断点: `(gdb) break foo`, 比如有一个函数名为 `foo`
*   条件断点: `(gdb) condition 1 x == 10`, 表示在编号为 1 的断点处, 只有满足条件 `x = 10` 时才停止
*   删除断点: `(gdb) delete 1`, 表示删除编号为 1 的断点
*   打印断点信息: `(gdb) info break`, 可以打印断点的编号

# tui

gdb 并不全是 cmd, 都 2023 年了, 求求你用一下 gdb 的 text UI 吧, 可以高亮当前执行源码(汇编代码)

*   cmd only: `(gdb) layout cmd`
*   cmd and source code: `(gdb) layout src`
*   cmd and assembly code: `(gdb) layout asm`
*   cmd, assembly code, source code: `(gdb) layout split`
*   switch: <kbd>ctrl + o</kbd>, 切换当前选中的 layout, 在 source code(assembly code) layout 中方向键可以看到其他部分的代码, 而在 command line layout 中方向键可以看到之前输入的命令
*   size: 有的时候需要调整窗口大小:
    *   height: `(gdb) winheight [layout-name] +/-[count]`: 分别表示增加/减少某个窗口的高度
    *   width: `(gdb) winwidth [layout-name] +/-[count]`: 分别表示增加/减少某个窗口的宽度
*   info: `(gdb) info win`: 打印各种窗口信息

# thread

gdb 还调试多线程程序

*   info: `(gdb) info thread`: 打印各种线程信息, 包含了线程 id
*   跟踪指定线程: `(gdb) thread 1`, 表示仅跟踪线程 id 为 1 的线程





