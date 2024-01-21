选择的是南大[操作系统：设计与实现](https://jyywiki.cn/OS/2023/)

# 应用视角的操作系统

## minimal

实现一个最小的应用程序, 学习任何语言的第一个例子基本上都是 "Hello World", 这里试图创建一个最小的应用程序, 只是输出一个 "Hello World"

```c
// minimal.c
#include <stdio.h>

int main() {
    printf("Hello World\n");
    return 0;
}
```

通过 gcc 进行编译得到最简程序 "minimal"

```shell
$ gcc minimal.c -o minimal
```

但这并不是一个最小的程序, 使用 objdump 反编译得到汇编代码, 查看 main 函数, 发现输出语句其实调用了 libc 中的输出函数

在程序编译的时候, 默认使用了 -lc 作为链接标志, 表示在程序运行的时候动态链接了 libc 中的方法, 使得运行中的 minimal 可以调用 libc 中定义的标准输出函数

如果在 gcc 编译的时候添加 `--static` 标志, 重新生成一个 minimal, 将得到一个很大的可执行目标文件:

```shell
$ gcc -o minimal minimal.c --static
$ ls -l minimal
-rwxr-xr-x 1 buzz buzz 826K Jun  1 15:12 minimal
```

>   标志 `--static` 表示了 gcc 使用静态库进行链接, 默认的 libc 是很大的, 如果使用 `--static` 标志将把整个 libc 都链接到可执行目标文件中, 这样在程序运行的时候不再需要寻找 shared library 进行链接了

使用 `--static` 会让 gcc 在编译的时候链接静态库, 默认 gcc 没有打印中间过程的输出, 添加 `--verbose` 标志, 打印 gcc 编译的过程

>   很多程序都可以通过添加参数 `--verbose` 打印执行日志

通过 gcc 得到 a.out 最后避免不了链接这个过程, 上述标志 `--verbose` 仅仅开启的编译得到 .o 文件的操作过程, 通过添加参数 `-Wl` 可以打印 linker 在 linking 阶段的操作过程(开启了这个选项后会产生大量的打印)

```shell
$ gcc -o minimal minimal.c --static -Wl,--verbose
```

>   标志 `-Wl` 用来向 linker 传递链接参数, 要注意的是此时参数之间通过 `,` 进行分割而不是空格, 并且 `-Wl` 和参数本身之间也需要一个 `,`

链接了大量 libc 库函数的 minimal 肯定不是最小的, 这里将 minimal.c 修改一下, 去掉 #include, 使用 gcc 仅编译得到 .o 文件, 并通过手动添加 library 的方式进行链接, 得到最小的 minimal

```shell
$ gcc -c minimal.c -o minimal.o
$ objdump -d minimal.o

minimal.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   48 8d 3d 00 00 00 00    lea    0x0(%rip),%rdi        # b <main+0xb>
   b:   e8 00 00 00 00          callq  10 <main+0x10>
  10:   b8 00 00 00 00          mov    $0x0,%eax
  15:   5d                      pop    %rbp
  16:   c3                      retq   
```

这种方式编译得到的 .o 文件反编译得到的汇编指令很简洁, 没有引入任何其他的函数, 不过现在通过 `ld` 强行链接 .o 文件, 将报错

```shell
$ ld minimal.o -o minimal
ld: warning: cannot find entry symbol _start; defaulting to 00000000004000b0
minimal.o: In function `main':
minimal.c:(.text+0xc): undefined reference to `puts'
```

>   linker 提示找不到指向 `puts` 的引用

去掉 main 中的 printf 可以得到一个 a.out, 但执行的时候会报错: segmentation fault; 而如果 main 中只包含了一个 while 死循环, 这个程序又可以正常执行

执行上述两种程序唯一的区别在于, 第一个程序不进行任何操作直接返回, 第二个程序进入死循环而不会返回, 也许 segmentation fault 和**程序的返回有关**

使用 gdb 调试, 并让程序单步执行汇编代码

>   在 gdb 中通过 `starti` 让程序从 main 开始执行, 通过 `disas` 查看当前执行的汇编代码的上下文, 通过 `si(stepi)` 单步执行汇编指令

```shell 
Dump of assembler code for function main:
   0x00000000004000b0 <+0>:     push   %rbp
   0x00000000004000b1 <+1>:     mov    %rsp,%rbp
   0x00000000004000b4 <+4>:     mov    $0x0,%eax
   0x00000000004000b9 <+9>:     pop    %rbp
=> 0x00000000004000ba <+10>:    retq   
End of assembler dump.
```

可以看到在执行返回命令之前, 程序一直正常运行

>   gdb 最神奇的一点是, 它可以 reverse debugging, 反向执行, 不过并不是所有的 executable file 都支持这个功能

retq 会将栈顶的值弹出并赋给 %rip(pc), 而此时 %rsp 指向的地址取值为 1

```shell
(gdb) x $rsp
0x7fffffffdd70: 0x00000001
```

即在 main 函数结束后, 程序将跳转到地址为 0x1 的地方执行, 在 linux X86/64 的环境下, 代码段的起始地址为 0x400000, 所以地址 0x1 显然是非法的

为了让程序可以平稳的停下来(而不是 segmentataion fault), 需要借助 `syscall`

```assembly
// minimal.S

#include <sys/syscall.h>

.globl _start
_start:
  movq $SYS_write, %rax   // write(
  movq $1,         %rdi   //   fd=1,
  movq $st,        %rsi   //   buf=st,
  movq $(ed - st), %rdx   //   count=ed-st
  syscall                 // );

  movq $SYS_exit,  %rax   // exit(
  movq $1,         %rdi   //   status=1
  syscall                 // );

st:
  .ascii "\033[01;31mHello, OS World\033[0m\n"
ed:
```

上述汇编代码其实就是调用了两个 syscall, 第一个 syscall 为 write, 第二个为 exit; 通过 mov 进行为 %rax 赋值, 为参数寄存器 %rdi, %rsi, %rdx 赋值, 然后通过汇编指令 syscall 进行系统调用

此时的汇编代码还不能需要通过 cpp(c preprocessor) 处理, 将各种宏替代, 比如 \$SYS_write, \$SYS_exit, #include, 在经过 cpp 预编译后会被展开替换

```shell
# 先通过 cpp 预处理, 再通过 cc 进行编译 得到汇编代码
$ cc -S minimal.S > minimal.s
```

编译后得到的汇编文件如下:

```assembly
// minimal.s

# 1 "minimal.S"
# 1 "<built-in>"
# 1 "<command-line>"
# 31 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 32 "<command-line>" 2
# 1 "minimal.S"
# 1 "/usr/include/x86_64-linux-gnu/sys/syscall.h" 1 3 4
# 24 "/usr/include/x86_64-linux-gnu/sys/syscall.h" 3 4
# 1 "/usr/include/x86_64-linux-gnu/asm/unistd.h" 1 3 4
# 13 "/usr/include/x86_64-linux-gnu/asm/unistd.h" 3 4
# 1 "/usr/include/x86_64-linux-gnu/asm/unistd_64.h" 1 3 4
# 14 "/usr/include/x86_64-linux-gnu/asm/unistd.h" 2 3 4
# 25 "/usr/include/x86_64-linux-gnu/sys/syscall.h" 2 3 4






# 1 "/usr/include/x86_64-linux-gnu/bits/syscall.h" 1 3 4
# 32 "/usr/include/x86_64-linux-gnu/sys/syscall.h" 2 3 4
# 2 "minimal.S" 2

.globl _start
_start:
  movq $1, %rax
  movq $1, %rdi
  movq $st, %rsi
  movq $(ed - st), %rdx
  syscall

  movq $60, %rax
  movq $1, %rdi
  syscall

st:
  .ascii "\033[01;31mHello, OS World\033[0m\n"
ed:

```

所以具体执行哪个 syscall, 其实取决于在调用 syscall 之前 %rax 的取值, 比如 1 就是 write, 60 就是 exit 

```shell
# assembler 汇编得到 minimal.o
$ as -c miminal.s
# linker 链接得到可执行目标文件
$ ld mimimal.o -o minimal
```

这样得到的可执行目标文件可以正常打印 Hello World 并正常结束

## hanio

对操作系统上程序的一个很重要的理解是程序是计算和系统调用组成的状态机, 操作系统当前的状态由当前时刻寄存器和内存中的值决定, 当操作系统取指执行时便发生了状态转移

对于高级语言而言, 如果能够编写一个解释器, 解析高级语言的每个指令, 并将其翻译为对应的机器指令, 那么也就相当于写好了编译器

这里 jyy_os 给出了一个例子, 即写出非递归的形式汉诺塔

>   汉诺塔可以通过下图描述:
>
>   ![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/06/01/21:54:11:hanio_tower.png)
>
>   上图中需要将 A 中所有的圆盘移动到 C 上, 在小圆盘上不能放大圆盘，在三根柱子之间一次只能移动一个圆盘
>
>   首先考虑递归实现, 对于初始状态而言, 将所有圆盘移动到 C 其实一共分为 "3 步":
>
>   *   "第一步": 将 A 最上方 n - 1 个圆盘移动到 B
>   *   "第二步": 将 A 上最后一个最大的圆盘移动到 C
>   *   "第三步": 将 B 上 n - 1 个圆盘移动到 C
>
>   考虑函数签名为: int hanio(int n, char from, char to, char via); 表示需要将 n 个圆盘从 from 移动到 to, 其中可以借助 via, 函数实现如下:
>
>   ```c
>   int hanio(int n, int from, int to, int via) {
>       int rst = 1;
>       if (n == 1) {
>           fprintf(stdout, "from: %d -> to: %d\n", from, to);
>           return rst;
>       }
>       rst += hanio(n - 1, from, via, to);
>       rst += hanio(1, from, to, via);
>       rst += hanio(n - 1, via, to, from);
>       return rst;
>   }
>   ```

这里进行非递归形式的修改并不是说修改了解题思路, 而是说尝试使用代码模拟栈的行为

当发生递归调用时模拟栈帧入栈操作, 栈帧中初始化 PC 为 0, 并将递归调用的参数传入栈帧即可完成入栈操作

同时要明确的是, 递归调用的函数返回会回到当前函数下一条指令, 且每次操作的一定是栈顶的栈帧

```c
// 定义一个栈帧, 栈帧中包括了 pc, 以及递归调用的各种参数, 局部变量等
typedef struct {
    int pc;
    int n;
    char from, to, via;
} Frame;

// 模拟栈帧入栈操作, 即在当前栈顶添加一个栈帧, 要注意新栈帧的 pc 初始化为 0
#define push(...) ({*(++top) = (Frame){.pc = 0, __VA_ARGS__};})
// 模拟栈帧出栈操作
#define pop() ({top--;})
// 跳转指令, 在一个栈帧内进行跳转, 最本质上就是修改 pc 的值
#define goto(pos) ({top->pc = pos - 1;})

static void hanio_non_iterative(int n, char from, char to, char via) {
    Frame stack[64];
    Frame* top = stack - 1;
    push(n, from, to, via);
    for (Frame* f = top; (f = top) >= stack; f->pc++) { 
        switch (f->pc) {
        case 0: if (f->n == 1) { fprintf(non_iterative_rst, "%c -> %c\n", f->from, f->to); goto(4); } break;
        case 1: push(f->n - 1, f->from, f->via, f->to); break;
        case 2: push(1, f->from, f->to, f->via); break;
        case 3: push(f->n - 1, f->via, f->to, f->from); break;
        case 4: pop(); break;
        default: break;
        }
    }
}
```

将递归调用的 hanio 编译得到的汇编代码类似于非递归的 case 语句, case 的顺序即为指令的顺序

所以也可以认为 c 语言本身也是一个状态机, 状态由堆栈确定, 汇编语句的执行就是状态转移, 将 c 翻译为状态机的过程即为编译

## trace syscall

各种复杂的程序其实和 minimal 都没有特别大的区别, 不过是多了些函数调用和系统调用, 程序的函数调用是很复杂的, 函数之间存在复杂的调用关系, 这里仅关心程序进行的系统调用, 使用 `strace` 跟踪程序执行的各种 syscall

在 csapp 基础部分知道了, cc 是用来编译得到汇编代码的, as 是用来汇编得到可重定位目标文件的, ld 是用来链接 .o 得到可执行目标文件的, 但有了 gcc 之后一个 gcc 就可以执行上面的所有操作, 这里希望知道 gcc 是如何进行编译的, 就可以通过 strace 进行跟踪

```c
// hello.c
#include <csapp.h>

int main() {
    fprintf(stdout, "os\n");
    return 0;
}
```

```shell
$ strace -f -o log.txt gcc -o main hello.c -lcsapp && vim log.txt
# 或者直接读进来
strace -f gcc -o main hello.c -lcsapp |& vim -
```

>   strace 的参数:
>
>   *   `-f` 表示跟踪当前进程和其创建的所有子进程
>   *   `-o` 后面跟随的输出文件, 如果没有这个的话, 默认应该直接输出到 terminal 上了吧
>
>   管道: `|&` 表示将 stdout 和 stderr 都导向管道, 在 linux 的管道机制中 `-` 可以表示是 stdout 也可以表示为 stdin, 在上面的例子中 `vim -` 就是使用 vim 编辑 stdin 的内容

如果只是为了知道 gcc 的编译过程, 其实只需要筛选 `execve` 的 syscall, 看看 gcc 是如何调用各个工具的

>   vim 下: `:g!/execve/d`

# Python 操作系统模型

高级语言实现的各种应用程序可以认为是状态机, 其行为可以分为两类: 数值计算, syscall

这里为了简化操作系统的内部实现, 使用 python 封装了一些具有相同功能的 syscall

*   sys_choose(xs): 随机从 xs 中返回一个
*   sys_write(s): 将字符串 s 输出
*   sys_spanw(fn): 创建可以运行的状态机 fn(这里可以简单理解为创建一个新的线程执行 fn)
*   sys_sched(): 随机切换到任意状态机执行(这里可以理解为操作系统进行线程调度)

使用了基于 "python syscall" 的应用程序的状态就变得很简单了, 甚至可以画出其状态转移图

"python syscall" 中最难实现的其实是 `sys_sched()`, 当控制切换到另一个状态机的时候, 前一个状态机的状态需要被保存下来, 否者下一次切换回来的时候, 当前状态机丢失了状态可能会导致程序运行出错

为了保存状态机的状态, 这里利用到了 python 语言中的 generator object 语法:

```python
def numbers():
    i = 0
    while True:
        ret = yield f'{i:b}'  # “封存” 状态机状态
        i += ret
        
n = numbers()  # 封存状态机初始状态
n.send(None)  # 恢复封存的状态
n.send(0)  # 恢复封存的状态 (并传入返回值)
```

numbers 定义了一个 generator, 调用 numbers() 后返回了一个 generator, 借助 yield 可以实现 generator 和主程序之间的通信, 桥梁就是 yield

考虑如果设计一个操作系统类: `OS`, 其上运行用户程序, 当用户程序执行系统调用的时候会通过 yield 归还控制, 而 `OS` 负责执行各种 syscall, 并将返回值返回给应用程序

```python
# application.py

count = 0

def func(name):
    global count
    for i in range(3):
        count += 1
        sys_write(f'{name} increase count by 1, count: {count}')

def main():
    n = sys_choose([1, 2, 3, 4, 5])
    sys_write(f"Thread count {n}")

    for i in range(n):
        sys_spawn(func, i)
    
    sys_sched()
```

应用程序很简单, 首先通过 sys_choose 得到一个随机的线程数量, 然后让每个线程对全局变量 count 执行 3 次自增操作

>   在实际的操作系统中 spawn 操作中得到的可能是线程, 但这里可以认为 main 是一个状态机, 并且通过 spawn 创建了 n 个并行的状态机

"操作系统"的实现相对复杂一点:

```python
# fake-os.py

from pathlib import Path
import sys
import random

class OS():

    SYSCALL = ['sys_choose', 'sys_write', 'sys_spawn', 'sys_sched']

    def __init__(self, src):
        variables = {}
        exec(src, variables)
        # fetch main function pointer from source code(main has not run yet)
        self._main = variables['main']
    
    def run(self):
        machines = [OS.StateMachine(self._main)]
        while machines:
            # wrap cases with `try` to catch generator object terminate exception
            try:
                # t := threads[0] will assign t with thread[0], and return threads[0]
                # while t = threads[0] will just make assignment
                match (t := machines[0]).step():
                    case 'sys_choose', xs:
                        t._retval = random.choice(xs)
                    case 'sys_write', s:
                        print(s)
                    case 'sys_spawn', (func, *args):
                        machines.append(OS.StateMachine(func, *args))
                    case 'sys_sched':
                        random.shuffle(machines)
            except StopIteration:
                machines.remove(t)
                random.shuffle(machines)

    class StateMachine:
        # star before args means args is an variable-length parameter
        def __init__(self, main_func, *args) -> None:
            self._func = main_func(*args)
            self._retval = None
        
        def step(self):
            syscall, args = self._func.send(self._retval)
            self._retval = None
            return syscall, args
 
if len(sys.argv) < 2:
    print(f'Usage: {sys.argv[0]} <source code file>\n')
    exit(1)

# read argv[1] and load source code
src = Path(sys.argv[1]).read_text()

for syscall in OS.SYSCALL:
    # replace `syscall` into `yield syscall` 
    src = src.replace(f'{syscall}', f'yield "{syscall}", ')

os = OS(src).run()

```

>   因为对 python 语法的不熟悉, 这里添加了部分用来解释语法的注释

整个 os 的逻辑还是很清晰的, 首先通过读取文件的方式将"应用程序"读入 fake-os, 然后进行替换, 在所有使用 "syscall" 的地方都添加了一个 yield, 使得这些函数均可以在调用到 "syscall" 的时候将控制归还给 fake-os, fake-os 针对不同的 "syscall" 执行不同的逻辑

特别的, fake-os 将应用程序视作一个状态机, 使用内部类 StateMachine 进行保存, 并 step 运行, 这里的 step 单位为 "syscall" 的个数, 所谓的 spawn 不过就是在 fake-os 内部的列表中额外添加一个 StateMachine, 所谓的 sched 不过是让 fake-os 重新执行有一次调度

尽管基于 python 的 fake-os 对真正的 os 进行了大量的简化, 但 jyy 提出的这个概念还是很好的解释了应用程序和操作系统之间的关系, 如果从 os 的角度上来看, 各种应用程序都可以认为是 os 的 "子程序", 它们在执行除了数值计算之外的任何操作时, 都需要将控制归还操作系统, 让操作系统替它们完成

而在南大的课程中还实现了一个功能更加完善的 [fake-os](https://jyywiki.cn/pages/OS/2023/mosaic/mosaic.py)

# 多处理器编程：从入门到放弃

操作系统上的文件是被所有进程共享的, 从某种程度上来讲, 操作系统和所有进程之间的关系, 类似某个进程中的 main thread 和 peer thread 之间的关系, 因此可以认为操作系统本身就是并发的程序

之前看到过程序在运行时在内存中的布局, 栈在地址空间中从高地址向着低地址增长

![](https://cdn.jsdelivr.net/gh/SunYuanI/img@latest/img/linux_program_memory.png)

而用户栈之间应该是相互独立的, 这就衍生出一个问题, 即在多线程的环境中, 栈空间在内存中是如何布局的? 不同线程的用户栈又有多大？

这里使用了一个无限递归的小程序, 粗略的对栈空间的大小进行估计:

```c
#include <csapp.h>

#define THREAD_CNT 64

typedef unsigned long long ULL;

static sem_t lock;
void* volatile low[THREAD_CNT];
void* volatile high[THREAD_CNT];

void update(int tid, void* ptr) {
    if (ptr < low[tid]) low[tid] = ptr;
    if (ptr > high[tid]) high[tid] = ptr;
}

void probe(int tid, int n) {
    update(tid, &n);

    ULL size = (ULL)high[tid] - (ULL)low[tid];
    // cal by KB
    size >>= 10;
    // little trick
    if (size > 8000) {
        P(&lock);
        fprintf(stdout, "thread: %d, size: %llu KB\n", tid, size);
        V(&lock);
    }

    probe(tid, n + 1);
}

void* routine(void* arg) {
    int tid = *(int*)arg;
    Free(arg);
    // 栈空间的 "最低位" 初始化为 Umax
    low[tid] = (void*)-1;
    // 栈空间的 "最高位" 初始化为 Umin
    high[tid] = (void*)0;
    update(tid, &tid);
    probe(tid, 0);
    return NULL;
}

int main() {
    // 设置缓存区为 NULL, 避免 fprintf 缓存输出
    setbuf(stdout, NULL);
    pthread_t threads[THREAD_CNT];
    Sem_init(&lock, 0, 1);
    int i;
    for (i = 0; i < THREAD_CNT; i++) {
        int* tid = Malloc(1 * sizeof(int));
        *tid = i;
        Pthread_create(&threads[i], NULL, routine, tid);
    }

    for (i = 0; i < THREAD_CNT; i++) Pthread_join(threads[i], NULL);

    return 0;
}

```

>   这里其实作弊了, 在打印的时候判断了, 只有当栈空间大于 8000 KB 的时候才进行打印

测试下来程序会在栈空间涨到接近 8192($2^{13}$) KB 的时候挂掉

多线程带来了很多问题, 比如:

*   原子性假设: 一条语句可能被翻译成若干条汇编指令, 就算只有一条汇编指令, 也无法保证指令在执行的过程中不被其他线程打断; 而在多个核心的情况下, 线程之间本身就是并行执行的; 因此代码的执行在多线程环境下并不是原子的; 在共享内存模型中实现原子性, 就是实现了互斥, 比如代码中看到的各种锁就是互斥的应用

*   顺序执行假设: 编译器会保证代码优化在单线程环境下的正确性, 而在多线程环境下, 编译器的优化可能导致程序出错, 比如:

    ```c
    // 当前线程在不满足 flag 的情况下进入死循环, 满足后才进行后续操作
    while (!flag);
    
    // complier optimized, 在编译器看来, flag 在循环内部没有进行任何修改, 因此读一次只要不是 true, 直接死循环
    if (!flag) {
        while (1);
    }
    
    ```

    一种解决这个问题的办法是使用 volatile 修饰变量, 相当于给编译器添加了一个内存屏障, 避免编译器将多次的内存读写优化为一次内存读写

*   处理器间可见性假设: 由于内存访问需要的巨大时间开销, 处理器一般配备了缓存, 在多处理器环境下, 某个核心对出内存的修改可能不会立刻同步到内存中, 这就可能导致处理器的缓存出现过期情况, 即某个处理器对内存的修改并不是对所有处理器立刻可见的

 # 并发控制

解决并发的方法: 对于某些特殊的代码块, 添加限制, 使其在并发的环境中退化为顺序执行, 从而保证代码块中的代码可以原子执行, 实现了这个功能即实现了互斥, 被包围的代码块为 critical section

```c
void func() {
    lock();
    // critical section   
    unlock();
}
```

互斥的实现关键就在于 `lock()` 和 `unlock()`

jyy 给出了一种尝试:

```c
int locked = UNLOCK;

void lock() {
retry:
    if (locked != UNLOCK) {
    	goto retry;
  	}
  	locked = LOCK;
}
void unlock() {
    locked = UNLOCK;
}

void func() {
	lock();
  	// critical section
	unlock();
}
```

在 lock 函数理想的状态是, 当锁未被占用时, 通过 if 判断失效, 锁被设置为占用状态, 而当锁被占用时, 会不断重新跳转到 retry 再次查询

>   未获取的锁的线程不断重新查询, 这种方式实现的是 CAS(compare and spin), 自旋锁

但实际上, if 分支判断和 locked 的赋值并不是原子的, 即可能出现两个线程先后进行 if 判断, 发现都不满足条件, 而两个线程同时对 locked 进行赋值, 从而导致两个线程进入了 critical section

## Peterson

实现 peterson 算法的前提是: 单次对内存的读/写一定是原子的, 且对内存的读写对于其他线程而言需要是可读的

通过代码的方式给出 peterson 算法的描述:

```c
int volatile flag_1 = 0, flag_2 = 0, lock = 0;

void T1() {
    flag_1 = 1;
    lock = 2;
    while (flag_2 && lock == 2);
    // critical section
    flag_1 = 0;
}

void T2() {
    flag_2 = 1;
    lock = 1;
    while (flag_1 && lock == 1);
    // critial section
    flag_2 = 0;
}
```

>   每个线程拥有独立的 flag, 和公共的 lock, 在进入 critical section 之前, 将 flag 置位, 并将公共的 lock 置为对方的 tid
>
>   当线程发现对方 flag 置位, 并且当前 lock 是对方的 tid 时, 将进入自旋状态

peterson 算法的正确性并不容易验证, 最暴力的证明方式就是枚举, 将程序的所有可能的转移的状态都枚举出来

考虑现在定义状态为: $(\text{PC}_1, \text{PC}_2, \text{flag}_1, \text{flag}_2, \text{lock})$, 那么初始状态即为 (0, 0, 0, 0, 0)

````mermaid
stateDiagram-v2
	state "(0, 0, 0, 0, 0)" as s0
	state "(1, 0, 1, 0, 0)" as s1
	state "(0, 1, 0, 1, 0)" as s2
	state "(1, 1, 1, 1, 0)" as s3
	state "(2, 0, 1, 0, 2)" as s4
	state "(0, 2, 0, 1, 1)" as s5
	state "(3, 0, 1, 0, 2)" as s6
	state "(2, 1, 1, 1, 2)" as s7
	state "(4, 0, 0, 0, 2)" as s8
	state "(2, 1, 1, 1, 2)" as s9
	state "(2, 2, 1, 1, 1)" as s10
	state "(3, 2, 1, 1, 1)" as s11
	state "(2, 2, 1, 1, 1)" as s12
	state "(4, 2, 0, 1, 1)" as s13
	state "(1, 2, 1, 1, 1)" as s14
	state "(0, 3, 0, 1, 1)" as s15
	state "(0, 4, 0, 0, 1)" as s16
	state "(2, 2, 1, 1, 2)" as s17
	state "(1, 2, 1, 1, 1)" as s18
	state "(2, 3, 1, 1, 2)" as s19
	state "(2, 2, 1, 1, 2)" as s20
	state "(2, 4, 1, 0, 2)" as s21
	state f1 <<fork>>
	state f2 <<fork>>
	state f3 <<fork>>
	state f4 <<fork>>
	state f5 <<fork>>
	state f6 <<fork>>
	state f7 <<fork>>
	state f8 <<fork>>
	state f9 <<fork>>
	state j1 <<join>>
	state j2 <<join>>
	state j3 <<join>>
	
	[*] --> s0
	s0 --> f1
	f1 --> s1
	f1 --> s2
	
	s1 --> f2
	s2 --> f3
	
	f2 --> s4
	f2 --> j1
	f3 --> j1
	f3 --> s5
	j1 --> s3
	
	s4 --> f4
	f4 --> s6
	f4 --> j2
	j2 --> s7
	
	s6 --> s8 
	s8 --> [*] : thread 1 ends
	
	s7 --> f5
	f5 --> s10
	f5 --> s9
	s9 --> s7 : thread 1 spins
	
	s10 --> f6
	f6 --> s11
	f6 --> s12 
	s12 --> s10 : thread 2 spins
	
	s11 --> s13 
	s13 --> [*] : thread 1 ends
	
	s3 --> j2
	
	s5 --> f7
	s3 --> j3
	f7 --> j3
	j3 --> s14
	f7 --> s15
	
	s15 --> s16 
	s16 --> [*] : thread 2 ends
	
	s14 --> f8
	f8 --> s17
	f8 --> s18
	s18 --> s14 : thread 2 spins
	
	s17 --> f9
	f9 --> s19
	f9 --> s20
	s20 --> s17 : thread 1 spins
	
	
	s19 --> s21
	s21 --> [*] : thread 2 ends
	
````

>   由于篇幅有限, 这里仅考虑了一个线程运行到终止的情况, 当一个线程终止后， 第二个线程不会继续 spin, 而是直接运行结束

如果只从状态转移图的角度考虑, peterson 算法确实可以保证同时只会有一个线程进入 critical section, 但如果实际运行上述代码, 则可能出现两个线程同时进入 critical section 的情况

考虑之前说过的前提, 内存读写的原子性和可见性其实是一个很强的条件, 单从编译器的优化选项并不能保证这个条件, 从硬件的角度还需要特殊的指令支持各种稀奇古怪的 barrier

所以给出的建议是: 该加锁的时候加锁, 不要试图进行优化



# 文件系统

## TODO

文件本质上可以认为是字节数组, 对于 os 而言, 不管是图片, 文本, 二进制可执行文件, 都是字节数组, 文件之间通过 low-level name 进行区分, 特别的在 unix-like 的 os 中 文件的 low-level name 是 inode(一个数字)

对于用户而言, 肯定是不希望通过数字访问文件的, 所以文件一般还具有一个 user-readable name, 这里的名字就是平时说的文件名(一个字符串)

> 用户通过向 os 提供 user-readable name 访问文件, 进行文件 I/O 操作时, os 首先会把 user-readable name 转化为 low-level name, 这个过程可以类比于 DNS 解析

os 使用目录维护 low-level name 和 user-readable name 之间的映射关系, 目录本质上也是一个文件, 只不过其对应在磁盘上的字节数组中保存的是映射对

正是因为目录也是文件, 因此目录也是具有 low-level name 和 user-readable name 的, 因此目录可以进行嵌套, 从而维护出目录的层级结构, 在 unix-like os 中最顶层目录为 `/`, 即根目录

在 [csapp](./CSAPP.md#linux file hierarchy) 中了解到, linux 使用三种数据结构维护当前被进程打开的文件, 其中 v-node table 保存了 file metadata, 而使用 inode 进行检索时就是在检索 v-node table

## VSFS(very simple file system)

### layout

unix-like 的 file system 的实现是很复杂的, 这里仅仅以仅介绍一个最简单的具有基本功能的 file system

file system 不会以字节为单位管理磁盘, 而是以 block 为单位, 这里考虑一个 block 大小为 4 KB, 同时假设磁盘一共由 64 个 block 组成, 那么这些 block 可以描述为下图:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/03/14:44:45:VSFS_blocks.png)

对于 file system 而言, 管理不同的文件显然是需要额外开销的, 并不是所有的 64 个 block 都可以用来存储用户数据, 考虑如果仅将后 56 个 block 作为可以被用户使用的 block, 这里将其称为 data region, 而最开始的 8 个 block 留给 file system 作为维护文件和自身的开销

那么此时所有的 block 可以被划分为两部分:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/03/14:48:55:VSFS_data_region.png)

file system 维护各个文件, 可能需要维护各个文件的 metadata, 比如文件的大小, 文件的访问权限, 文件的各种日期(创建日期, 上次修改日期, 上次访问日期), 文件在磁盘上保存的位置信息, 这些信息保存在 v-node table 中, 因此下一步就是在所有 block 中划分出区域保存 v-node table

这里假设 metadata 大小为 256 Bytes, 那么对于一个 4 KB 的 block, 其可以保存 16 组 metadata, 在 VSFS 中使用 5 个 block 保存 v-node table, 一共包括了 80 组 metadata

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/03/15:00:53:VSFS_inodes.png)

为了维护 v-node table 中的每个 entry 和 data region 中每个 block 的占用情况, 这里使用 bitmap 的方式进行记录, 即如果当前 block 已经被占用, 那么该 block 对应下标的 bit 被置位

>   这里使用的 bitmap 是一种维护 free space 的一种比较简单的方式, 管理磁盘空间和管理内存是类似的, 其实可以使用 free list 进行维护, 借助在 csapp 中实现的 segretated free list 可以快速完成特定大小 block 的占用和释放

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/03/15:04:24:VSFS_bitmap.png)

上图中 i 表示 inode bitmap, d 表示 data region bitmap, 其 bitmode 分别表示了 inode 和 data block 的占用情况

>   要注意的是, 上面的这种分配是比较奢侈的, 毕竟一共 80 个 metadata, 56 个 block, 分别使用 10 Bytes 和 7 Bytes 就可以描述, 但还是为各自分配了 4 KB 的空间, 这里主要是为了说明作用

至于最前面的 block, 这里将其用作 superblock, 以描述当前 file system 的信息, 包括当前 file system 一共维护了多少个 inode, 维护了多少 block, v-node table 偏移地址, data region 的偏移地址, 当前为了区分不同的 file system, superblock 还可能以一个 magic number 开头, 在 os mount 当前 file system 时, 显式的告知 os 当前 mount 的是 VSFS

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/03/15:13:41:VSFS_disk_model.png)



### inode

v-node table 的一个 entry 保存了文件的 metadata, 特别的在 ext2 中具有如下格式:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/03/18:36:19:ext2_inode.png)

文件和 inode 是一一对应的, 一个 inode 可以定位到文件实际在物理磁盘上的存储位置, 如果一个文件只有 4 KB, 那么 inode 只需要保存一个指针即可, 直接指向了磁盘中的某个 block, 但如果一个文件有 4 GB, 那么 inode 一共需要维护 $10^6$ 个指针才能实现对文件的随机访问, 此时 inode 需要 8 MB 的空间保存指针, 前面也知道 inode 是以数组的形式保存的, 即 inode 的大小需要统一对齐, 这也就意味着为了保存某些大小较大的文件, 需要让每个 inode 的大小达到 MB 这个级别, 这显然是不太合理的

>   并不是说 inode 太大不合理, 为了保存一个大小为 4 GB 的文件, 而需要的 8 MB 的开销是可以接受的, 不能接受的是为了保存某些大文件需要让每个 inode 都很大

实际 inode 的保存 block 指针的方式其实是类似与分级页表的, 当文件太大, 那么 inode 需要通过 indirect pointer 访问对应的区域, 一个 indirect pointer 指向了一个 block, 该 block 中保存数据为 direct pointer, 在 block 大小为 4 KB, 且 8 Bytes 寻址的条件下一个 block 可以存储 512 个 direct pointer, 因此一个 indirect pointer 可以指向一个大小为 2 MB 的区域(文件)

不过显然文件的大小大于 2 MB 的情况并不少见, 当一个 indirect pointer 不够用了, 根据分级页表的处理方式, 这里完全可以引入 double indirect pointer, triple indirect pointer ...

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/03/19:12:44:inode.drawio.svg)

在 inode 内 direct pointer 的数量最多, indirect pointer 其次, double indirect pointer 最少, 当然如果需要还会存在 triple indirect pointer, 其中一个 direct pointer 可以访问大小为 4 KB 的区域, 一个 indirect pointer 可以访问大小为 2 MB 的区域, 一个 double indirect pointer 可以访问大小为 1 GB 的区域...

具体每种 pointer 的数量不同的 os 进行了不同的取舍, 不过各种 pointer 之间的关系基本都符合上面的描述, 因为大部分文件基本上都不会超过 4 KB (目录也算是文件)

>   其实除了上面这种直接通过指针指向每个 block 的方式之外, 另一种常用的方式是使用 extents 的方式分配, 即使用一个指针记录起始地址和一个变量记录占用的空间大小
>
>   使用 extents 的好处是, 其会一次分配一段连续的磁盘空间, 而当文件太大时, extents 也允许使用多个指针分配多段连续的空间
>
>   extents 是有助于减少存储碎片化的, 一个文件占用了多段连续的空间
>
>   当然使用 link list 维护一个文件的多个 block 之间的关系也是可行的, 每个 block 包含一个指向下一个 block 的指针, 不过这种方式并不利于文件的随机访问, 毕竟遍历链表的成本是很高的

### directory

前面也说过了 directory 中保存的是 low-level name 和 user-level name 之间的映射关系, 而 os 肯定是允许用户为文件起长度不同的 user-level name, 因此映射对的长度应该不是固定的, 所以 directory 保存 entry 时还包括了其大小信息

考虑一个名为 dir 的目录具有如下结构, 可以看到其中包含了三个文件(或者目录), entry 记录的不只有映射本身, 还包括了 entry 的大小和 name 的长度信息

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/04/10:04:56:VSFS_dir.png)

>   名为: '.' 的文件对应的 inode 为当前目录, 名为: '..' 的文件对应的 inode 为父级目录

在实际的 directory 实现中, entry 保存的信息可能更多, 比如包含了文件的类型信息, 文件部分的 metadata

### read & write

首先说明的是, 程序对磁盘上的文件进行读写时, 首先需要将文件的对应 block 加载到内存中, 然后才能进行 read & write 操作, 在完成这个操作后, 由于更新尚未同步到磁盘中(持久化), 此时可能出现更新丢失的情况

这里考虑 file system 已经被 os mount 的情况, 此时该 file system 的 superblock 已经被加载进入内存了, 这意味着 os 可以找到当前 file system 的根路径

#### read from disk

不管是 read 还是 write, 操作文件之前都需要先 open 这个文件, 得到一个 file descriptor

对于应用程序而言, 打开文件直接通过 `Open("/foo/bar", O_RDONLY)` 即可, 而对于 os 而言, 打开这个文件就比较麻烦了

>   注意到这里使用的是 `Open` 即 csapp 的封装函数

对于 os 而言, 打开一个文件就是要找到这个文件的 inode, 然后在 open file table 中添加一个 entry (当然如果已经添加了 entry, 就不需要再添加了) 指向 v-node table 中的 inode, 并在对应进程的 file discriptor table 中添加一个 entry 并让其指向 open file table 中刚刚添加的 entry

os 定位 inode 的唯一依据是应用程序提供的路径, 在上面的例子中路径的开头为 `/` 即根路径, 在 unix-like 的 os 中根路径的 inode 为 2 (类似一种约定)

>   就算不是 2, 在 superblock 中也可以获取根路径的 inode 信息

os 首先访问 inode 为 2 的 block: 首先定位到 disk 上的 v-node table, 然后根据 inode 找到对应的 entry, entry 中包含了 direct pointer 和 indirect pointer, 借助各种 pointer 找到存储了映射对的 block, os 在各种映射对中遍历, 找到 user-level name 为 foo 的 pair, 获取到 foo 对应的 inode

至此 os 通过根目录的 inode 获取到下一级目录的 inode, os 也将通过同样的方式获取最终需要读的文件 `bar` 的 inode, 获取 inode 之后 os 进行上面提到各种修改, 最终返回应用程序一个 file descriptor

在文件 open 之后, 读取操作就变得很简单了, os 会根据文件位置向 disk 获取对应的 block, 同时要注意的是 v-node table 中保存的 metadata 可能需要在读取的过程中修改, 比如 last visited time

文件 close 操作对于 os 而言是很简单的, 直接在当前进程的 file discriptor table 中删除对应的 entry, 同时修改 open file table 中 entry 的 refcnt, 如果 refcnt 为 0, 则还需要删除 open file table 中的对应 entry

按照时间上的先后顺序读取磁盘, 得到下表

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/04/10:49:59:VSFS_read.png)

>   如果仅仅是 read 某个文件, 那么将不涉及 bitmap 的读写, 只有当考虑新建文件, 文件扩容, 删除文件的时候才会涉及到 bitmap

#### write to disk

相比之下 write 显得麻烦了很多, 最开始创建的文件可能很小, 但随着数据的写入, 文件会越来越大

考虑还是针对文件 `/foo/bar`, 此时应用程序打开文件的方式为: `Open("/foo/bar", O_WRONLY | O_CREAT)`

如果文件已经创建了, 那么在 open 阶段和 read 是类似的, 但如果文件尚未被创建, 那么将涉及到大量的 I/O, 当 os 找到 foo 的 inode 时, 首先会读取 `foo` 的 data block, 遍历所有的映射对, 找不到名为 `bar` 的 inode, 此时 os 需要读取 inode bitmap, 并找到一个空闲的 inode, 并为 `bar` 分配一个 inode, 并将结果写回磁盘上的 inode bitmap 中, 当然因为是创建在 `foo` 的子目录下, 还需要在 `foo` 的 data block 中添加一个 pair, 对应了 `bar` 和刚刚分配得到的 inode, 到最后才完成对 `bar` 的 inode 的一次读写

>   要注意, 上面的这些 IO 还不包括当目录的 data block 不足时, 还需要对目录的 data block 进行扩容, 此时还需要修改目录的 data bitmap 

这里涉及的 IO: `foo` data block 的 read, inode bitmap 的 read & write, `foo` data block 的 write, `bar` data block 的 read & write

每个对于文件的写入操作都涉及到 5 次 IO 操作: bitmap 的 read & write(分别用来找到当前可以进行写入的 data block 和将需要新占用的 block 对应的 bit 置位并写入磁盘), `bar` inode 的 read & write (分别用来找到对应的 data block 和修改文件的 metadata), `bar` data block 的 write

上述过程按照时间先后顺序可以得到下图:

<img src="https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/04/14:28:01:VSFS_write.png" style="zoom:100%;" />

### cache & buffer

之前那对于 data block 的读写操作都没有考虑到缓存的情况, 如果程序多次读某个文件的同一个 block, 那么此时 os 还是会不断向磁盘发出请求, 这显然是不太合理的

涉及到缓存, 就涉及到缓存的替换算法, 以及缓存的大小, 不同的缓存策略都可以写好多论文了, 这里不详细说了

有了缓存之后, 只有第一次打开文件的时候才需要 os 不断遍历访问路径, 不断找到目录的 inode 和其对应的 data block, 而后续进行读文件的时候(缓存足够大, 而文件是小文件), 整个文件会被一点点加载到内存中, 第二次打开相同文件, 读取时, 所有的查询和读文件操作都发生在内存中

写操作并不如读操作那么直接, 就算将 data block 缓存进入内存中, 进行文件的修改时, 还是需要考虑将文件持久化操作, 因此所有的 block 最终还是要写回磁盘, 但此时 os 通过延迟写操作实现优化: 在写入了一定数目的 block 之后再将其写回磁盘, 一次写入大量的 data block; 此外还可能避免重复操作, 如果程序先对某个 data block 进行了修改, 随后又将内容恢复, 那么此时并不需要将这两个写操作同步到磁盘上

但延迟写操作也是存在一些问题的, 修改如果不能及时持久化, 那么修改就可能丢失, 对于数据库程序而言, 数据的丢失可能造成巨大损失, 为了防止数据丢失, 操作系统向磁盘写操作时, 存在类似事务的机制, 即开始进行写操作时开启事务, 完成写操作后提交事务

## FFS

VSFS 实在是太慢了, 文件查找的时候, 首先需要找到 inode, 然后找到 inode 指向的 data block， inode 和 data block 并不是紧挨着的, 因此访问 data block 的时候不能很好的利用到 space locality

此外 VSFS 在不断分配回收 data block 时会出现 fragmented 的情况, 即一个文件的 data block 可能是散布在整个 data block 中的, 而不是连续的, 这会进一步恶化 space locality

考虑 data block 按照如下方式分配, A, B, C, D 四个文件依次占用空间大小为两个 data block

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/09/19/11:11:48:VSFS_fragemented-Page-1.drawio.svg)

随后文件 B 和 D 被删除, 其占用的 data block 被回收

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/09/19/11:16:11:VSFS_fragemented-Page-2.drawio.svg)

此时考虑一个占用 4 个 data block 的新文件 E, 那么 E 占用 data block 后内存结构如下:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/09/19/11:16:57:VSFS_fragemented-Page-3.drawio.svg)

此时 E 的 data block 不是连续的, 当访问文件 E 时, 会因为 space locality 的缺失, 额外增加 data block 访问时间

FFS 通过将文件系统分组提高文件访问效率

### cylinder group

物理磁盘是由多个叠片 (platter) 堆叠得到的, 一个圆形叠片是通过多个 tracks 组成的, track 上各处距离磁盘中心的位置是相同的

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/09/19/21:53:55:FFS_cylinder_group.png)

上图展示的磁盘由 6 个叠片组成, 每个叠片上具有 4 个 track

此外管理磁盘时: 不同叠片, 相同距离的 track, 可以组成一个 cylinder, 这样不同的 track 就可以构成不同的 cylinder, 有多少个 track 就可以构成多少个 cylinder

多个 cylinder 构成一个 cylinder group, 上图展示的 cylinder group 中包含了 3 个 cylinder (最内侧的黑色 track 并没有计入 cylinder group 的统计)

由于物理磁盘并不会直接被 os 管理, 因此 os 并不能准确获取到某个 block 所在的 cylinder 信息, 因此实际 os 只是将所有的 block 分为若干个 group

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/09/19/22:20:42:FFS_block_group.png)

通过将一个文件置于一个 group 中, 可以利用 space locality 减少访问任意两个 block 需要的寻址时间

每个 group 内部, 内存结构和之前的 VSFS 完全一致 (甚至包括 super block)

### allocation policy

一句话来说: keep related stuff together, FFS 会将具有相关性的 block 放入同一个 group 中, 将不相关的 block 放入不同 group 中

FFS 采用启发式的放置方式, 当 FFS 需要放置一个目录时, 会搜索所有的 cylinder group, 找到一个包含了大量 free inode block 和仅保存了少量 directory block 的 group 作为目标 group

>   包含大量 free inode block 可以在当前目录下更多的文件，只保存了少量 directory block 可以尽可能避免多个目录相互竞争 free inode block

当 FFS 需要放置一个文件时, 它会保证文件的 inode block 和 data block 位于一个 group 中, 在此基础上 FFS 还会尽可能保证文件和其所在的目录位于同一个 group 中

按照上面的放置策略, 考虑一个放置的例子: 根目录为 `/`, 根目录下包含了两个目录 `a/` 和 `b/`

在子目录 `a/` 中包含了 3 个文件 `c`, `d`, `e`; 在子目录 `b/` 中包含了 1 个文件 `f`

每个目录占用 1 个 data block, 每个文件占用 2 个 data block

那么 FFS 会按照下面的方式放置目录和文件

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/09/20/11:46:13:FFS_allocation.png)

### large files

将整个 disk 分为若干 group 后确实可以提高小文件的访问速度, 但对于大文件并不友好, 大文件很容易填充满整个 group 的 data group, 从而导致后续其他小文件丢失 space locality

解决方式也很简单, 直接将大文件的 data block 放入不同的 group 即可, 这样做会破坏顺序访问大文件时的 space locality, 但这有助于后续提升在相同目录下访问小文件时的性能

此外通过增加 chunck size 可以改善 large file 分布在多个 group 时, 顺序访问的 space locality 的; 要注意的是 chunck size 和 block size 并不相同, 一个 chunck 可以包含若干 block

>   一个很简单的计算, 如果磁盘 seek & position 的平均开销为 10 ms, 磁盘到内存的 data transfer speed 为 40 MBps, 这里考虑内存以 chunck 为单位加载数据
>
>   如果加载开销为 50% 时, 此时寻道时间和 chunck transfer 时间均为 10 ms, 那么一个 chunck 大小为: $10 \times 10 ^{-3} \times 40 \times 1024 \text{KB} = 409.6 \text{KB}$
>
>   如果加载开销为 10% 时, 此时 chunck transfer 时间为 90 ms, chunck size 为: $90 \times 10 ^{-3} \times 40\text{MB} = 3.6 \text{MB}$
>
>   如果加载开销为 1% 时, chunck size 为: $990\times 10 ^{-3} \times 40 \text{MB} = 39.6 \text{MB}$
>
>   因此只要 chunck 足够大, 那么大部分时间将花在 data transfer 上
>
>   不过要注意的是, 磁盘 seek & position 的时间受限于机械臂本身的设计, 因此多年来并没有显著的提升, 而磁盘传输速率却提升的很快, 因此在使用最新的磁盘, 达到相同开销的情况下, 需要的 chunck size 是更大的 (不过大家都开始用上固态了, 寻道时间已经没有那么久了)

FFS 具体的放置方案: inode 中前 12 个 direct pointer 指向了当前 group 中的 data block, 其余的所有 indirect pointer 都指向了其他的 group 中的某个 data block, 实际的数据也保存在了其他的 group 中, 所以在 FFS 中, 只有文件的前 48 KB 保存在了当前 group 中, 后面的数据都保存在了其他 group 中

>   在 block size 为 4 KB, 采用 64 bit 寻址的情况下, 一个 indirect pointer 指向的 block 可以索引 512 个 data block, 因此一个 indirect pointer 可以索引 2 MB 的 data block

### one more thing

FFS 还针对小文件进行了优化, 那些大小小于 4 KB 的文件才是多数, 在 FFS 中存在大小为 512 字节的 sub-block, 当需要保存一个文件时, 数据首先会被保存在 sub-block 中, 当文件占用 block size 总和达到了 4 KB 后, FFS 会将数据从 sub-block 复制到一个空闲的 block 中, 并释放其占用的 sub-block

这种方式有效的避免了 internal fragment, 但也因为复制引入了额外的开销, 为此最初的人们修改了 libc, 使得所有的 write 操作是带有缓存的, 当程序 write 大小达到 4 KB 后直接将数据写入 data block

此外在 FFS 之前文件连续的 data block 会按照顺序保存在磁盘上, 如下图:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/09/24/22:06:31:FFS_original_disk_layout.png)

当按照顺序读取数据的时候, 磁盘控制器根据接收到的指令读取对应的扇区, 首先收到指令 0, 磁盘旋转到扇区 0, 进行数据读取, 随后当磁盘控制器收到指令 1 时磁盘已经旋转到扇区 1 了, 按照 three easy piece 上的说法, 此时已经来不及了, 需要旋转一周, 转到扇区 1 的开头, 因此在这种布局的情况下, 顺序读取 block 时需要多次旋转磁盘

后面最初的人们想到了交叉保存连续 block

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/09/24/22:12:31:FFS_parameterized_disk_layout.png)

此时读取所有的 block 需要磁盘旋转两周, 人们对这个性能还不是很满意

再后面他们给磁盘设计了缓存, 不管读哪个 sector, 磁盘首先查一下内部缓存, 如果 cache-miss, 直接将对应的 track 缓存下来...

FFS 还引入了 symbolic link, 相比于 hard link, 其区别在于 symbolic link 的 data block 保存的是路径信息, 其 inode 和 link 到的文件并不相同

对于 hard link, 由于相同 inode 的限制, 必然存在访问两个文件之一时 inode 保存在和 directory 不同的 group 的情况, 此外 hard link 本身是不支持跨磁盘的, 因为 hard link 需要 inode 区分不同的文件, 而默认挂载不同的磁盘可能存在相同 inode 的情况, 因此 hard link 只能存在于一个磁盘内部

这些问题都可以通过 symbolic link 解决

# ChCore

这个实验官方放在了 [gitee](https://gitee.com/ipads-lab/chcore-lab-v2) 上, 到今天(23/6/1) 为止, 已经没有了后续更新, 所有的 lab 都已经是最新的状态, 所以不需要再考虑原文档中关于远程分支的若干复杂的配置了

直接从官方仓库 fork 到自己的仓库, 然后 clone 下来, 默认的 upstream 就已经是自己的仓库了, 因为不需要再考虑原仓库更新的情况, 因此不需要重新设置 upstream 了, 如果想保存在远程就直接 push, 不然的话, 每次 pull 对应的远程分支即可

## 环境

官方建议环境: Ubuntu 20.04(Debian 10) 以上

chcore 是一个应用于 raspi3 的内核, 在进行内核编译的时候需要将其编译为适用于 arm 平台的代码, 官方给出了两种方式:

*   使用 docker(建议)
*   使用 gcc-aarch64-linux-gnu: 用来在 x86 平台编译得到适用于 aarch64 平台的可执行文件(在这个 lab 中为 .img 镜像 => 在 qemu 中运行)

更推荐上者, 官方已经给出了 Dockerfile, 并且默认的 makefile 就是写好了使用 Dockerfile 进行内核编译

在安装 ubuntu server 的时候默认没有携带 docker, 需要到官网下: [Install Docker Engine on Ubuntu | Docker Documentation](https://docs.docker.com/engine/install/ubuntu/)

我使用的这个 server 是一个最简的, 什么都没有, 这里需要确认安装:

*   gcc

*   make

*   cmake

*   qemu-system-arm(用来模拟 arm 平台硬件)

*   gdb, gdb-multiarch: 后一个用来连接到 qemu 中的 gdb server

*   binutils-aarch64-linux-gnu: 用来通过 objdumpa 反编译适用于 aarch64 平台的可执行文件

    >   如果之前安装了 `gcc-aarch64-linux-gnu`, 这里就不需要重复安装了, 因为 `binutils-aarch64-linux-gnu` 本身是作为其子集存在的
    >
    >   如果之前选择的是通过 docker 进行编译, 就需要安装了
    
*   expect: 用来评分的

## AArc64

因为是 arm 架构, 所以是 RISC 指令集, 指令长度都是 4 个字节(32 位), aarch64 可以操作 64 位地址

也是因为 RISC 指令集的原因需要的寄存器数量比 X86 更多:

*   31 个通用寄存器: X_0 ~ X_30, 当使用其引用 32 位地址的时候, 使用 W_0 ~ W_30 表示

    >   类似 X86 中 %rax 与 %eax 的关系

*   4 个栈寄存器: SP_EL0, SP_EL1, SP_EL2, SP_EL3, 其中 SP_EL0 用于用户态, SP_EL1 用于内核态

    >   X86 中的栈寄存器为 %rsp

*   3 个异常链接寄存器: ELR_EL1, ELR_EL2, ELR_EL3

*   3 个程序状态寄存器: SPSR_EL1, SPSR_EL2, SPSR_EL3

*   1 个 pc

上述寄存器只有程序状态寄存器是 32 位的, 剩下的都是 64 位的

后面的部分可以先跳过, 后面看不懂代码后再调回来

### shift mode

在 aarch64 中 shift 可以用到各种指令中, 一共支持的移位方式有四种:

*   LSL: logical shift left(逻辑左移), 低位补 0 
*   LSR: logical shift right(逻辑右移), 高位补 0
*   ASR: arithmetic shift right(算数右移), 高位补符号位(负数补 1, 正数补 0) 
*   ROR: rotate right(循环右移)

### load/store address mode

这里说的是 aarch64 从内存中加载/存储到内存的寻址方式, 一个基本的写法为 `LDR Xd, []`, 这里表示将 `[]` 对应地址的数据读取到寄存器 Xd 中

在 aarch64 中读取内存的写法其实很容易看, 只要使用了 `[]` 包裹的, 那么就表示从括号内运算得到的地址读取数据

括号内的写法有根据不同的模式区分为:

>   在下面的写法中大括号表示可以省略

*   base plus offset: 

    *   `[Xn{, #imm}]`: 对应的地址为 Xn + imm(立即数), 比如 `LDR x0, [x1, #4]` 表示将内存中 x1 + 4 的数据读取到 x0 中
    *   `[Xn, Xm{,  #imm}]`: 这里的地址会受到移位的影响, 地址为 Xn + Xm << imm(这里以逻辑左移举例)

*   pre-indexed: `[Xn, #imm]!` 地址的计算方式和上面一样, 区别在于这里还会修改基址寄存器 Xn

*   post-indexed: `[Xn], #imm` 目标地址就是 Xn 中的地址, 完成指令后会修改 Xn

    >   pre-indexed 和 post-indexed:
    >
    >   *   pre-index 需要内存中的地址为 Xn + imm
    >   *   post-index 需要内存中的地址为 Xn
    >   *   二者在完成指令后都会修改基址寄存器为 Xn + imm

*   literal: 比如 `LDR x0, =my_const`, 其中 `=` 表明 `my_const` 是一个字面量, 这里需要注意的是使用 literal 方式时, 对字面量本身有位置限制, 其与当前 pc 的偏移需要可以通过 19 bit 的符号数表示, 也即最大偏移: $(2^8 - 1) \text{KB}\approx (256 \text{KB})$ 最小偏移: $2^8\text{KB} = 256 \text{KB}$, 当考虑 4 字节对齐的情况时, 地址空间的偏移可以达到 $\pm 1\text{MB}$

### 指令

#### AND

API: 

*   立即数: `AND Xd, Xn, #imm` => Xd = Xn AND imm
*   寄存器: `AND Xd, Xn, Xm{, shift #amount}` => Xd = Xn AND shift(Xm, amount), 其中 shift 符合 [shift mode](#shift mode)

#### ORR

Bitwise OR.

API:

*   立即数: `ORR Xd|SP, Xn, #imm` => Xd = Xn OR imm
*   寄存器: `ORR Xd|SP, Xn, Xm{, shift #amount}` => Xd = Xn OR shift(Xm, amount), 其中 shift 符合 [shift mode](#shift mode)

#### LDR

load resigter

API: `LDR Xt, []`, 中括号内部的地址的写法可以见上面[load/store address mode](#load/store address mode)

#### CMP

Compare, 其实就是通过减法设置状态寄存器

API: 

*   立即数: `CMP Xn/SP, #imm{, shift}`, 这里的 imm 范围为 0 ~ 4095, 而 shift 的取值只能是 0 或者 12, 在填写 shift 的情况下, 执行 LSL(逻辑左移)
*   寄存器(extened): `CMP Xn/SP, Xm{, extend #{amount}}`
*   寄存器(shifted): 

#### B

branch, 类似于 X86 中的 unconditional jump: `jmp`

API: `B label`

#### BL

branch with link, 类似于 X86 中的 `call`

和 B 最大的区别在于, BL 会保存程序返回地址, 即下一条指令的地址, 在 aarch64 中指令都是 32 bit 长度的, 因此下一条指令的地址为 PC + 4

返回地址保存在 LR(link register)中, 特别的在 aarch64 中使用 X30 作为 LR

API: `BL label`

#### CBZ

Compare and Branch on Zero, 类似于 X86: 先 `cmp` 然后 `je`

API: `CBZ Xt, label`, 当 Xt 为 0 时跳转到 label 处

#### 和 system register 交互

##### MRS

Move, Resigter, System => 将 system register 中的值保存在某个 general register 中

API: `MRS Xt, (systemreg|Sop0_op1_Cn_Cm_op2)`, 表示将 system register 中的值复制到 Xt 中

比如: `MSR x0, mpidr_el1`, 就是将 system register(mpidr_el1) 中的值保存在 x0 中

##### MSR

Move, System, Register => 某个 general register 的值赋给 system register

API: `MSR (systemreg|Sop0_op1_Cn_Cm_op2), Xt`, 表示将 Xt 的值赋给 system register

比如: `MSR elr_el2, x9`, 就是将 x9 的值赋给 elr_el2(异常链接寄存器)

#### 无符号数位扩展(UBFM)

Unsigned Bitfield Move, 用来将某个寄存器的某些位复制到另一个寄存器的某些位

API: `UBFM Xd, Xn, #<immr>, #<imms>`, 表示将 Xn 中 (immr, imms) 位复制到 Xd 中, 因为是无符号数操作, 高位和低位填充 0

比如: `UBFM X1, X0, #8, #4` 表示将 X0 的 (11 ~ 8) 位 复制到 X1 中

当知道需要复制的位数的时候可以使用 UBFX(Unsigned Bitfield Extract), API: `UBFX Xd, Xn, #Lsb, #width`, 等价于 `UBFM Xd, Xn, #Lsb, #(Lsb + width - 1)`

此外, 当知道需要复制的数据长度时, 还可以直接按照字节长度取出

有很多等价形式:

*   UXTB(Unsigned Extend Byte), 等价于 `UBFM Xd, Wn, #0, #7`
*   UXTH(Unsigned Extend Halfword), 等价于 `UBFM Xd, Wn, #0, #15`
*   UXTW(Unsigned Extend Word), 等价于 `UBFM Xd, Wn, #0, #31`

#### 符号数位扩展(SBFM)

Signed Bitfield Move, 用来将某个寄存器的某些位复制到另一个寄存器的某些位, 和上面 UBFM 最大的区别在于这里进行符号扩展, 

API: `SBFM Xd, Xn, #<immr>, #<imms>`, 表示将 Xn 中 (immr, imms) 位复制到 Xd 中, 因为是符号数操作, 低位补充 0, 高位补充符号位

比如: `SBFM X1, X0, #8, #4` 表示将 X0 的 (11 ~ 8) 位 复制到 X1 中

当知道需要复制的位数的时候可以使用 SBFX(Signed Bitfield Extract), API: `SBFX Xd, Xn, #Lsb, #width`, 等价于 `SBFM Xd, Xn, #Lsb, #(Lsb + width - 1)`

此外, 当知道需要复制的数据长度时, 还可以直接按照字节长度取出

有很多等价形式:

*   SXTB(Signed Extend Byte), API: `SXTB Xd, Wn`, 等价于 `SBFM Xd, Wn, #0, #7`
*   SXTH(Signed Extend Halfword), API: `SXTH Xd, Wn`, 等价于 `SBFM Xd, Wn, #0, #15`
*   SXTW(Signed Extend Word), API: `SXTW Xd, Wn`, 等价于 `SBFM Xd, Wn, #0, #31`

#### ADR

API: `ADR Xd, label`, 用来计算 label 和当前 PC 之间的相对地址, 并把相对地址的取值存储到 Xd 中

### system register

#### mpidr_el1

Multiprocessor Affinity Register, 在多核环境下, 用来标识当核心, 在 aarch64 下具有如下结构:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/06/06/10:34:59:mpidr_el1.png)

其中用来标识核心的字段只有: Aff0, Aff1, Aff2, Aff3

#### CurrentEL

Current Exception Level, 保存当前处理器的异常级别

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/06/07/11:50:54:CurrentEL.png)

尽管这是一个 64 位的寄存器, 但只 2 bit 有用, 一共 4 个异常级别, 正好 2 bit 就可以表示

#### SCTLR_EL1

一个用来对 EL1 和 EL0 进行各种通用配置的寄存器

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/06/09/16:27:34:SCTLR_EL1.png)

各种字段的含义建议直接查手册 ...

### 页表

aarch64 下常用的配置为低 48 位参与地址翻译, 4 级页表, 虚拟页大小为 4KB, 物理页大小和虚拟页大小相同, 从而实现一个虚拟页可以直接映射到一个物理页

页表基址寄存器为 TTBRx_EL1, 其中 x 可以取 0 或 1, 分别表示低地址翻译和高地址翻译, 因为习惯上操作系统内核运行在高地址中, 所以一般 TTBR0_EL1 保存的页表映射的是用户地址空间

>   因为只有低 48 位参与地址翻译, 因此常用配置用户地址空间范围: 0x0000_0000_0000_0000 ～ 0x0000_ffff_ffff_ffff
>
>   内核地址空间范围: 0xffff_0000_0000_0000 ～ 0xffff_ffff_ffff_ffff

页表的每一项(映射地址)大小为 8 Bytes, 页表本身的大小一般为 4 KB, 从而一个页表中通常有: $\frac{4\text{K}}{8} = 512(2^9)$ 项

考虑在 4 级页表条件下, 有效的低 48 位虚拟地址由: 

*   9 位 L0 page table offset
*   9 位 L1 page table offset
*   9 位 L2 page table offset
*   9 位 L3 page table offset
*   12 位(4 K大小)物理页内偏移量组成

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/06/12/11:17:03:aarch64_address_trans.svg)

特别的为了增加 TLB 的命中率, aarch64 中支持大页的存在, 一个三级页表可以直接指向一个大小为 2 MB 的 block, 从而使得一个 TLB 可以映射到物理内存中 2 MB 的 block(大页可以通过配置页表的某个 bit 实现)

对于 L0, L1, L2 级, PTE 的最低位 (bits[0]) 为 0 时, 该 PTE 无效; 对于有效的 PTE 而言, 当次高位 (bits[1]) 为 0 时, 该 PTE 指向一个 block (大页), 而次高位 (bits[1]) 为 1 时指向下一级 PTE 

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/06/12/14:54:45:aarch64_page_table_1)

对于 L3 级 PT, 其次高位 (bits[1]) 为 0 时表示当前 VP 尚未映射到一个 PP, 此时应该触发缺页异常

![aarch64_page_table_2](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/06/18/14:30:06:aarch64_page_table_2) 

## 机器启动

### 从汇编开始

在填写汇编的时候查阅了很多手册, 也找到了一个基于 [arm 的操作内核]([buzzxI/zircon: Zircon Kernel 文档以及源码注释 (github.com)](https://github.com/buzzxI/zircon)), 其中 bootloader 部分可以作为参考

首先看 `_start` 函数, 这个是 os 的入口函数, 默认位置在 `kernel/arch/aarch64/boot/raspi3/init/start.S` 下, 在 chcore lib 提供的环境中, qemu 会启动 4 个核心, 但通常只会让一个核心先进入初始化, 所以上来就问, 说 chcore 是如何让一个核心进入初始化流程, 而其余核心暂停执行的, 这里给出的提示就是看代码, 这里最关键的就是 _start 的前 3 行

```assembly
/* start.S */

/* other code */

BEGIN_FUNC(_start)
	mrs	x8, mpidr_el1
	and	x8, x8,	#0xFF
	cbz	x8, primary

	/* hang all secondary processors before we introduce smp */
	b 	.
	
/* other code */
```

首先将 `mpidr_el1` 中的值移到 x8 中, 在[前面](#mpidr_el1)已经说过了 `mpidr_el1` 是 system register, 用来表示当前的处理器核心的

然后取出 x8 的低 8 bit, 并和 0 进行比较, 只有低 b 位都是 0 的情况下, 才会跳转到 primary 处, 否则停留在原地

也就是通过这种方式, 使得只有一个核心会执行 _start 函数中的 primary 逻辑, label primary 也很简单:

```assembly
primary:
	/* Turn to el1 from other exception levels. */
	bl 	arm64_elX_to_el1

	/* Prepare stack pointer and jump to C. */
	ldr 	x0, =boot_cpu_stack
	add 	x0, x0, #INIT_STACK_SIZE
	mov 	sp, x0

	bl 	init_c

	/* Should never be here */
	b	.
```

首先通过 bl 调用 arm64_elX_to_el1, 将异常级别降低为 EL1, 后续进行栈的初始化(通过 ldr 得到栈底, 再通过 add 得到栈顶, 并将栈顶赋给栈指针 sp), 初始状态下得到一个大小为 #INIT_STACK_SIZE 的栈

>   #INIT_STACK_SIZE 定义在 `kernel/arch/aarch64/boot/raspi3/include/consts.h` 中, 大小为 0x1000, 即$2^{12}$($4\text{KB}$)

最后跳转到 init_c

首先看一下 arm64_elX_to_el1 的实现:

```assembly
BEGIN_FUNC(arm64_elX_to_el1)
	/* LAB 1 TODO 1 BEGIN */
	
	/* LAB 1 TODO 1 END */

	// Check the current exception level.
	cmp x9, CURRENTEL_EL1
	beq .Ltarget
	cmp x9, CURRENTEL_EL2
	beq .Lin_el2
	// Otherwise, we are in EL3.

	// Set EL2 to 64bit and enable the HVC instruction.
	mrs x9, scr_el3
	mov x10, SCR_EL3_NS | SCR_EL3_HCE | SCR_EL3_RW
	orr x9, x9, x10
	msr scr_el3, x9

	// Set the return address and exception level.
	/* LAB 1 TODO 2 BEGIN */

	/* LAB 1 TODO 2 END */

.Lin_el2:
	// Disable EL1 timer traps and the timer offset.
	mrs x9, cnthctl_el2
	orr x9, x9, CNTHCTL_EL2_EL1PCEN | CNTHCTL_EL2_EL1PCTEN
	msr cnthctl_el2, x9
	msr cntvoff_el2, xzr

	// Disable stage 2 translations.
	msr vttbr_el2, xzr

	// Disable EL2 coprocessor traps.
	mov x9, CPTR_EL2_RES1
	msr cptr_el2, x9

	// Disable EL1 FPU traps.
	mov x9, CPACR_EL1_FPEN
	msr cpacr_el1, x9

	// Check whether the GIC system registers are supported.
	mrs x9, id_aa64pfr0_el1
	and x9, x9, ID_AA64PFR0_EL1_GIC
	cbz x9, .Lno_gic_sr

	// Enable the GIC system registers in EL2, and allow their use in EL1.
	mrs x9, ICC_SRE_EL2
	mov x10, ICC_SRE_EL2_ENABLE | ICC_SRE_EL2_SRE
	orr x9, x9, x10
	msr ICC_SRE_EL2, x9

	// Disable the GIC virtual CPU interface.
	msr ICH_HCR_EL2, xzr

.Lno_gic_sr:
	// Set EL1 to 64bit.
	mov x9, HCR_EL2_RW
	msr hcr_el2, x9

	// Set the return address and exception level.
	adr x9, .Ltarget
	msr elr_el2, x9
	mov x9, SPSR_ELX_DAIF | SPSR_ELX_EL1H
	msr spsr_el2, x9

	isb
	eret

.Ltarget:
	ret
END_FUNC(arm64_elX_to_el1)
```

>   看起来有点麻烦, 大体上其实时根据当前的异常级别进行不同的设置, 最后进行异常降级

第一个 TODO 需要获取当前核心的异常级别, 提示中也说了, 在 system register CurrentEL 中可以获取到对应的异常级别

```assembly
/* LAB 1 TODO 1 BEGIN */
MRS x9, CurrentEL
/* LAB 1 TODO 1 END */
```

通过 gdb 调试得到执行后的 x9 为 0xc, 即 0b1100, 此时异常级别为 3

在降低异常级别之前需要设置当前级别的控制寄存器, 用来控制低一级别的状态(通过 scr_el3 设置 EL2 的行为, 通过 hcr_el2 设置 EL1 的行为)具体的配置需要查阅手册

这里的第二个 todo 需要编写从 EL3 跳转到 EL1 的逻辑, 本以为很难写, 但这里需要设置的只有 elr_el3(exception link register, 用来保存进入 EL3 后的异常返回地址) 和 spsr_el3(save program status register, 用来保存在 EL3 中完成异常处理后的状态信息)

这里可以参考在后面将 EL2 降级为 EL1 的逻辑

```assembly
.Lno_gic_sr:

	/* other code*/

	// Set the return address and exception level.
	adr x9, .Ltarget
	msr elr_el2, x9
	mov x9, SPSR_ELX_DAIF | SPSR_ELX_EL1H
	msr spsr_el2, x9

	isb
	eret
```

第二个 todo 和这里的代码完全一致, 直接把 el2 换成 el3 即可

```assembly
	// Set the return address and exception level.
	/* LAB 1 TODO 2 BEGIN */
	adr x9, .Ltarget
	msr elr_el3, x9
	mov x9, SPSR_ELX_DAIF | SPSR_ELX_EL1H
	msr spsr_el3, x9
	/* LAB 1 TODO 2 END */
```

### 第一行 c 代码

从汇编跳转到的第一个 c 函数是 `init_c` 这是一个初始化函数, 首先进行 .bss 段的初始化, 然后进行 uart 的初始化, uart 是用来输出的串口, 具体的可以看: [raspberry-pi-os/rpi-os.md at master · s-matyukevich/raspberry-pi-os (github.com)](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/docs/lesson01/rpi-os.md#mini-uart-initialization)

总之 uart 的初始化是一个十分复杂的过程, 和硬件上的端口一一对应, 不过好在 chcore 中并不需要完成初始化函数的编写, 在完成 uart 端口的初始化后, 理论上就可以让 chcore 输出打印到 qemu 中了

>   在 chcore 中, 关于 uart 的设置函数放在了: `kernel/arch/aarch64/boot/raspi3/peripherals/uart.c` 中

这里 chcore 完成了函数: `static void early_uart_send(unsigned int c)` 这是一个打印单个字符的函数, 留下来需要填写一个打印字符串的方法 `void uart_send_string(char *str)`

打印字符的接口都留好了, 打印字符串不过就是多了一层遍历:

```c 
void uart_send_string(char *str) {
    /* LAB 1 TODO 3 BEGIN */
	int i;
    for (i = 0; str[i] != '\0'; i++) {
            early_uart_send(str[i]);
    }
    /* LAB 1 TODO 3 END */
}

```

在 init_c 的初始化流程中调用了 uart_send_string, 所以其实把上面的函数填好了, 那么再次运行 qemu 的时候就能看到输出了

### 重新回到汇编

默认情况下 page table, MMU 处于禁用状态, 所以在 init_c 中先后对二者进行使能, 这里使能 MMU 的函数是 `el1_mmu_activate`, 十分遗憾的是这个函数的实现还是在 tools.S 中

这里需要补全的配置是通过设置 SCTLR_EL1 完成对 MMU 的使能

```assembly
BEGIN_FUNC(el1_mmu_activate)
	
	/* other code */
	
	mrs     x8, sctlr_el1
	/* Enable MMU */
	/* LAB 1 TODO 4 BEGIN */
	orr		x8, x8, #SCTLR_EL1_M
	/* LAB 1 TODO 4 END */
	
	/* other code */
	
END_FUNC(el1_mmu_activate)
```

这里使用 x8 保存配置, 通过位运算完成对 MMU 的使能

完成使能后, 再次运行会发生: `Translation Fault`, 毕竟这里仅仅配置了 MMU, 而没有进行页表的配置

>   使用 gdb 进行调试可以看到程序最后卡住在 0x200 的位置

### 后日谈

#### kernel.img

通过 make build 生成的 kernel.img 本质上是一个符合 ELF 格式的可执行目标文件, 这个文件格式之前在学习 csapp 的时候见过:

![](https://cdn.jsdelivr.net/gh/SunYuanI/img@latest/img/executable_obj_file_format.png)

>   其中 .text 是代码段, .init 是在执行应用程序之前的代码(bootloader), lab1 中编写的主要就是 .init 的内容

对于 kernel.img 既然就是一个 executable objective file, 符合 ELF 格式, 那么就可以通过 readelf 获取其文件信息

```shell
# -h 表示读取 ELF header
$ readelf -h kernel.img
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           AArch64
  Version:                           0x1
  Entry point address:               0x80000
  Start of program headers:          64 (bytes into file)
  Start of section headers:          270816 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         4
  Size of section headers:           64 (bytes)
  Number of section headers:         15
  Section header string table index: 14
```

这里面可以看到程序的 entry point(程序入口), 这个特殊的数字 0x80000, 如果在前面 gdb 调试的时候有印象, _start 函数就是从这个位置开始的, [后面](#0x80000 和 ffffff0000000000)会说明这个地址是如何确认的

```shell 
# -S 表示读取 section header table -> 解析得到 section headers
$ readelf -S kernel.img
There are 15 section headers, starting at offset 0x421e0:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] init              PROGBITS         0000000000080000  00010000
       000000000000cfa0  0000000000000000 WAX       0     0     4096
  [ 2] .text             PROGBITS         ffffff0000090000  00020000
       0000000000001928  0000000000000000  AX       0     0     8
  [ 3] .got              PROGBITS         ffffff00000a0000  00030000
       0000000000000028  0000000000000008  WA       0     0     8
  [ 4] .got.plt          PROGBITS         ffffff00000a0028  00030028
       0000000000000018  0000000000000008  WA       0     0     8
  [ 5] .rodata           PROGBITS         ffffff00000b0000  00040000
       000000000000003f  0000000000000000   A       0     0     8
  [ 6] .bss              NOBITS           ffffff00000b0040  0004003f
       0000000000008000  0000000000000000  WA       0     0     16
  [ 7] .debug_line       PROGBITS         0000000000000000  0004003f
       000000000000067a  0000000000000000           0     0     1
  [ 8] .debug_info       PROGBITS         0000000000000000  000406b9
       0000000000000719  0000000000000000           0     0     1
  [ 9] .debug_abbrev     PROGBITS         0000000000000000  00040dd2
       0000000000000303  0000000000000000           0     0     1
  [10] .debug_aranges    PROGBITS         0000000000000000  000410e0
       00000000000000f0  0000000000000000           0     0     16
  [11] .debug_str        PROGBITS         0000000000000000  000411d0
       00000000000002c5  0000000000000001  MS       0     0     1
  [12] .symtab           SYMTAB           0000000000000000  00041498
       0000000000000960  0000000000000018          13    60     8
  [13] .strtab           STRTAB           0000000000000000  00041df8
       0000000000000364  0000000000000000           0     0     1
  [14] .shstrtab         STRTAB           0000000000000000  0004215c
       0000000000000081  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```

可以看到每个 section 的名称(name), 加载到的虚拟地址(address), 当前在文件中偏移量(offset)

ELF 被加载(load), 执行(execute)是两个很关键的过程:

*   load: 按照每个段的加载内存地址 (Load Memory Address，LMA) 将其**从硬盘拷贝**到内存上指定的地址处
*   execute: 按照每个段的虚拟内存地址 (Virtual Memory Address，VMA) 将其**从内存里拷贝或映射**到指定的内存地址处

通过通过反编译指令可以看到每个段的 LMA 和 VMA

```shell
# 因为 kernel.img 是可以在 aarch64 上直接运行的 ELF 文件, 所以在 x86 平台上的 objdump 是无法直接反编译的
# -h 参数 表示反编译每个 section header
$ aarch64-linux-gnu-objdump -h kernel.img 

kernel.img:     file format elf64-littleaarch64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 init          0000cfa0  0000000000080000  0000000000080000  00010000  2**12
                  CONTENTS, ALLOC, LOAD, CODE
  1 .text         00001928  ffffff0000090000  0000000000090000  00020000  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .got          00000028  ffffff00000a0000  00000000000a0000  00030000  2**3
                  CONTENTS, ALLOC, LOAD, DATA
  3 .got.plt      00000018  ffffff00000a0028  00000000000a0028  00030028  2**3
                  CONTENTS, ALLOC, LOAD, DATA
  4 .rodata       0000003f  ffffff00000b0000  00000000000b0000  00040000  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .bss          00008000  ffffff00000b0040  00000000000b0040  0004003f  2**4
                  ALLOC
  6 .debug_line   0000067a  0000000000000000  0000000000000000  0004003f  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
  7 .debug_info   00000719  0000000000000000  0000000000000000  000406b9  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
  8 .debug_abbrev 00000303  0000000000000000  0000000000000000  00040dd2  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
  9 .debug_aranges 000000f0  0000000000000000  0000000000000000  000410e0  2**4
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 10 .debug_str    000002c5  0000000000000000  0000000000000000  000411d0  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
```

除了 .init 段之外每个段的偏移量都是 ffffff0000000000, 为什么都是这个偏移量, 具体原因可以看[后面](#0x80000 和 ffffff0000000000)

>   .debug* 的段先不管, debug 用的, 一看就是编译的时候添加 -g 参数才生成的 

#### 0x80000 和 ffffff0000000000

chcore 的实现中使用了 linker script 用来控制每个 section 在内存中的精确位置, 具体的在 chcore-v2 中这个 script 放在了 `kernel/arch/aarch64/boot/linker.tpl.ld` 中

linker.tpl.ld 引用了 `image.h`, 定义了各种常量, 其中就有:

*   `#define TEXT_OFFSET 0x80000`
*   `#define KERNEL_VADDR 0xffffff0000000000`

可以看到 .init 段 (bootloader) 在最开头, 且其在内存中的地址是通过 TEXT_OFFSET 定义的, 而 .init 的内容是通过 ${init_objects} 定义的

\${init_object} 可以在 `kernel/arch/aarch64/boot/raspi3` 的 CMakeLists.txt 中找到, 进去之后可以看到 ${init_object} 是通过组合若干源代码得到的:

*   init/start.S(定义了 _start 的地方, lab1 的开始)
*   init/tools.S(可以看成是 helper function, 初始化过程中的各种汇编代码都放在这里了)
*   init/mmu.c(别看叫 mmu, 其实是用来初始化页表的)
*   init/init_c.c(麻烦的汇编后见到的第一个 c 程序)
*   peripherals/uart.c(串口配置的函数, 在 qemu 中配置好串口就可以打印输出了)

linker.tpl.ld 中的其余的字段都是按照顺序一排列的, 此外 .text 段 (.init 后的第一个 section) 并不是严格意义上紧挨着 .init 的, 可以看到其位置在 .init 后还多了一个大小为 KERNEL_VADDR 的偏移量, 这也是除了 .init 之外所有的 section 都有一个偏移量的原因

#### :raised_hand: 再写下去就不礼貌了

在 chcore 加载运行的过程中, 首先启动的是 bootloader, 存储在了 ELF 文件的 .init 段中, 操作系统内核部分的保存在 .text 段中

从整体上看的话, 操作系统也不是一直运行在高地址的, 至少 bootloader 部分都是运行在低地址的

## 内存管理

### 配置内核启动页表

os 内核运行在高地址(0xffff_0000_0000_0000), 而在初始化阶段使用的还是低地址, 因此这里需要对内核页表进行配置, 将虚拟内存中的高地址映射到物理地址

在 chcore 中 Kernel 地址从 0xffff_ff00_0000_0000 开始, 这里进行地址映射的时候采用了最简单的方式, 即将 0xffff_ff00_0000_0000 + addr 映射到物理地址 addr

因为 chcore 需要运行在树莓派上, 因此这里的物理内存映射需要符合对应的硬件要求, 更为具体的, 需要配置到物理地址为 0x0000_0000 到 0x8000_0000 的映射

其中 0x0000_0000 到 0x3f00_0000 需要映射为 normal memory, 作为物理内存存在, 0x3f00_0000 到 0x4000_0000 需要映射为 device momory, 作为共享外设内存存在

从 0x0000_0000 到 0x4000_0000 的内存映射粒度为 2 MB

>   当 page size 为 4 KB 时, EL3 级的一个页表可以覆盖大小为 2 MB 的地址空间, 同理 EL2 级的一个页表可以覆盖大小为 1GB 的地址空间
>
>   因此映射粒度为 2 MB, 就等价于让 EL2 中的一个 PTE 映射到一个 block, 而当映射粒度为 1 GB 时, 等价让 EL1 的一个 PTE 映射到一个 block

而从 0x4000_0000 到 0x8000_0000 也需要映射为 device memory, 作为每个核心的自由的本地外设内存, 这部分的内存映射粒度为 1 GB

因为用户地址空间和内核地址空间应该是相互独立的, 这里在进行地址映射的时候, chcore 分别为低地址和高地址做了映射, 保证不管是用户程序还是内核都可以对物理内存和外设进行正常访问

在 lab2 中需要实现的是内核高地址页表对物理内存的映射:

```c
// mmu.c

void init_boot_pt(void) {
    // other code
    
    /* TTBR1_EL1 0-1G */
        /* LAB 2 TODO 1 BEGIN */
        /* Step 1: set L0 and L1 page table entry */
        vaddr =  PHYSMEM_START + KERNEL_VADDR;

        boot_ttbr1_l0[GET_L0_INDEX(vaddr)] = ((u64)boot_ttbr1_l1) | IS_TABLE
                                                            | IS_VALID | NG;

        boot_ttbr1_l1[GET_L1_INDEX(vaddr)] = ((u64)boot_ttbr1_l2) | IS_TABLE
                                                            | IS_VALID | NG;
        /* Step 2: map PHYSMEM_START ~ PERIPHERAL_BASE with 2MB granularity */
        for (; vaddr < PERIPHERAL_BASE + KERNEL_VADDR; vaddr += SIZE_2M) {
                boot_ttbr1_l2[GET_L2_INDEX(vaddr)] = 
                (vaddr - KERNEL_VADDR)
                | UXN
                | ACCESSED
                | NG
                | NORMAL_MEMORY
                | IS_VALID;
        }

        /* Step 2: map PERIPHERAL_BASE ~ PHYSMEM_END with 2MB granularity */
        for (; vaddr < PHYSMEM_END + KERNEL_VADDR; vaddr += SIZE_2M) {
                boot_ttbr1_l2[GET_L2_INDEX(vaddr)] = 
                (vaddr - KERNEL_VADDR)
                | UXN
                | ACCESSED
                | NG
                | DEVICE_MEMORY
                | IS_VALID;
        }
        /* LAB 2 TODO 1 END */
    
    // other code
}

```

>   init_boot_pt 函数中首先实现的是低地址到物理内存的映射, 以此作为参考很容易实现高地址到物理内存的映射

### 物理内存管理

完成 init_c.c 中的 init_boot_pt() 后页表就完成粗粒度的初始化, 在经过 lab 1 中的 mmu 使能后, 调用 start_kernel 跳转到高地址并进入内核, 从此 bootloader 结束

在内核的 main.c 中, 只进行了两个初始化, 一个是 uart (串口), 一个是 mm (memory), 前者在 bootloader 阶段就已经完成了一部分初始化, 而后者实现的就是对物理内存的管理

chcore 使用了 buddy system 对所有 PP (physical page) 的管理, 更为具体的, 在 main.c 中调用了 mm_init() 完成 buddy system 的初始化

>   配置页表的时候将 0x0000_0000 到 0x3f00_0000 映射为物理内存, 这里使用 buddy system 管理的就是这部分内存

buddy system 管理的 block 具有不同的阶, 一个具有 n 阶的 block, 包含了 $2^\text{n}$ 个 page, 特别的在 chcore 中一个 page 大小为 4 KB

chcore 的实现中 buddy system 中包含了两部分, 头部为用于管理 buddy system 的 metadata, 后面紧跟的是实际用来分配给程序使用的内存

的抽象为 phys_mem_pool 
