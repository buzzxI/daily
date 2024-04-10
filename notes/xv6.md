# xv6

当前版本的 xv6 基于 risc-v, mit 给出了 xv6 的[手册](https://pdos.csail.mit.edu/6.828/2022/xv6/book-riscv-rev3.pdf)，除此之外还可以查询 risc-v 的[指令集手册](https://github.com/riscv/riscv-isa-manual)

在实现 xv6 的各个 lab 的时候一定要好好看 lab 的要求和提示, 有大作用

## util

这个 lab 需要实现一些小工具, 并不涉及到 xv6 的 kernel, 需要编写用户程序, 调用 xv6 提供的 syscall 完成一系列功能

完成这个 lab 之前, 官方建议先看一下 xv6 手册的第一章:

*   xv6 的 shell 是一个 user program (不是 syscall), 默认的 shell 很简单, 就是一个主循环不断从 stdin 读取输入, 同时对于每个读入的 command, 通过 syscall -> fork 一个子进程处理 command; 如果当前 command 需要执行另外的进程, 子进程会执行另外的 syscall -> exec 处理; 在此期间父进程 (shell 本身) 会 wait 子进程的执行结束

    >   默认情况下, xv6 shell 也支持 background jobs, 此时子进程会再次 fork 一个 "孙"子进程, 而子进程会直接结束, 将 job 交给 "孙"子进程处理

*   xv6 使用 file descriptor 表示文件, 所有进程默认启用了三个 fd: stdin(0), stdout(1), stderr(2); 执行 syscall -> fork 会将父进程的地址空间复制一份给子进程, 其中也包括了父进程了 file descriptor table; 执行 syscall -> exec 会对子进程的地址空间进行覆盖, 但保留子进程的 file descriptor table; 这种特性可以使得在 fork 得到子进程后, exec 加载新程序之前, 子进程可以对 fd 进行重定向, 最简单的例子就是重定向 stdin, 使得 stdin 不再来源于 console, 而是来自于某个文件, 对于行为 `cat < input.txt`, 可以表示为:

    ```c
    char *argv[2];
    argv[0] = "cat";
    argv[1] = 0;
    
    if(fork() == 0) {
        close(0);
        open("input.txt", O_RDONLY);
        exec("cat", argv);
    }
    ```

    xv6 的另外一个特点是, 默认通过 open 返回的 fd 一定是在 file descriptor table 中未使用过的 fd 中最小的那个; 在上面是 snippet 中, open 返回的一定是 0, 此时相当于将 stdin(0) 重定向到了 input.txt;

    >   这里 xv6 对 syscall fork 和 exec 的设计进行了解释, 从实现的角度上, 确实可以将 fork 和 exec 绑定为一个 forkexec 的 syscall, 一步完成子进程的加载执行; 但是这种 syscall 会增加 shell 的实现重定向的难度, 作为父进程的 shell 需要在 forkexec 之前主动重定向自身的 stdin, stdout, stderr; 如果分为两个步骤, 那么 shell 就可以让 fork 得到的子进程负责 IO 重定向, 此时子进程对于 file descriptor table 的修改不会影响父进程, 将重定向的目标由父进程转向为子进程

    在 xv6 中父子进程共享的不止有 file descriptor, 还有 file position, 即父进程对文件的修改, 会使得子进程的 file position 发生变化

    ```c
    if(fork() == 0) {
        write(1, "hello ", 6);
        exit(0);
    } else {
        wait(0);
        write(1, "world\n", 6);
    }
    ```

    在上述 snippet 执行结束后, 文件中将被写入 hello world; 在 xv6 中, dup 获得的 fd 和原 fd 之间也是共享的 file position 的, 通过其中一个 fd 对文件进行的修改, 会导致另外一个 fd 对应的 file position 发生变化

    >   不知道这种设计是故意的还是只是 legacy

*   xv6 也支持 pipe, xv6 原生提供了 syscall -> pipe, 其需要一个大小为 2 的 int 数组作为参数, 完成调用后 p[0] 用来从 pipe 中读取数据, p[1] 用来向 pipe 中写入数据 (类比 stdin 和 stdout, 这两个分别是 0 和 1)

    在 unix 中使用 pipe 可以实现进程之间的通信, 借助 pipe 和 dup, 可以在不修改原程序的情况下, 让一个程序的输出作为另一个程序的输入 -> xv6 的 shell (xv6 支持级联的 pipeline: [command a] | [command b] | [command c] | ... 具体原因在于其在进行 cmdline 解析的时候, 以左侧优先的原则, 同时支持解析 cmdline 的递归调用)

    从执行的结果上来看 pipe 在作用等价于临时文件

    ```shell
    # pipe
    $ echo 'hello pipe' | wc
    # redirect
    $ echo 'hello pipe' > /tmp/log; wc < /tmp/log
    ```

    上面的 pipe command 等价于先将 echo 的结果保存在临时的 log 文件中, 然后再调用 wc 统计字符

    但是如果使用 redirect 实现相同功能, 在完成调用后, 还需要进一步将临时文件清理掉; 此外临时文件是具有大小限制的, 反正肯定受到磁盘本身大小的限制；而最难接受的是, redirect 完全将 pipe 分为了两个阶段, 考虑 pipe 嵌套的情况, redirect 还会进一步进行拆分, 使得每个阶段是相互独立, 前一个阶段必须完成后, 才能执行后一个阶段, 从整个程序的运行角度考虑, 并发性能很差

*   xv6 直接通过绝对路径和相对路径定位文件, 提供了 syscall -> chdir 可以修改 current directory; 

    文件名可能相同, 但 xv6 在底层是通过 inode 定位各个文件的, 每个 inode 可以有多个 file name, 在 xv6 中将其称为 link; 本质上, link 就是 directory 中的一个 entry, 可以粗略的将 directory 看成是一个 map, 以 file name 为 key, inode 为 value

    在 xv6 中每个 inode 中保存了文件的 metadata: type (file/directory/device), length, location of file content on disk, number of links; xv6 可以通过 syscall -> fstat 获取 inode 的 metadata

    由于 xv6 使用 inode 记录文件, 因此只有当 inode 被删除后, 文件才会被真正删除, 在 os 层面, 只有当 inode 的 link 数量为 0, 并且没有 file descriptor 指向该 inode, 才会将该 inode 删除

    基于这个机制, 可以创建和进程生命周期相同的临时文件, 比如:

    ```c 
    fd = open("/tmp/xyz", O_CREATE|O_RDWR);
    unlink("/tmp/xyz");
    ```

    文件 /tmp/xyz 被创建后, 返回了一个 file descriptor, 但马上就通过 syscall -> unlink 将对应 inode 的 link 置为 0, 这样, 只要当前进程将 file descriptor close 掉之后, 该文件就会自动被 os 删除, 对应的资源会被直接回收  

### sleep

这里需要实现的 sleep, 单位是 tick, 即一个时钟周期, 所以在不同的机器上一个 tick 对应的实际物理时间可能是不太一样的

>   xv6 启用了定时中断并在每个时钟周期都会触发, 对于 kernel 而言, 很容易完成 tick 的计数

根据 xv6 的习惯, 所有的用户程序放在了 `/user` 下, 这里将实现放在了 `/user/sleep.c` 中

>   lab 发布的网站给出了实现的各种提示, 一定要先看

#### implementation

```c
// sleep.c
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char** argv) {
    if (argc != 2) {
        fprintf(2, "Usage: %s [ticks]\n", argv[0]);        
        exit(1);
    }

    int ticks = atoi(argv[1]);

    if (sleep(ticks) < 0) {
        fprintf(2, "sleep error\n");
        exit(1);
    }
	// xv6 要求程序通过 syscall -> exit 结束, 后面看到 kernel 的 exec 的实现的时候就知道为什么了
    exit(0);
}
```

xv6 提供了一个完备的 makefile, 因为这里为 xv6 添加了一个用户程序, 因此需要修改 makefile, 这里在 `UPROGS` 中添加上 sleep 即可, 具体的写法和之前的用户程序保持一致

```makefile
# makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_sleep\
```

最后 `make qemu` 完成构建并进入 qemu 启动 xv6, 即可手动进行 sleep 的测试, 不过 xv6 本身编写了测试脚本

```shell
$ make qemu					# 这个命令会进行 xv6 的编译, 并在 qemu 中运行 xv6
$ ./grade-lab-util sleep	# 这个命令会使用 xv6 的测试脚本对 sleep 进行测试, 具体的参数可以直接看 grade-lab-util 的实现, 就是一个 python 脚本
```

#### track

为什么调用了 sleep 就直接进入 kernel 执行 syscall 了呢

xv6 在进行编译的时候保留了很多副产物, 比如编译得到的汇编文件:

```assembly
; sleep.asm
user/_sleep:     file format elf64-littleriscv


Disassembly of section .text:

0000000000000000 <main>:
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char** argv) {
   0:	1141                	addi	sp,sp,-16
   2:	e406                	sd	ra,8(sp)
   4:	e022                	sd	s0,0(sp)
   6:	0800                	addi	s0,sp,16
    if (argc != 2) {
   8:	4789                	li	a5,2
   a:	02f50163          	beq	a0,a5,2c <main+0x2c>
        fprintf(2, "Usage: %s [ticks]\n", argv[0]);        
   e:	6190                	ld	a2,0(a1)
  10:	00001597          	auipc	a1,0x1
  14:	81058593          	addi	a1,a1,-2032 # 820 <malloc+0xf2>
  18:	4509                	li	a0,2
  1a:	00000097          	auipc	ra,0x0
  1e:	628080e7          	jalr	1576(ra) # 642 <fprintf>
        exit(1);
  22:	4505                	li	a0,1
  24:	00000097          	auipc	ra,0x0
  28:	2d4080e7          	jalr	724(ra) # 2f8 <exit>
    }

    int ticks = atoi(argv[1]);
  2c:	6588                	ld	a0,8(a1)
  2e:	00000097          	auipc	ra,0x0
  32:	1ca080e7          	jalr	458(ra) # 1f8 <atoi>

    if (sleep(ticks) < 0) {
  36:	00000097          	auipc	ra,0x0
  3a:	352080e7          	jalr	850(ra) # 388 <sleep>
  3e:	00054763          	bltz	a0,4c <main+0x4c>
        fprintf(2, "sleep error\n");
        exit(1);
    }

    exit(0);
  42:	4501                	li	a0,0
  44:	00000097          	auipc	ra,0x0
  48:	2b4080e7          	jalr	692(ra) # 2f8 <exit>
        fprintf(2, "sleep error\n");
  4c:	00000597          	auipc	a1,0x0
  50:	7ec58593          	addi	a1,a1,2028 # 838 <malloc+0x10a>
  54:	4509                	li	a0,2
  56:	00000097          	auipc	ra,0x0
  5a:	5ec080e7          	jalr	1516(ra) # 642 <fprintf>
        exit(1);
  5e:	4505                	li	a0,1
  60:	00000097          	auipc	ra,0x0
  64:	298080e7          	jalr	664(ra) # 2f8 <exit>
```

可以看到调用 sleep 时对应了 `jalr	850(ra) # 388 <sleep>`, 即跳转到 `0x388` 处

```assembly
; sleep.asm

0000000000000388 <sleep>:
.global sleep
sleep:
 li a7, SYS_sleep
 388:	48b5                	li	a7,13
 ecall
 38a:	00000073          	ecall
 ret
 38e:	8082                	ret
```

`0x388` 处将 13 放入了寄存器 a7 中, 并调用 ecall, 查询手册知道 ecall 是一条 privilege instruction, 在 user mode 下调用会触发 exception, 从而进入 kernel 被 kernel 的 exception handler 处理

符号 `SYS_sleep` 被定义为 13 (syscall.h), handler 因此可以知道触发当前 exception 是因为 ecall, 并且此时寄存器 `a7` 为 13, 从而知道了 user-application 需要执行 sleep

kernel handler 处理 sleep 的时候调用了 sys_sleep, xv6 对于 sleep 的实现很简单, 每次调用 sys_sleep 都会记录当前 ticks(ticks0), 并让 xv6 调用其他进程, 当 xv6 重新调度到当前进程的时候再次比较 ticks0 和 ticks(全局变量, 每次发生 timer interrupt 的时候都会自增), 如果二者之差小于 n, 则重复上面的过程 

```c
// sysproc.c
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;

  argint(0, &n);
  acquire(&tickslock);
  ticks0 = ticks;
  while(ticks - ticks0 < n){
    if(killed(myproc())){
      release(&tickslock);
      return -1;
    }
    sleep(&ticks, &tickslock);
  }
  release(&tickslock);
  return 0;
}
```

>   这里可以看到 sys_sleep 调用了 sleep, 这里和前面用户程序的 sleep 已经不是同一个了, 这里的 sleep 定义在 kernel 中, user-application 是无法直接访问的
>
>   这里可以粗略认为 sleep 就是进行一次进程调度即可

sleep 可以保证休眠实现不少于 n ticks

### pingpong

要求通过父子进程之间通过 pipe 完成通信, 这里需要使用 syscall -> pipe() 创建一个用于在父子进程之间通信的 pipe, 默认的参数为一个大小为 2 的数组(pipefd[2]), 其中 pipefd[0] 是读取端, pipefd[1] 是写入端

pipe() 本身的特性决定了创建的一个 pipefd 是具有方向的, 即一个 pipefd[2] 不可以用来实现父子进程之间的双向通信

>   考虑一个简单的 parent -> child -> parent 的例子, 当 parent 完成写入后, 需要等待 child 的写回, 所以按理来说应该阻塞在 pipe 的 read 端
>
>   但因为一个 pipe 是不区分父子进程的, 此时会出现子进程和父进程同时 read 的情况, 因此 parent 可能 read 到自己刚刚写入的内容, 而子进程可能无法读取 

在本 lab 的要求下, 肯定是需要创建两个 pipe 的

```c
#include "kernel/types.h"
#include "user/user.h"

int main(int arc, char** argv) {
    // pipes for parent and child, pipes[0] is used for reading, pipes[1] is used for writing
    int p_2_c[2];
    int c_2_p[2];

    if (pipe(p_2_c) < 0 || pipe(c_2_p)) {
        fprintf(2, "create:pipe:error\n");
        exit(1);
    }

    int pid;
    
    if ((pid = fork()) < 0) {
        fprintf(2, "fork:child:error\n");
        exit(1);
    }

    char one_byte;

    if (pid == 0) {
        // child process
        close(p_2_c[1]);
        close(c_2_p[0]);

        if (read(p_2_c[0], &one_byte, 1) == 1) {
            fprintf(1, "%d: receive ping\n", getpid());
            if (write(c_2_p[1], &one_byte, 1) == 1) {
                // write success
            }
        }

        close(p_2_c[0]);
        close(c_2_p[1]);

    } else {
        close(p_2_c[0]);
        close(c_2_p[1]);
        one_byte = 42;

        if (write(p_2_c[1], &one_byte, 1) == 1) {
            if (read(c_2_p[0], &one_byte, 1) == 1) {
                // read success
                fprintf(1, "%d: receive pong\n", getpid());
            } 
        }
        
        close(p_2_c[1]);
        close(c_2_p[0]);
    }

    exit(0);
}
```

>   read 和 write 的返回值为实际 read 和 write 的字节数, 在 linux 中可能出现 short count

### prime

要求实现一个素数筛, 只不过需要使用多进程配合实现, 不过这里并不需设计这样一个算法, 这里已经给出了 [Bell Labs and CSP Threads (swtch.com)](https://swtch.com/~rsc/thread/)

使用的 api 和上面的 pingpong 没有什么区别, 只是需要将上面的算法编码一下

由于 xv6 的资源并没有那么充裕, 而在上面的算法下, 每个素数都需要对应一组 pipefd 和一个进程, 因此 lab 只要求打印出 35 以内的所有素数

```c
#include "kernel/types.h"
#include "user/user.h"

void process_routine(int p_2_c) {
    int prime;
    read(p_2_c, &prime, 1);

    if (prime > 0) {
        fprintf(1, "prime %d\n", prime);
        int pipefd[2];
        if (pipe(pipefd) >= 0) {
            int pid;
            if ((pid = fork()) >= 0) {
                if (pid == 0) {
                    close(pipefd[1]);
                    process_routine(pipefd[0]);
                    close(pipefd[0]);
                    exit(0);
                }
                else {
                    close(pipefd[0]);
                    for (int num = 0;;) {
                        read(p_2_c, &num, 1);
                        if (num == 0 || num % prime != 0) write(pipefd[1], &num, 1);
                        if (num == 0) break;
                    }
                    close(pipefd[1]);
                    int status;
                    wait(&status);
                }
            }
        }
    }
}

int main(int argc, char** argv) {
    int pipefd[2];
    
    if (pipe(pipefd) < 0) {
        fprintf(2, "pipe:error\n");
        exit(1);
    }

    int pid;

    if ((pid = fork()) < 0) {
        fprintf(2, "fork:error\n");
        exit(1);
    }

    if (pid == 0) {
        close(pipefd[1]);
        process_routine(pipefd[0]);
        close(pipefd[0]);
    } else {
        close(pipefd[0]);
        int i;
        for (i = 2; i <= 35; i++) write(pipefd[1], &i, 1);
        i = 0; 
        write(pipefd[1], &i, 1);
        close(pipefd[1]);
     	// wait to reap child process
        int status;
        wait(&status);
    }

    exit(0);
}
```

注意到一定要让最顶层的父进程回收所有的子进程之后再退出, 否则一旦父进程结束, 在 xv6 中会直接打印 prompt($)

### find

find all the files in a directory tree with a specific name. 

xv6 要求实现的 find 版本和 unix 并不相同, 对于示例给出的输入:

```shell
$ tree
.
├── a
│   ├── aa
│   │   └── b
│   └── b
└── b
$ find . b
./b
./a/b
./a/aa/b
```

在 xv6 的语法环境中 `find` 有两个参数, 第一个参数为 path 表示起始目录(文件), 第二个参数为 pattern 表示需要匹配的文件名

这个 lab 的 hint 说明了, 可以学习 `ls` 中遍历目录的方式, 查找所有文件, 所以代码结构和 `ls` 是很类似的

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

const static char* current_dir = ".";
const static char* parent_dir = "..";

void find(char* path, char* pattern) {
    int fd;
    struct stat st;

    if ((fd = open(path, 0)) < 0) {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if (fstat(fd, &st) < 0) {
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    char buf[512];
    strcpy(buf, path);
    char* p = buf + strlen(buf);
    switch (st.type) {
        case T_FILE:
            *p = 0;
            while (p > buf && *p != '/') p--;
            p++;
            if (strcmp(p, pattern) == 0) printf("%s\n", path);
            break;
        case T_DIR:
            *p++ = '/';
            struct dirent de;
            while (read(fd, &de, sizeof(de)) == sizeof(de)) {
                // jump empty directory entry
                if (de.inum == 0) continue;
                // jump current directory and parent directory
                if (strcmp(de.name, current_dir) == 0) continue;
                if (strcmp(de.name, parent_dir) == 0) continue;
                int len = strlen(de.name);
                memmove(p, de.name, len);
                p[len] = 0;
                // recurse find in sub-dir
                find(buf, pattern);
            }
            break;
    }

    close(fd);
}

int main(int argc, char** argv) {
    if (argc != 3) {
        fprintf(2, "Usage: %s [path] [pattern]", argv[0]);
        exit(1);
    }
    
    find(argv[1], argv[2]);

    exit(0);
}
```

>   这里的代码还有改进空间, 上面的 find 并不会处理目录, 即不会列出目录名和 pattern 相同的情况

### xargs

xv6 中的 xargs 会将前一条指令的输出作为当前指令的参数执行:

```shell
$ [command1] | xargs [command2]
```

将 command1 运行的结果作为 command2 的参数并执行, xargs 允许多个参数, xargs 会为 command1 输出的每行结果执行一次 xargs

```shell 
$ tree
.
├── a
│   └── b
├── b
└── c
    └── b
$ find . b | xargs grep hello
```

在上面的目录结构下, xargs 会为查询到的三个文件 b 分别执行一次 grep hello

xargs 本身也是 unix 中一个常用的工具, 因此这里的写法上和前面的 ls 保持一致

```c
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/param.h"

void xargs(int argc, char** argv) {
    char* args[MAXARG];
    int idx = 0;
    for (int i = 1; i < argc; i++, idx++) args[idx] = argv[i];
    char buff[512];
    int end = 0;
    while (read(0, &buff[end], 1) == 1) {
        if (buff[end] == '\n') {
            buff[end] = 0;
            args[idx] = buff;
            args[idx + 1] = 0;
            int pid;
            if ((pid = fork()) == 0) {
                exec(args[0], args);
                // never reach
                exit(0);
            }
            else {
                int status;
                wait(&status);
            }
            end = 0;
        } else end++;
    }    
}

int main(int argc, char** argv) {
    if (argc < 2) {
        fprintf(2, "Usage: %s [command]\n", argv[0]);
        exit(1);
    }
    xargs(argc, argv);
    exit(0);
}
```

## syscall

### xv6 trap

xv6 将 syscall, exception(illegal operations), interrupt(中断) 统称为 trap, 在 xv6 中 trap 的调用机制为: uservec() -> usertrap() -> usertrapret() -> userret()

>   在 csapp 中 trap 特指 syscall, 而使用 exception 作为统称. 无论怎么叫, 只需要知道当应用程序遇到了 syscall, exception, interrupt 后程序 control flow 发生改变, 处理器不会立刻执行下一条指令, 而是跳转到 exception handler, 而最终返回到程序的**下一条指令, 当前指令或者终止程序**

trap handler 进行异常处理的时候不会修改应用程序本身的状态 (除非是非法操作导致的程序终止), 因此 xv6 在进行异常处理的前需要保存应用程序状态, 并在返回的时候恢复应用程序的状态信息

>   这里所谓的状态信息其实就是应用程序进入异常时, 应用程序寄存器和内存的值

risc-v 提供了专门用于处理异常的 privilege register:

*   stvec: trap handler address, 发生异常后跳转到 stvec 对应的地址

*   sepc: store pc, 异常处理结束后的返回地址, 在进入 trap handler 时, 取值为用户应用程序的 pc

    >   在 trap handler 中可能对 sepc 进行修改, 比如 syscall 的返回地址应该是当前指令的下一条指令, 比如缺页异常的返回地址应该是当前指令

*   scause: reason for trap(number), 一个整数表示, 引发 trap 的原因

    >   比如 syscall 的 scause 为 8

*   sscratch: buffer, 字如其名, 本身没有什么特殊含义, 更多是用于临时存储, 后面说到 trapframe 的时候会再次提到

*   sstatus: 可以认为是控制寄存器, 其对状态的控制通过 bit mode 决定, 这里主要考虑 SIE 和 SPP bit

    *   SIE: whether enable device interrupt, 将 SIE 置 0 后 risc-v 不再接收硬件中断
    *   SPP: 用来区分来自 user mode 和 supervisor mode 的 trap, SPP 为 0 表示 trap 是在 user mode 下产生的, SPP 为 1 表示 trap 是在 supervisor mode 下产生的

>   上述寄存器使用 s 作为前缀, 在 risc-v 中表示, 前缀为 s 的寄存器只能在 supervisor mode 下访问到, 前缀为 m 的寄存器只能在 machine mode 下访问到

*   stval: 当 load/store page fault 时, stval 为导致 fault 发生的地址

### user space trap

user application 通过 ecall 引起 trap, 此时程序会跳转到寄存器 stvec 指向的地址, risc-v 不会在发生 trap 时自动完成 user process page table 和 kernel page table 的切换, 直接将 trap handler 放在 user mode 是不太合适的, 因此 xv6 提供了一个 trampoline page, 该 page 下完成了 page table 的切换

对于 os 而言, trap handler 相对于 user application 应该是透明的, 即在执行 trap handler 之前, 需要保存当前进程的上下文信息, 以便在 handler 返回的时候可以回到正确的位置(发生 trap 的地方, 或者发生 trap 的下一条指令, 具体的取决于 trap 的类型), xv6 将寄存器信息保存在了 trapframe 中

进程的 trampoline page 和 trapframe page 被映射到了地址空间的最高处:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/31/11:35:00:xv6_process_virtual_address_space_layout.png)

在 xv6 中, stvec 指向的地址为 uservec, 因为涉及到保存进程上下文(寄存器状态)和页表切换, 因此这部分是使用汇编实现的:

```assembly
# trampoline.S
.globl uservec
uservec:    
	#
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #

        # save user a0 in sscratch so
        # a0 can be used to get at TRAPFRAME.
        csrw sscratch, a0

        # each process has a separate p->trapframe memory area,
        # but it's mapped to the same virtual address
        # (TRAPFRAME) in every process's user page table.
        li a0, TRAPFRAME
        
        # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
        sd t1, 80(a0)
        sd t2, 88(a0)
        sd s0, 96(a0)
        sd s1, 104(a0)
        sd a1, 120(a0)
        sd a2, 128(a0)
        sd a3, 136(a0)
        sd a4, 144(a0)
        sd a5, 152(a0)
        sd a6, 160(a0)
        sd a7, 168(a0)
        sd s2, 176(a0)
        sd s3, 184(a0)
        sd s4, 192(a0)
        sd s5, 200(a0)
        sd s6, 208(a0)
        sd s7, 216(a0)
        sd s8, 224(a0)
        sd s9, 232(a0)
        sd s10, 240(a0)
        sd s11, 248(a0)
        sd t3, 256(a0)
        sd t4, 264(a0)
        sd t5, 272(a0)
        sd t6, 280(a0)

	# save the user a0 in p->trapframe->a0
        csrr t0, sscratch
        sd t0, 112(a0)

        # initialize kernel stack pointer, from p->trapframe->kernel_sp
        ld sp, 8(a0)

        # make tp hold the current hartid, from p->trapframe->kernel_hartid
        ld tp, 32(a0)

        # load the address of usertrap(), from p->trapframe->kernel_trap
        ld t0, 16(a0)


        # fetch the kernel page table address, from p->trapframe->kernel_satp.
        ld t1, 0(a0)

        # wait for any previous memory operations to complete, so that
        # they use the user page table.
        sfence.vma zero, zero

        # install the kernel page table.
        csrw satp, t1

        # flush now-stale user entries from the TLB.
        sfence.vma zero, zero

        # jump to usertrap(), which does not return
        jr t0
```

>   trampoline.S 的代码的主要目的是实现寄存器的保存, 而因为对于应用程序而言, 为了将所有寄存器的值保存在 trapframe 中, 首先需要将 trapframe 的地址保存到某个寄存器中, 这里使用的是寄存器 a0
>
>   而言应用程序可能正在使用寄存器 a0, 如果直接进行覆盖将导致用户数据的丢失, 因此这里用到了 buffer register -> sscratch, 先将 a0 的值保存在 sscratch 中, 然后将 trapframe 的地址保存在 a0 中, 再通过 a0 保存各个寄存器的值, 最后再将 a0 的值保存下来

xv6 使用 trapframe page 保存当前进程的上下文, 在完成页表切换后, 才会跳转到真正的 trap handler -> usertrap()

>   因此 trampoline page 是一个特殊的 page, 其同时保存在了 kernel page table 和 user application page table 中

```c
// trap.c
void usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    setkilled(p);
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

usertrap() 根据 trap 的类型进行处理, 寄存器 sacuse 保存了造成 trap 的原因, 可以通过默认的分支看出来, scause 为 8 时表示发生了 syscall

>   更多的 scause 需要查询 risc-v privilege instruction 手册

可以看到的时当发生 syscall 后, 程序的返回值应该时造成 syscall 发生的下一条指令, 因此在 syscall (scause 为 8) 的情况下, 手动让其 trapframe 的 epc (返回地址) + 4

usertrap 的结尾调用了 usertrapret(), usertrapret() 看上去有点奇怪, usertrapret() 首先保存了部分信息, 比如将函数 uservec 的地址保存在了寄存器 stvec 中, 将 kernel page table 的位置, kernel stack 的位置, trap handler 的地址保存在了当前进程的 trapframe 中

这些信息看起来并不像是每次进入 kernel 都会变化的, 在 os 启动之后就不会再次修改, 主要原因在后面解释

随后 usertrapret 设置好了 sepc, 调用 userret, 将应用程序的上下文信息写回寄存器

```assembly
userret:
        # userret(pagetable)
        # called by usertrapret() in trap.c to
        # switch from kernel to user.
        # a0: user page table, for satp.

        # switch to the user page table.
        sfence.vma zero, zero
        csrw satp, a0
        sfence.vma zero, zero

        li a0, TRAPFRAME

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld gp, 56(a0)
        ld tp, 64(a0)
        ld t0, 72(a0)
        ld t1, 80(a0)
        ld t2, 88(a0)
        ld s0, 96(a0)
        ld s1, 104(a0)
        ld a1, 120(a0)
        ld a2, 128(a0)
        ld a3, 136(a0)
        ld a4, 144(a0)
        ld a5, 152(a0)
        ld a6, 160(a0)
        ld a7, 168(a0)
        ld s2, 176(a0)
        ld s3, 184(a0)
        ld s4, 192(a0)
        ld s5, 200(a0)
        ld s6, 208(a0)
        ld s7, 216(a0)
        ld s8, 224(a0)
        ld s9, 232(a0)
        ld s10, 240(a0)
        ld s11, 248(a0)
        ld t3, 256(a0)
        ld t4, 264(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)

	# restore user a0
        ld a0, 112(a0)
        
        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret
```

sret 的返回地址就是之前使用 sepc 保存的地址

### first process

这里考虑的是 xv6 中的第一个进程是如何启动的

#### entry

xv6 的入口是 `/kernel/entry.S`, 在编译的时候对 linker 进行配置, 保证 entry.S 会加载到 0x80000000

>   xv6 通过 memory-mapping 的方式将硬件的寄存器映射到地址范围: `0x0 ~ 0x80000000`, 当 xv6 操作这部分内存的时候就是在对硬件寄存器进行配置

这部分代码的功能是初始化函数栈(起始就是设置寄存器 sp), 用来执行后续的 c 程序

```assembly
# entry.S
		# qemu -kernel loads the kernel at 0x80000000
        # and causes each hart (i.e. CPU) to jump there.
        # kernel.ld causes the following code to
        # be placed at 0x80000000.
.section .text
.global _entry
_entry:
        # set up a stack for C.
        # stack0 is declared in start.c,
        # with a 4096-byte stack per CPU.
        # sp = stack0 + (hartid * 4096)
        la sp, stack0
        li a0, 1024*4
        csrr a1, mhartid
        addi a1, a1, 1
        mul a0, a0, a1
        add sp, sp, a0
        # jump to start() in start.c
        call start
spin:
        j spin

```

>   `stack0` 定义在了 start.c 中:
>
>   ```c
>   // start.c
>   // entry.S needs one stack per CPU.
>   __attribute__ ((aligned (16))) char stack0[4096 * NCPU];
>   ```
>
>   这里的 NCPU 定义在了 param.h 中, xv6 最多支持 8 个 CPU

栈从高地址向低地址生长, hartid 为 0 的核心在运行了 entry.S 后, 栈空间范围为 `stack0 ~ stack0 + 4K`, 初始化空间如下:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/29/17:26:37:xv6_bootloader_function_stack.svg)

随后通过 `call` 指令跳转到 `start.c` 执行第一个 c 程序 `start()`, 尽管这里是 c 函数, 但是都是通过内联汇编的形式配置 risc-v 的 privileged register

```c
// start.c
// entry.S jumps here in machine mode on stack0.
void start()
{
  // set M Previous Privilege mode to Supervisor, for mret.
  unsigned long x = r_mstatus();
  x &= ~MSTATUS_MPP_MASK;
  x |= MSTATUS_MPP_S;
  w_mstatus(x);

  // set M Exception Program Counter to main, for mret.
  // requires gcc -mcmodel=medany
  w_mepc((uint64)main);

  // disable paging for now.
  w_satp(0);

  // delegate all interrupts and exceptions to supervisor mode.
  w_medeleg(0xffff);
  w_mideleg(0xffff);
  w_sie(r_sie() | SIE_SEIE | SIE_STIE | SIE_SSIE);

  // configure Physical Memory Protection to give supervisor mode
  // access to all of physical memory.
  w_pmpaddr0(0x3fffffffffffffull);
  w_pmpcfg0(0xf);

  // ask for clock interrupts.
  timerinit();

  // keep each CPU's hartid in its tp register, for cpuid().
  int id = r_mhartid();
  w_tp(id);

  // switch to supervisor mode and jump to main().
  asm volatile("mret");
}
```

机器刚刚启动的时候处理器处于 machine mode, 而 kernel 需要运行在 supervisor mode, 在完成了若干配置之后 xv6 通过 `mret` 进入 supervisor mode

>   `mret` 表示从 machine mode 返回到先前的异常级别, 其返回地址保存在 `mepc` 中, 因为刚刚开机, 默认处于 machine mode, 因此 main.c 手动修改寄存器 `mstatus`, 将之前的异常级别设置为 supervisor mode

start.c 修改了寄存器 `mepc`, 在 xv6 中 epc 表示当异常级别从高优先级降低到低优先级时的返回地址, 在这里设置了进入 supervisor mode 后跳转到 main() (kernel/main.c)

>   发生 trap 后, 进入 supervisor mode, trap handler 处理结束后设置了寄存器 sepc, 设置的是从 supervisor mode 回到 user mode 的返回地址

上面的各种操作仅仅涉及对寄存器的修改, 每个处理器核心拥有自己的处理器核心, 因此不需要考虑并发问题, 但在进行 kernel 初始化的时候, 会对 kernel 的在内存中的数据结构进行各种初始化操作, 此时涉及到操作内存, 就需要考虑并发问题了, 因此在 xv6 (chcore 也是一样的) 中仅使用单颗核心用来进行初始化操作, 而让其他核心自旋 (hartid 为 0 的核心初始化 kernel, 其余核心自旋)

>   这里值得一提的是, 因为 kernel 尚未完成初始化, 当然 xv6 的 spinlock 也是处于未初始化状态, 因此这里的自旋锁是一个 volatile 类型的变量, 所有被"阻塞"的核心会不断读取这个变量, 在变量为 0 的时候自旋

main 进行了各种初始化操作, userinit() 创建了被 xv6 管理的第一个进程, scheduler() 调度了这个进程 (就是让其运行起来)

```c
// main.c
// start() jumps here in supervisor mode on all CPUs.
void main()
{
  if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode table
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();        
}
```

#### userinit()

```c
// proc.c
// Set up first user process.
void userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy initcode's instructions
  // and data into it.
  uvmfirst(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}
```

在 kernel 中将进程抽象为 `struct proc`, userinit() 初始化了这个结构体, 值得关注的是, 在 allocproc() 中, 将这个进程初始状态下的寄存器 `ra` 设置地址为 forkret, 进程 p 运行的起点即为 forkret

userinit() 创建的进程运行的代码不是通过 exec 加载的, 这部分代码逻辑写在了 `initcode.S` 中, xv6 将其编译得到二进制码, 存放在 `proc.c` 的数组 `initcode` 中, xv6 强制将其映射到 virtual address 中地址为 0 的地方

#### scheduler()

```c
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();

    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
      }
      release(&p->lock);
    }
  }
}
```

从形式上来看, scheduler() 整体的结构就是一个 for 循环, 在 for 循环内部并没有明显的 break 语句, 看起来进入 scheduler 之后就不会返回了, 但 scheduler 的真正核心在于函数 swtch(), 正是这个函数完成了进程上下文的切换

```assembly
# swtch.S
.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret
```

包含两个 `struct context` 类型的参数, 将当前寄存器的状态保存在第一个参数中, 将第二个参数的值赋给寄存器

因为第一个参数为 `cpu->context`, 第二个参数为 `proc->context`, 在调用了 swtch() 后, 将当前进程的上下文保存在 `cpu->context` 中, 同时将 `proc->context` 换入, 因此函数的返回值不是 scheduler() 的下一条指令, proc 的寄存器 ra 保存的地址

因此 main() 函数调用 scheduler() 结束后, 不会返回, 而是执行 forkret

>   当发生定时器中断或者 user application 调用 syscall 的时候, 可能发生进程调度, 其调度方式也是调用函数 swtch()
>
>   ```c
>   // proc.c
>   // Switch to scheduler.  Must hold only p->lock
>   // and have changed proc->state. Saves and restores
>   // intena because intena is a property of this
>   // kernel thread, not this CPU. It should
>   // be proc->intena and proc->noff, but that would
>   // break in the few places where a lock is held but
>   // there's no process.
>   void sched(void)
>   {
>    int intena;
>    struct proc *p = myproc();
>   
>    if(!holding(&p->lock))
>      panic("sched p->lock");
>    if(mycpu()->noff != 1)
>      panic("sched locks");
>    if(p->state == RUNNING)
>      panic("sched running");
>    if(intr_get())
>      panic("sched interruptible");
>   
>    intena = mycpu()->intena;
>    swtch(&p->context, &mycpu()->context);
>    mycpu()->intena = intena;
>   }
>   ```
>
>   swtch 换出进程 p 的上下文, 换入 cpu 的上下文, 因为程序运行的起点就是 scheduler(), 因此 cpu 上下文换入的结果就是进入 scheduler 的 for 循环内部, 所以 xv6 的进程调度是很简单的, 只不过是循环遍历整个 proc 数组

#### forkret

```c
// proc.c
void forkret(void)
{
  static int first = 1;

  // Still holding p->lock from scheduler.
  release(&myproc()->lock);

  if (first) {
    // File system initialization must be run in the context of a
    // regular process (e.g., because it calls sleep), and thus cannot
    // be run from main().
    first = 0;
    fsinit(ROOTDEV);
  }

  usertrapret();
}
```

static 类型的 first 仅仅会在第一次调用 forkret 时起作用, 整个 forkret 看起来就是 usertrapret() 的 wrapper function, xv6 运行程序都是从 trap handler 的 "后一半" 开始的

还记得上面提到过了, 看到 usertrapret() 的时候很奇怪, 而看到这里就知道了, usertrapret() 开头的保存寄存器的操作其实是服务于初始化进程的, 比如为即将运行的进程设置 trap handler vector, 为了可能存在的 trap handler 保存 kernel page table, kernel stack, 这部分信息都保存在进程 virtual address 的 trapframe 中

trap handler 返回时异常级别从 supervisor mode 降为 user mode, 在 allocproc() 中设置了寄存器 epc 的值为 0, 因此随后执行的就是 initcode 的地址为 0 处的代码

#### initcode

initcode.S 仅仅执行了一个 syscall -> exec, 其执行的文件为 `/init`

```assembly
# initcode.S
#include "syscall.h"

# exec(init, argv)
.globl start
start:
        la a0, init
        la a1, argv
        li a7, SYS_exec
        ecall

# for(;;) exit();
exit:
        li a7, SYS_exit
        ecall
        jal exit

# char init[] = "/init\0";
init:
  .string "/init\0"

# char *argv[] = { init, 0 };
.p2align 2
argv:
  .long init
  .long 0
```

initcode 存在的目的就是为了执行 `/init`, 这个程序写在了 `/user/init.c` 中, 从 initcode 转变为 init, 也表示了从 kernel 进入了 user application

#### init

```c
// user/init.c
char *argv[] = { "sh", 0 };

int main(void)
{
  int pid, wpid;

  if(open("console", O_RDWR) < 0){
    mknod("console", CONSOLE, 0);
    open("console", O_RDWR);
  }
  dup(0);  // stdout
  dup(0);  // stderr

  for(;;){
    printf("init: starting sh\n");
    pid = fork();
    if(pid < 0){
      printf("init: fork failed\n");
      exit(1);
    }
    if(pid == 0){
      exec("sh", argv);
      printf("init: exec sh failed\n");
      exit(1);
    }

    for(;;){
      // this call to wait() returns if the shell exits,
      // or if a parentless process exits.
      wpid = wait((int *) 0);
      if(wpid == pid){
        // the shell exited; restart it.
        break;
      } else if(wpid < 0){
        printf("init: wait returned an error\n");
        exit(1);
      } else {
        // it was a parentless process; do nothing.
      }
    }
  }
}
```

init.c 整体结构是一个死循环, 循环内部通过 fork + exec 的组合运行 shell 程序, 当 shell 程序因为异常退出后, 下一次循环会再次运行一个 shell

至此 xv6 完成启动

### use gdb

这部分就是使用 gdb 调试 xv6 在启动时候的状态:

#### Q1: Looking at the backtrace output, which function called `syscall`?

前面说 xv6 启动时的各个操作, 可以知道第一个 syscall 发生在 initcode 运行时, 调用 syscall -> exec 加载并运行 init

```shell
(gdb) bt 
#0  syscall () at kernel/syscall.c:133
#1  0x0000000080001d14 in usertrap () at kernel/trap.c:67
#2  0x0505050505050505 in ?? ()
```

从 gdb 的打印结果来看, syscall 是在 usertrap() 中调用的

#### Q2: What is the value of `p->trapframe->a7` and what does that value represent? (Hint: look `user/initcode.S`, the first user program xv6 starts.)

initcode.S 将 SYS_exec 保存在了 a7 中, 表示了当前 syscall 的编号, xv6 内部是通过一个数组保存各个 syscall 的实现的, 因此这里的 a7 其实是用来索引的, 实际调试的时候得到 a7 的值为 7

```c
// syscall.c
// An array mapping syscall numbers from syscall.h
// to the function that handles the system call.
static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
};
```

如果查询 xv6 中保存的 syscalls 数组会发现, "碰巧" sys_exec 的索引就是 7

#### Q3: What was the previous mode that the CPU was in?

前面提到过, 寄存器 sstatus 是控制寄存器, 保存了各种配置, 其 SPP bit 是用来区分在进入 trap 之前的异常级别的, 这里打印 sstatus 得到的结果为 0x22, 不管在 32 位还是 64 位下, SPP 都是 sstatus 的第 8 个 bit, 因此在进入 trap 之前程序位于 user mode (SPP 为 0)

#### Q4: Write down the assembly instruction the kernel is panicing at. Which register corresponds to the varialable `num`?

这里要求将 num 的值强制写为地址 0 处的值, 这里涉及到一次读取地址 0 处变量的值, 因为地址 0 在 kernel page table 并不是可读的, 因此一定会产生异常, xv6 对于这种非法操作会使用调用 panic, 并打印状态寄存器的值, 包括寄存器 sepc 的值, 即为造成 panic 的指令的地址

这里要求根据 kernel 编译得到的中间代码 kernel.asm, 找到导致 kernel panic 的指令, 这里的指令就是访问 0 地址的指令:

```assembly
# kernel.asm
// num = p->trapframe->a7;
  num = *(int*)0;
    80001ff4:	00002683          	lw	a3,0(zero) # 0 <_entry-0x80000000>
```

其中保存了 num 的寄存器为 a3

#### Q5: Why does the kernel crash? Hint: look at figure 3-3 in the text; is address 0 mapped in the kernel address space? Is that confirmed by the value in `scause` above?

原因就不解释了, 上面已经说过了, 查询 risc-v 的 privilege instruction 手册, 可以找到 scause 为 0xd (13) 时, 对应的错误类型为: load page fault, 即在读取对应页面的时候出错

#### Q6: What is the name of the binary that was running when the kernel paniced? What is its process id (`pid`)?

第一个 syscall 是 initcode 中调用的 exec 触发的, 因为它是整个 xv6 的第一个进程, 因此 pid 为 1

### system call tracing

要求添加一个 syscall -> sys_trace(), 接收一个参数 mask, 用来跟踪打印 mask 对应的 syscall

这里的 mask 通过 bitmode 的方式指定, 在这个 lab 中 xv6 原生只有 21 个 syscall, 因为使用 int 类型的 bitmode 就可以覆盖所有的 syscall

根据 lab 的要求, trace 的 mask 属于某个进程, xv6 允许修改 struct proc, 这样 sys_trace 的工作就很简单了, 直接将入参 mask 赋给对应的进程即可

当进程调用 `syscall.c` 中的函数后, 返回前, 添加对当前 syscall number 的判断, 当其匹配到进程的 syscall_mask 时添加打印

lab 要求当前进程 fork 得到的子进程会继承得到父进程的 syscall_mask, 因此这里还需要修改 sys_fork

```c
// sysproc.c
uint64
sys_trace(void) {
  int mask;
  
  argint(0, &mask);

  struct proc* p = myproc();

  p->syscall_mask = mask;

  return 0;
}

// proc.c
// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  // added for lab: syscall, child inherits parent syscall mask
  np->syscall_mask = p->syscall_mask;

  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;

  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  release(&np->lock);

  acquire(&wait_lock);
  np->parent = p;
  release(&wait_lock);

  acquire(&np->lock);
  np->state = RUNNABLE;
  release(&np->lock);

  return pid;
}

// syscall.c
// An array mapping syscall numbers from syscall.h
// to the function that handles the system call.
static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
[SYS_trace]   sys_trace,  // added for lab: syscall
};

// added for lab: syscall
const static char* syscall_name[] = {
  [SYS_fork]    "fork",
  [SYS_exit]    "exit",
  [SYS_wait]    "wait",
  [SYS_pipe]    "pipe",
  [SYS_read]    "read",
  [SYS_kill]    "kill",
  [SYS_exec]    "exec",
  [SYS_fstat]   "fstat",
  [SYS_chdir]   "chdir",
  [SYS_dup]     "dup",
  [SYS_getpid]  "getpid",
  [SYS_sbrk]    "sbrk",
  [SYS_sleep]   "sleep",
  [SYS_uptime]  "uptime",
  [SYS_open]    "open",
  [SYS_write]   "write",
  [SYS_mknod]   "mknod",
  [SYS_unlink]  "unlink",
  [SYS_link]    "link",
  [SYS_mkdir]   "mkdir",
  [SYS_close]   "close",
  [SYS_trace]   "trace",
};

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();
    if (p->syscall_mask & (1 << num)) printf("%d: syscall %s -> %d\n", p->pid, syscall_name[num], p->trapframe->a0);
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

xv6 限定了 syscall 的打印格式, 因此这里额外使用了一个常量数组保存字面量, 使用和索引 syscalls 数组的相同的方式索引 syscall_name 数组

xv6 定义的用户程序放在了 /user/trace.c 中, 默认情况下不参与编译, 因此需要修改 makefile, 同时修改 user.h, usys.pl, 额外声明一个 syscall -> trace, 这部分在 lab 的 hint 部分写的很详细

>   xv6 的测试例中使用了 fork 子进程, 如果机器性能太差 (比如 1 core 2g :cry:) 的话可能过过不了测试用例, 这里将超时时间修改到 50 s 

### sysinfo

要求实现一个 syscall -> sysinfo, 用来获取当前系统中的进程个数和可用内存大小, xv6 默认并没有直接提供获取进程个数和内存的接口, lab 要求在 proc.c 和 kalloc.c 中分别添加获取进程个数和可用内存大小的接口

```c
// proc.c
// added for lab: syscall
uint64 available_process() {
  uint64 rst = 0;
  for (int i = 0; i < NPROC; i++) {
    acquire(&proc[i].lock);
    if (proc[i].state != UNUSED) rst++;
    release(&proc[i].lock);
  } 
  return rst;
}

// kalloc.c
// added for lab: syscall
uint64 available_mem() {
  uint64 rst = 0;
  acquire(&kmem.lock);
  struct run* r = kmem.freelist;
  for (; r; r = r->next) rst += PGSIZE;
  release(&kmem.lock);
  return rst;
}

```

>   xv6 kernel 使用 free list 进行内存管理, 管理单位为一个 page (4 KB), page 之间通过链表的形式串联起来, 因此在获取可用内存大小的时候直接遍历链表就行

xv6 在进行进程创建和内存分配的时候会加锁, 比如在 allocproc() 中分配进程的时候, 需要获取每个进程的锁, 在 kalloc() 分配内存的时候需要获取 kernel memory 的锁

这里采用了类似的加锁方式, 但就算不加锁也可以通过测试例

sysinfo 的参数是一个 sysinfo 类型的结构体指针, 因此获取参数的时候需要使用 argaddr()

kernel 在 trap handler 中处理 syscall 的时候使用的是 kernel page table, 而不是 user page table, 因此当用户程序传入了一个指针, 并 kernel 并不能直接填写这个指针, 用户程序提供的 struct sysinfo* 指向的地址在 kernel 中大概率为非法地址 (xv6 一般会把用户程序的代码段放在低地址, 对于 kernel 而言低地址被映射到了外设)

因此这里实现 sysinfo 的时候需要使用 kernel 中的结构体保存结果, 然后调用 copyout 将其写入用户程序的页表中

```c
// sysproc.c
uint64
sys_sysinfo(void) {
  uint64 addr;
  argaddr(0, &addr);

  struct sysinfo info;
  info.freemem = available_mem();
  info.nproc = available_process();

  struct proc *p = myproc();
  return copyout(p->pagetable, addr, (char*)&info, sizeof(info)); 
}
```

剩下的操作和之前的 syscall 基本相同, 需要在 syscall.c 中添加函数声明, 修改 makefile, user.h, usys.pl

## page table

xv6 的虚拟地址有 64 bit, 其中仅低 39 bit 用来寻址, 由于 xv6 使用大小为 4 KB 的页, 从而在 39 bit 中低 12 bit 作为页偏移, 高 27 bit 用作页表索引, 因此每个进程可以使用的虚拟地址空间大小被限制为 $2^{39} = 512\text{GB}$

>   事实上 risc-v 可以支持 48 bit 的虚拟地址, 此时使用类似 aarch64 的 4 级页表, 单个进程可以使用的内存大小增加到 $256 \text{TB}$

xv6 使用三级页表, 每级页表使用 9 bit 索引 (正好对应了 27 bit), 从而每级页表可以容纳的 PTE 有 $2^9 = 512$ 个, 可以粗略认为每个 PTE 包含了一个 64 bit 的地址, 因此每个页表大小为 $2^9 * 2^3 = 2^{12} = 4\text{KB}$ 大

在 risc-v 中 64 bit 地址仅使用了低 56 bit 进行物理寻址, 对于 risc-v 处理器而言, 其寻址的空间范围为 0 ~ $2^{56} - 1$, 从而使得 risc-v 可以支持从物理内存最大为 $2^{56} = 64 \text{PB}$

>   这些示例的物理内存不仅仅作为 DRAM 存在, 接入的外设也需要占据地址空间的一部分

因此 PTE 的高 10 bit 在 xv6 中不具有任何作用, 而低 10 bit 作为页表的 flag 存在 (控制作用), 从而每个 PTE 可以用作查询物理地址的部分仅有中间的 44 bit, 为了拼凑得到 56 bit 的物理地址, 在进行查询的时候, 需要补充 12 bit, 对于最低级页表, 补充的 12 bit 为 VPO(virtual page offset), 而其他级页表补充的是 12 bit 0

具体的地址翻译如图:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/08/02/15:37:21:risc-v_xv6.png)

>   PTE 的 flag bit 表示的权限信息包括了: 当前 PTE 是否有效(PTE_V), 当前 page 是否允许 read/write/execute(PTE_R, PTE_W, PTE_X), 当前 page 是否可以在 user mode 下访问到(PTE_U)

### speed up system calls

当应用程序执行 sycall 的时候需要提高异常级别, 在 kernel mode 下进行 syscall, trap 对于应用程序而言是透明的, 因此在执行 trap handler 之前需要保存当前进程的上下文, 在返回之前需要恢复进程上下文

某些 syscall, 比如 getpid() 的目的仅仅是读取某个值, 此时可以让 kernel 和应用程序 share data, 通过将 share data 映射到用户程序的 read only page, 可以高效, 安全的完成数据共享

在这一 part 中, xv6 需要加速 syscall -> getpid(), xv6 在虚拟地址中额外添加了一个 page -> USYSCALL (就在 trapframe 下面), 为了加速 getpid(), 需要在创建进程时, 将进程的 pid 映射到 USYSCALL 中

```c
// memlayout.h
// User memory layout.
// Address zero first:
//   text
//   original data and bss
//   fixed-size stack
//   expandable heap
//   ...
//   USYSCALL (shared with kernel)
//   TRAPFRAME (p->trapframe, used by the trampoline)
//   TRAMPOLINE (the same page as in the kernel)
#define TRAPFRAME (TRAMPOLINE - PGSIZE)
#ifdef LAB_PGTBL
#define USYSCALL (TRAPFRAME - PGSIZE)

struct usyscall {
  int pid;  // Process ID
};
#endif
```

这里为了保存 usyscalll page 的内容, xv6 允许修改 struct proc, 将 struct usyscall 保存在 proc 中

```c
// proc.h
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  struct usyscall *syscall;     // added for lab: pgtbl
};
```

>   在 proc 中新增了指针 syscall, 其和默认存在的 trapframe 是类似的, 相比之下区别在于, trapframe 不允许被用户程序访问, 而 usyscall page 是允许被访问的 (PTE 的 user access bit 被置位)

这里实现 usyscall page 时完全参考了 trapframe page 的实现

xv6 使用 allocproc() 完成进程分配, 使用 freeproc() 回收进程, 这两个函数中分别进行了 proc->trapframe 的初始化和回收, 因此也需要在这两个函数中完成 proc->syscall 的分配和回收

```c
// proc.c
// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();
  p->state = USED;

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // added for lab: pgtbl
  if ((p->syscall = (struct usyscall *)kalloc()) == 0) {
    freeproc(p);
    release(&p->lock);
  }
  // store process pid into p->syscall
  p->syscall->pid = p->pid;

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}

// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  // added for lab: pgtbl
  if (p->syscall) kfree((void *)p->syscall);
  p->syscall = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```

xv6 使用 proc_pagetable 初始化页表, 使用 proc_freepagetable() 回收页表, 这两个函数中分别进行了 trapframe page 的映射与销毁, 因此也需要在这两个函数中完成 usyscall page 的映射和销毁

```c
// proc.c
// Create a user page table for a given process, with no user memory,
// but with trampoline and trapframe pages.
pagetable_t
proc_pagetable(struct proc *p)
{
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if(pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }

  // map the trapframe page just below the trampoline page, for
  // trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  // added for lab: pgtbl
  // USYSCALL page is read-only for user, this page should be set PTE_R and PTE_U
  if (mappages(pagetable, USYSCALL, PGSIZE, (uint64)(p->syscall), PTE_R | PTE_U) < 0) {
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  return pagetable;
}

// Free a process's page table, and free the
// physical memory it refers to.
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  // added for lab: pgtbl
  uvmunmap(pagetable, USYSCALL, 1, 0);
  uvmfree(pagetable, sz);
}
```

理论上使用 shared page 还可以用来加速 fstat, 当用户程序通过 open 打开文件的时候, 可以将对应文件的 fd 和 stat 保存在 USYSCALL page 中, 调用 close 关闭文件的时候, 将 fd 和 stat 从 USYSCALL page 中释放

### print a page table

xv6 要求实现一个函数可以用来打印 page table, 要求在 vm.c 中添加函数 vmprint

>   xv6 默认实现了 walk 将虚拟地址翻译得到实际的物理地址, 不过这里其实可以参考一下 freewalk 的实现

这种打印很容易递归的方式实现:

```c
// vm.c
// added for lab: pgtbl
static void page_print_recursive(pagetable_t pagetable, char* prefix) {
  char next_prefix[16];
  char* p = prefix;
  int i = 0;
  for (; i < 16 && *p != 0; i++, p++) next_prefix[i] = *p;
  next_prefix[i++] = ' ';
  next_prefix[i++] = '.';
  next_prefix[i++] = '.';
  next_prefix[i] = 0;

  for (int i = 0; i < 512; i++) {
    pte_t pte = pagetable[i];
    if ((pte & PTE_V)) {
      uint64 pa = PTE2PA(pte);
      printf("%s%d: pte %p pa %p\n", prefix, i, pte, pa);
      // current pte points to next level page table
      if ((pte & (PTE_R | PTE_W | PTE_X)) == 0) page_print_recursive((pagetable_t)pa, next_prefix);
    }
  }
}

void vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);
  char prefix[] = "..";
  page_print_recursive(pagetable, prefix);
}

```

>   最麻烦的地方居然是, 如何简单高效的写好前缀

### detect which pages have been accessed

risc-v 的 PTE 的 control bit 中包括了 A bit 和 D bit, 前者表示 PTE 对应的 page 是否被 read/write/fetch (access) 过, 后者表示 PTE 对应的 page 是否被 write 过

这里 xv6 需要借助 access bit, 从而 detect which page has been accessed

这部分需要实现一个 syscall -> sys_pgaccess(), 该 syscall 有三个参数: 第一个参数是一个 virtual address, 表示需要检查的地址, 第二个参数表示了需要检查的页数, 第三个参数用来保存检查的结果

由于 xv6 中最大支持 64 bit 的数据类型, 因此该 syscall, 有一个检查页数的上限 -> 64

syscall 的实现其实是很简单的, 从 user application 提供的 virtual address 开始搜索页表, 获取最后一级 PTE, 查看 access bit 即可

```c 
// sysproc.c
int
sys_pgaccess(void)
{
  // lab pgtbl: your code here.
  uint64 target, rst;
  int page_count;
  argaddr(0, &target);
  argint(1, &page_count);
  argaddr(2, &rst);
  
  // argument illegal
  if (page_count <= 0 || page_count > 64) return -1;

  struct proc *p = myproc();
  pte_t *pte;
  uint64 addr = target; 
  uint64 tmp = 0;

  for (int i = 0; i < page_count; addr += PGSIZE, i++) {
    pte = walk(p->pagetable, addr, 0);
    if (!pte) return -1; 
    if (*pte & PTE_A) {
      tmp |= (1 << i);
      // clear access bit after syscall
      *pte &= (~PTE_A);
    }
  }

  copyout(p->pagetable, rst, (char*)(&tmp), 8);
  return 0;
}
```

需要注意的是, 在 walk through page table 的时候, 将结果保存在了临时变量 tmp 中, 而在最后通过 copyout 才将结果换回 user address space, 这是因为 user address 在 kernel address space 是非法的, kernel 并不能直接访问

>   如果实际 debug 的话会发现这里提供的 user adderss 是一个很小的数, 在 kernel address space 中这个地址对应的将是某些外设

## traps

### RISC-V assembly

这部分不需要修改 xv6 本身的代码, xv6 给出了 call.c, 通过 `make fs.img` 将其编译为 call.asm, 这部分需要查询 risc-v reference manu 回答给出的问题

*   Which registers contain arguments to functions? For example, which register holds 13 in main's call to `printf`?

    ![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/09/03/11:02:21:risc-v_assembly_handbook.png)

    risc-v 使用 a0-a7 保存函数调用参数, 在 call.asm 的 printf 中, 使用寄存器 a2 保存 13

*   Where is the call to function `f` in the assembly code for main? Where is the call to `g`? (Hint: the compiler may inline functions.)

    在 call.asm 的 main 函数中, 尽管 printf 其中的一个参数需要调用 f 得到, 但是编译器将其直接优化了:

    ```assembly
      printf("%d %d\n", f(8)+1, 13);
      24:	4635                	li	a2,13
      26:	45b1                	li	a1,12
    ```

    complier 直接将 f 的调用结果 (12) 保存在寄存器 a1 中

    而在 f 调用 g 的部分, complier 也没有真的进行函数调用:

    ```assembly
    000000000000000e <f>:
    
    int f(int x) {
       e:	1141                	addi	sp,sp,-16
      10:	e422                	sd	s0,8(sp)
      12:	0800                	addi	s0,sp,16
      return g(x);
    }
      14:	250d                	addiw	a0,a0,3
      16:	6422                	ld	s0,8(sp)
      18:	0141                	addi	sp,sp,16
      1a:	8082                	ret
    ```

    由于函数调用的第一个参数会保存在 a0 中, 而函数返回的结果会保存在 a0 中, 而调用 g 不过是让参数自增 3, 因此 complier 这里直接将其优化为让寄存器 a0 自增 3

*   At what address is the function `printf` located?

    ```assembly
    000000000000001c <main>:
    
    void main(void) {
      1c:	1141                	addi	sp,sp,-16
      1e:	e406                	sd	ra,8(sp)
      20:	e022                	sd	s0,0(sp)
      22:	0800                	addi	s0,sp,16
      printf("%d %d\n", f(8)+1, 13);
      24:	4635                	li	a2,13
      26:	45b1                	li	a1,12
      28:	00000517          	auipc	a0,0x0
      2c:	7c850513          	addi	a0,a0,1992 # 7f0 <malloc+0xe8>
      30:	00000097          	auipc	ra,0x0
      34:	61a080e7          	jalr	1562(ra) # 64a <printf>
      exit(0);
      38:	4501                	li	a0,0
      3a:	00000097          	auipc	ra,0x0
      3e:	298080e7          	jalr	664(ra) # 2d2 <exit>
    ```

    在 main 函数中, 通过 jalr 进行函数调用, 跳转到 0x64a 处的 printf 函数

*   What value is in the register `ra` just after the `jalr` to `printf` in `main`?

    ```assembly
    auipc rd, imm
    ```

    指令 auipc 可以让 pc 的值自增 imm, 并将结果保存在寄存器 rd 中, 因此在调用 printf 之前, ra 的值和 pc 相同 (0x30), 执行 printf 之前, 指令将 ra 的值保存在了 printf 的栈帧中, 因此从 printf 返回后, ra 的仍为 0x30

*   Run the following code.

    ```c
    unsigned int i = 0x00646c72;
    printf("H%x Wo%s", 57616, &i);
    ```

    What is the output? [Here's an ASCII table](https://www.asciitable.com/) that maps bytes to characters.

    The output depends on that fact that the RISC-V is little-endian. If the RISC-V were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616` to a different value?

    [Here's a description of little- and big-endian](http://www.webopedia.com/TERM/b/big_endian.html) and [a more whimsical description](https://www.rfc-editor.org/ien/ien137.txt).

    输出结果为: He110 World#, 前一个字符很好解释, 57616 对应的 16 进制为 0xe110, 打印后一个字符使用 %s, 对应地址为 &i, 因此打印出的字符为 0x00646c72 为对应 ASCII 表中的字符

    由于默认情况下, risc-v 使用小端法保存数据, 因此打印的字符的 ASCII 码的顺序为 0x72, 0x6c, 0x64, 0x00, 因此结果为: rld(null)

    如果 risc-v 使用大端法保存数据, 那么在打印字符时, 需要将 i 修改为 0x726c6400, 而 57616 不需要修改, 因为不管大小端法, 打印的数字都应该正常显示

*   In the following code, what is going to be printed after `'y='`? (note: the answer is not a specific value.) Why does this happen?

    ```c
    printf("x=%d y=%d", 3);
    ```

    将其编译得到 .asm 文件:

    ```assembly
    printf("x=%d y=%d", 3);
      38:	458d                	li	a1,3
      3a:	00000517          	auipc	a0,0x0
      3e:	7ce50513          	addi	a0,a0,1998 # 808 <malloc+0xee>
      42:	00000097          	auipc	ra,0x0
      46:	61a080e7          	jalr	1562(ra) # 65c <printf>
    ```

    调用 printf 时, 只会将 3 保存在 a1 中, 而 printf 包括了两个 place holder, 因此打印的结果并不是确定的, 多次打印得到的结果不同

### backtrace

在修改 xv6 kernel 时很容易改出 bug, 这里要求实现一个 backtrace, 用来打印当前的调用栈, risc-v 的栈结构可以参考: [RISC-V calling convention, stack frames, and gdb](https://pdos.csail.mit.edu/6.1810/2022/lec/l-riscv.txt)

```asciiarmor
                   .				<---- top of stack
                   .
      +->          .
      |   +-----------------+   |
      |   | return address  |   |
      |   |   previous fp ------+
      |   | saved registers |
      |   | local variables |
      |   |       ...       | <-+
      |   +-----------------+   |
      |   | return address  |   |
      +------ previous fp   |   |
          | saved registers |   |
          | local variables |   |
      +-> |       ...       |   |
      |   +-----------------+   |
      |   | return address  |   |
      |   |   previous fp ------+
      |   | saved registers |
      |   | local variables |
      |   |       ...       | <-+
      |   +-----------------+   |
      |   | return address  |   |
      +------ previous fp   |   |
          | saved registers |   |
          | local variables |   |
  $fp --> |       ...       |   |
          +-----------------+   |
          | return address  |   |
          |   previous fp ------+
          | saved registers |
  $sp --> | local variables |
          +-----------------+ 		<---- bottom of stack
```

在 risc-v 中进行函数调用后, 寄存器 fp 指向了前一个栈帧的起始地址, 要说明的是, 返回地址保存在地址偏移 fp - 8 处, 前一个栈帧的地址保存在地址偏移 fp - 16 处

打印函数调用栈，就是打印所有函数返回地址, 借助寄存器 fp 可以找到栈帧中的返回地址和前一个栈帧的地址

默认情况下, 栈顶并不能直接获取, xv6 分配的所有函数栈都仅仅占了一页的大小, 且在 xv6 中, 地址都是按页对齐的, 因此只要知道了寄存器 fp 的值, 就可以知道其所在的 page, 从而推测出栈顶

为了获取寄存器 fp 的值, 需要通过内联汇编的方式读取寄存器的值, xv6 默认并没有提供这个函数, 不过好在 lab 的手册页提供了这个函数

```c
// riscv.h
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```

>   xv6 将所有通过内敛汇编读写控制寄存器的函数都保存在了 riscv.h 中, 这里遵从 xv6 的习惯, 也将这个函数保存在了这个文件中

lab 使用 bttest 测试 backtrace 的功能, 其要求在 sys_sleep (sysproc.c) 中调用 backtrace (kernel/printf.c), 因此需要在 defs.h 中进行函数声明

```c
// printf.c
void backtrace(void) {
  uint64 fp = r_fp();
  // stack is always a single page
  uint64 top_stack = PGROUNDDOWN(fp) + PGSIZE;

  printf("backtrace:\n");
  while (fp < top_stack) {
    uint64 ra = *((uint64*)(fp - 8));
    printf("%p\n", ra);
    fp = *((uint64*)(fp - 16)); 
  }
}
```

在后面进行调试的时候, 可以在 panic 中调用 backtrace, 从而跟踪导致 panic 的指令

仅仅打印地址其实没什么用, 为了方便调试, 需要知道该地址对应的指令, 幸运的是 xv6 会将内核编译为 kernel.asm, 保存了地址和函数指令的关系, 借助 linux 的工具 addr2line 可以将地址翻译为指令

编写了一个小驱动程序, 可以从 backtrace.txt 读取调用地址, 驱动 addr2line 完成地址翻译

```c
// backtrace.c
#include <csapp.h>
#define STACK_SIZE 32

int main(int argc, char** argv) {
    int fd = Open("./backtrace.txt", O_RDONLY, S_IRUSR);
    rio_t rio;
    Rio_readinitb(&rio, fd);
    char* args[STACK_SIZE] = {"/usr/bin/addr2line", "-e", "kernel/kernel"};
    char buff[MAXBUF];
    char* p = buff;
    ssize_t len = 0;
    int idx = 3;
    for (; idx < STACK_SIZE && (len = Rio_readlineb(&rio, p, MAXBUF)); idx++) {
        args[idx] = p;
        p += len;
        if (*(p - 1) == '\n') *(p - 1) = 0;
        else {
            *p = 0;
            p++;
        }
    } 
    args[idx] = NULL;
    Close(fd);
    for (int i = 3; i < idx; i++) {
        fprintf(stdout, "args[%d]:%s\n", i, args[i]);
    }
    pid_t pid;
    if ((pid = Fork()) == 0) {
        Execve(args[0], args, environ);
    }
    while (wait(NULL) != pid) ;
    fprintf(stdout, "reap:addr2line\n");
    return 0;
}
```

>   这里借助了 csapp 的 wrapper function 避免 short count

这样每次将 backtrace() 输出的结果导入 backtrace.txt 中即可完成翻译

### alarm

这里 xv6 需要实现的是 user-level signal handler, 实现两个 syscall: 

*   sigalarm(interval, handler): install a handler, trigger this handler every interval ticks
*   sigreturn(): every handler should call this function before handler returns

handler 和应用程序本身是并发的关系, 因此当控制转移到 handler 后需要保存程序的上下文, 当程序切换回来的时候需要将上下文换回来

alarm 机制相对来讲比较复杂, 实验手册分为两部分实现, 首先让程序通过 test0, 然后通过 test1/test2/test3

首先需要在 xv6 中注册 syscall (user.h, usys.pl, syscall.h, syscall.h, sysproc.c)

在 trap.c 的 usertrap 中, xv6 处理了各种 trap, 当然也包括了定时器中断, 每个时钟周期都会产生一个中断, 从而很容易对某个进程的时钟周期计数

为了记录进程的时钟周期和触发 handler 需要的 tick, 需要在 xv6 的 proc 中额外添加字段:

```c
// proc.h
// added for lab: traps 
struct sig_handler {
  uint64 valid;                // this field is used to valid or invalid sig handler
  uint64 total_ticks;          // used for record total ticks in sigalarm
  uint64 current_ticks;        // used for record current ticks after sigalarm
  uint64 handler;              // used for record user-level handler
  struct trapframe *cache;     // used to cache context, before handler
};

// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  
  // added for lab: traps
  struct sig_handler handler;
  
};
```

handler 包含了 valid 字段, 用来指示当前是否配置了 signal handler, 由于 sigalaram 本身的参数可能存在取为 0 的情况, 因此这里额外添加了 valid 字段表明当前 handler 是否有效 (是否进行 tick 的计数)

handler 中包含了一个 cache 用来缓存定时器中断触发时的程序上下文信息, 写法上遵从 proc 中默认的 trapframe 的写法, 需要在进程创建的时候分配 trapframe, 在进程释放的时候释放对应的 trapframe

```c
// proc.c
// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();
  p->state = USED;

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  // added for lab: traps
  p->handler.valid = 0;
  p->handler.total_ticks = 0;
  p->handler.current_ticks = 0;
  p->handler.handler = 0;
  if ((p->handler.cache = (struct trapframe*)kalloc()) == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  return p;
}

// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  
  // added for lab: traps
  if (p->handler.cache) kfree(p->handler.cache);
  p->handler.cache = 0;
  
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```

>   其实更好的写法是在, sys_sigalarm 中进行 handler cache 的分配

首先实现 sys_sigalarm

```c
// sysproc.c
// added for lab: traps
uint64 sys_sigalarm(void) {
  int ticks;
  uint64 handler;
  
  argint(0, &ticks);
  argaddr(1, &handler);

  struct proc *p = myproc();
  if (ticks == 0) p->handler.valid = 0;
  else {
    p->handler.valid = 1;
    p->handler.total_ticks = ticks;
    p->handler.current_ticks = 0;
    p->handler.handler = handler;
  }

  return 0;
}
```

当计数到达之前的配置的超时时间后, usertrap 处理完定时器事件, 需要返回到 signal handler 的地址

trap 的返回地址由寄存器 epc 决定, 因此这里触发 handler 的方式是直接将返回地址修改为之前配置的 signal handler 的地址

```c
// trap.c
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok

    // added for lab: traps
    if (which_dev == 2) {
      // timer event
      if (p->handler.valid) {
        p->handler.current_ticks++;
        if (p->handler.current_ticks == p->handler.total_ticks) {
          memmove(p->handler.cache, p->trapframe, sizeof(struct trapframe));
          p->trapframe->epc = p->handler.handler; 
          p->handler.valid = 0;
        }
      } 
    }

  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    setkilled(p);
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

signal handler 并不是可重入的, 因此当程序执行 signal handler 时需要将 tick 计数阻塞掉 (将 valid 重置为 0)

随后实现 sigreturn, 由于 sigreturn 需要将程序的上下文从 handler 返回到之前程序中断的地方, 这里直接在 sys_sigreturn 中将之前缓存下来的 trapframe 还原即可

```c
// sysproc.c
uint64 sys_sigreturn(void) {
  struct proc *p = myproc();
  p->handler.valid = 1;
  p->handler.current_ticks = 0;
  memmove(p->trapframe, p->handler.cache, sizeof(struct trapframe));
  return p->trapframe->a0;
}
```

对于普通的 syscall 而言, 返回 0 表示 syscall 执行正常, 然而这里的返回值不能直接写为 0, 这是因为 risc-v 使用寄存器 a0 保存返回值, 如果 sigreturn 返回 0, 会直接用 0 将程序上下文中寄存器 a0 的值覆盖掉, 因此这里为了避免被覆盖, 这里需要将返回值设置为原来 a0 的值

## copy-on-write fork

当父进程 fork 得到子进程时, 并不需要将父进程的地址空间完整复制一份给子进程, 不仅耗时长, 还浪费内存, 如果子进程仅对某些 page 执行 read 操作, 那么对应的 page 完全可以被父子进程共享

copy-on-write fork 在执行 fork 时会让子进程的页表指向父进程页表指向的 physical page, 对于 readable page 是无所谓的, 但对于 writable page 而言, 由于父子进程指向了相同的 physical page, 因此当其中一个进程进行修改时, 另外一个进程可以看到修改, 违背了进程的独立性

在 copy-on-write fork 中, 父进程会清空 PTE 的 writable 标志, 当其中一个进程触发了 store page fault 时, xv6 kernel 会为其分配一个 physical page, 将原先的 page 复制到新的 page, 修改进程页表重新映射到新的 page

不过要注意的是, 当 xv6 处理 store page fault 时, 仅会对那些原来 writable page 重新映射 physical page, 当用户程序试图写一个 readable page, xv6 还是应该直接终止当前进程

所以对于所有触发 store page fault 的 page, 需要区分其原来是否为 writable 的, risc-v 并没有使用页表所有的控制位 (PTE[9] 和 PTE[8] 保留了)

这里定义 PTE[8] 为 PTE_EN_W, 在 fork 时, 当前 page 是 writable 的, 那么对应 page 的 PTE_EN_W 会被置位, 所有 page 的 PTE_W 都会被清空

如果每次遇到 store page fault 时都分配一个新的 physical page, 会导致内存泄漏, 这里 xv6 需要实现一个 "garbage collector", 每当 xv6 分配一个 physical page 或者 fork 一个子进程时, garbage collector 会增加其引用计数, 当程序调用 kfree 释放物理内存时, garbage collector 会减少其引用计数, 当引用为 0 时, physical page 才会被重新添加到 freelist 中

xv6 本身还提供了函数 copyout (vm.c), 可以将值写回用户地址空间, 由于是 kernel 直接修改 physical page, 因此此时并不会触发 store page fault, 因此在 copyout 中需要查询页表获取 PTE, 当 PTE 的 PTE_EN_W 位被置位, 需要分配一个新的 physical page (模拟 store page fault handler)

首先设计一个 "garbage collector": 由于可用物理内存的范围是已知的, 分配的 physical page 是对齐的, 为了记录每个 page 的 reference count, 使用一个数组计数, 数组大小和 physical page 的个数相同

由于 xv6 本身很小, 每个 page 使用一个字节计数即可, 将 physical memory 的结构为: reference array + pages

```c
// kalloc.c
// Physical memory allocator, for user processes,
// kernel stacks, page-table pages,
// and pipe buffers. Allocates whole 4096-byte pages.

#include "types.h"
#include "param.h"
#include "memlayout.h"
#include "spinlock.h"
#include "riscv.h"
#include "defs.h"

void freerange(void *pa_start, void *pa_end);

extern char end[]; // first address after kernel.
                   // defined by kernel.ld.

struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  struct run *freelist;
  // added for lab: cow
  // this physical mem struct is like:
  // reference counts + actual pages
  char *mem_begin;
  char *page_begin;
} kmem;

void
kinit()
{
  initlock(&kmem.lock, "kmem");

  // added for lab: cow
  uint64 pg_cnt = (PHYSTOP - (uint64)end) / (PGSIZE + 1);
  kmem.page_begin = (char*)(PHYSTOP - pg_cnt * PGSIZE);
  kmem.mem_begin = (char*)((uint64)kmem.page_begin - pg_cnt);

  freerange(kmem.page_begin, (void*)PHYSTOP);
}

void
freerange(void *pa_start, void *pa_end)
{
  // modified for lab: cow
  char *p = pa_start;
  for (uint64 i = 0; p + PGSIZE <= (char*)pa_end; p += PGSIZE, i++) {
    kmem.mem_begin[i] = 1; 
    kfree(p);
  }
}

// Free the page of physical memory pointed at by pa,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void
kfree(void *pa)
{
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  acquire(&kmem.lock);

  // added for lab: cow
  uint64 idx = ((uint64)((char*)pa - kmem.page_begin)) / PGSIZE;
  kmem.mem_begin[idx]--;
  if (kmem.mem_begin[idx] < 0) panic("kfree reference error");

  // this page is not be referenced => garbage
  if (kmem.mem_begin[idx] == 0) {
    // Fill with junk to catch dangling refs.
    memset(pa, 1, PGSIZE);
    struct run *r = (struct run*)pa;
    r->next = kmem.freelist;
    kmem.freelist = r;
  }
  
  release(&kmem.lock);
}

// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r) {
    // added for lab: cow
    uint64 idx = ((uint64)((char*)r - kmem.page_begin)) / PGSIZE;
    kmem.mem_begin[idx] = 1;
    kmem.freelist = r->next;
  }
    
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}

// added for lab: cow
// this function is used to increase reference count of physical page
void dup_pa(void *pa) {
  if (((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP) panic("dup_pa");

  acquire(&kmem.lock);
  uint64 idx = ((uint64)((char*)pa - kmem.page_begin)) / PGSIZE;
  kmem.mem_begin[idx]++;
  release(&kmem.lock);
}
```

xv6 使用 fork 得到子进程时, 调用 uvmcopy 将父进程的页表复制给子进程

```c 
// vm.c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    
    // added for lab: cow
    
    // mark PTE_EN_W
    if (*pte & PTE_W) *pte |= PTE_EN_W;
    // do not clear PTE_EN_W bit if PTE_W is not set !!!
    // else *pte &= (~PTE_EN_W);

    // clear PTE_W
    *pte &= (~PTE_W);

    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);

    dup_pa((char*)pa);

    if(mappages(new, i, PGSIZE, pa, flags) != 0){
      kfree((char*)pa);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

默认情况下, xv6 不会处理 store page fault, 直接终止当前进程

```c
// trap.c
// 
// this is an old version of usertrap
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    setkilled(p);
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

xv6 默认仅处理了 scause 为 8 的情况, 查询 risc-v 手册可知, 当出现 store page fault 时, scause 为 15

出现 store page fault 时根据当前 PTE 是否可写 (PTE_EN_W) 选择是否分配 physical page 并重新进行页表映射

```c
// trap.c

// added for lab: cow
// default trap handler => terminate process
static void usertrap_exception_handler(struct proc *p) {
  printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
  printf("            sepc=%p stval=%p\n", p->trapframe->epc, r_stval());
  setkilled(p);
}

// handler for store page fault
static void store_page_fault_handler(struct proc *p) {
  uint64 addr = r_stval();
  
  // check parameter, xv6 may have illegal input
  if (addr >= MAXVA) {
    usertrap_exception_handler(p);
    return;
  }

  pte_t *pte = walk(p->pagetable, addr, 0);

  if (pte == 0 || !(*pte & PTE_EN_W)) {
    usertrap_exception_handler(p);
    return;
  }

  char *mem;
  if ((mem = kalloc()) == 0) {
    usertrap_exception_handler(p);
    return;
  }

  // clear en writeable bit and set writeable bit
  *pte &= (~PTE_EN_W);
  *pte |= (PTE_W);

  uint64 pa = PTE2PA(*pte);
  // copy content
  memmove(mem, (char*)pa, PGSIZE);
  // free old physical page
  kfree((char*)pa);

  // remap pte 
  *pte = PA2PTE((uint64)mem) | PTE_FLAGS(*pte);
}

//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    intr_on();

    syscall();
  } else if (r_scause() == 15) store_page_fault_handler(p);
  else if((which_dev = devintr()) != 0){
    // ok
  } else usertrap_exception_handler(p);

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

最后不要忘了修改 copyout(), 这个函数用来将 kernel space variable 换到 user space 中

```c
// vm.c
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    // modified for lab: cow
    va0 = PGROUNDDOWN(dstva);
    if (va0 >= MAXVA) return -1; 
    pte_t *pte = walk(pagetable, va0, 0);

    if (pte == 0 || !(*pte & PTE_U) || !(*pte & PTE_V)) return -1;

    if (!(*pte & PTE_W)) {
      if (!(*pte & PTE_EN_W)) return -1;

      char *mem;
      if ((mem = kalloc()) == 0) return -1;
      
      // clear en writeable bit and set writeable bit
      *pte |= PTE_W;
      *pte &= (~PTE_EN_W);

      uint64 pa = PTE2PA(*pte);
      // copy content
      memmove(mem, (char*)pa, PGSIZE);
      // free old physical page
      kfree((char*)pa);

      // remap pte
      *pte = PA2PTE((uint64)mem) | PTE_FLAGS(*pte);
    }

    pa0 = PTE2PA(*pte);

    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

从结构上来看, store_page_fault_handler 和 copyout 是类似的, 当 copyout 遇到当前 PTE 不可写的时候, 需要模拟 handler 的行为重新映射一个 physical page

## multi_threading

### uthread: switch between threads

xv6 要求实现一个 user-level 的线程, 其已经实现了一部分框架, 需要补全剩余部分, 这部分需要修改的文件是 uthread.c

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

/* Possible states of a thread: */
#define FREE        0x0
#define RUNNING     0x1
#define RUNNABLE    0x2

#define STACK_SIZE  8192
#define MAX_THREAD  4

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
};
struct thread all_thread[MAX_THREAD];
struct thread *current_thread;
extern void thread_switch(uint64, uint64);
              
void 
thread_init(void)
{
  // main() is thread 0, which will make the first invocation to
  // thread_schedule().  it needs a stack so that the first thread_switch() can
  // save thread 0's state.  thread_schedule() won't run the main thread ever
  // again, because its state is set to RUNNING, and thread_schedule() selects
  // a RUNNABLE thread.
  current_thread = &all_thread[0];
  current_thread->state = RUNNING;
}

void 
thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;
  for(int i = 0; i < MAX_THREAD; i++){
    if(t >= all_thread + MAX_THREAD)
      t = all_thread;
    if(t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0) {
    printf("thread_schedule: no runnable threads\n");
    exit(-1);
  }

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
  } else
    next_thread = 0;
}

void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
}

void 
thread_yield(void)
{
  current_thread->state = RUNNABLE;
  thread_schedule();
}

volatile int a_started, b_started, c_started;
volatile int a_n, b_n, c_n;

void 
thread_a(void)
{
  int i;
  printf("thread_a started\n");
  a_started = 1;
  while(b_started == 0 || c_started == 0)
    thread_yield();
  
  for (i = 0; i < 100; i++) {
    printf("thread_a %d\n", i);
    a_n += 1;
    thread_yield();
  }
  printf("thread_a: exit after %d\n", a_n);

  current_thread->state = FREE;
  thread_schedule();
}

void 
thread_b(void)
{
  int i;
  printf("thread_b started\n");
  b_started = 1;
  while(a_started == 0 || c_started == 0)
    thread_yield();
  
  for (i = 0; i < 100; i++) {
    printf("thread_b %d\n", i);
    b_n += 1;
    thread_yield();
  }
  printf("thread_b: exit after %d\n", b_n);

  current_thread->state = FREE;
  thread_schedule();
}

void 
thread_c(void)
{
  int i;
  printf("thread_c started\n");
  c_started = 1;
  while(a_started == 0 || b_started == 0)
    thread_yield();
  
  for (i = 0; i < 100; i++) {
    printf("thread_c %d\n", i);
    c_n += 1;
    thread_yield();
  }
  printf("thread_c: exit after %d\n", c_n);

  current_thread->state = FREE;
  thread_schedule();
}

int 
main(int argc, char *argv[]) 
{
  a_started = b_started = c_started = 0;
  a_n = b_n = c_n = 0;
  thread_init();
  thread_create(thread_a);
  thread_create(thread_b);
  thread_create(thread_c);
  thread_schedule();
  exit(0);
}

```

uthread 管理的线程都放在了 all_thread 中, 在 main 中调用 thread_init 创建第一个线程, 这里 xv6 认为其为 main 线程(当前线程), 随后调用 thread_create 创建了三个执行用户程序的线程, 最后调用 thread_schedule, 进行一次线程调度

在 xv6 提供的注释中, 给出了提示, 调用位于 uthread_switch.S 中的 thread_switch 完成线程上下文切换

xv6 kernel 调用位于 swtch.S 中的 swtch 进行进程上下文的切换, 其本质就是进行寄存器取值的切换

```assembly
# swtch.S

.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret
```

>   swtch 的函数签名为: `void swtch(struct context*, struct context*)`, 结合汇编代码可知, 其将各个 `callee saved register` 的取值保存在了第一个参数中, 并从第二个参数中读取各个寄存器的值

在进行上下文切换的时候, xv6 kernel 仅仅保存了 `callee saved register`, 这是因为在调用 swtch 之前, 各个 caller saved register 都已经被当前进程保存起来了(要么是在栈帧中, 要么是在堆中)

这里考虑线程上下文切换, 完全参考了进程的切换, xv6 允许在 thread 中添加额外的 field, 这里添加了 context 用来保存各个 callee saved register

```c
// uthread.c
typedef unsigned long uint64;

struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context context;       /* the thread's context */
};
```

线程上下文切换和进程上下文切换完全相同 (直接抄过来)

```assembly
# uthread_switch.S

thread_switch:
	/* YOUR CODE HERE */
	sd ra, 0(a0)
   	sd sp, 8(a0)
   	sd s0, 16(a0)
   	sd s1, 24(a0)
   	sd s2, 32(a0)
   	sd s3, 40(a0)
   	sd s4, 48(a0)
   	sd s5, 56(a0)
   	sd s6, 64(a0)
   	sd s7, 72(a0)
   	sd s8, 80(a0)
   	sd s9, 88(a0)
   	sd s10, 96(a0)
   	sd s11, 104(a0)

   	ld ra, 0(a1)
   	ld sp, 8(a1)
   	ld s0, 16(a1)
   	ld s1, 24(a1)
   	ld s2, 32(a1)
   	ld s3, 40(a1)
   	ld s4, 48(a1)
   	ld s5, 56(a1)
   	ld s6, 64(a1)
   	ld s7, 72(a1)
   	ld s8, 80(a1)
   	ld s9, 88(a1)
   	ld s10, 96(a1)
   	ld s11, 104(a1)
	ret    /* return to ra */
```

在所有保存的寄存器中, 有两个重要寄存器: ra 和 sp 分别表示返回地址保存寄存器和栈指针寄存器, 在 thread_create 中需要将返回地址为配置为的函数地址(输入参数), 并将上下文中的 sp 配置为当前线程的栈

```c
// uthread.c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->context.sp = (uint64)(t->stack + STACK_SIZE);
  t->context.ra = (uint64)func;
}
```

>   栈是从高地址向低地址生长的, 因此配置栈空间的时候需要将上下文中的栈地址配置为高地址

### using threads

这个 lab 的后两个 part 都不需要在 xv6 环境中运行, 而是在普通的 linux 环境中 (需要 link libpthread)

ph.c 其实就是一个 hash table, 在多线程环境中 hash table 的 put 是不安全的, 为了保证当前键值对的修改是对后续修改可见的, 因此需要对 put 加锁

这里使用 mutex 加锁, 在 csapp 中使用 P 和 V 操作进行加锁和释放, xv6 使用 pthread_mutex_lock 和 pthread_mutex_unlock 进行加锁和释放 (更高级的 wrapper function)

为了保证 put 的安全性, 这里可以直接在 put 的开头和结尾加锁/释放锁

```c
// ph.c

pthread_mutex_t table_lock;

static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  pthread_mutex_lock(&table_lock);
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&table_lock);
}

int
main(int argc, char *argv[])
{
  // ...
  // lock need to be initialize before put
  pthread_mutex_init(&table_lock, NULL);
  // ...
}
```

但此时的性能很差, 因为在 put 的开头进行了加锁, 使得同时间只能有一个线程 put, 此外线程获取锁, 释放锁也存在开销, 在这种情况下, 多线程 put 的性能甚至比单线程 put 还差

xv6 给出的解决思路是将 giant lock 拆分 => one lock per bucket

```c
// ph.c

struct entry *table[NBUCKET];
pthread_mutex_t table_lock[NBUCKET];

static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  pthread_mutex_lock(&table_lock[i]);
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&table_lock[i]);
}

int
main(int argc, char *argv[])
{
  // ...
  for (int i = 0; i < NBUCKET; i++) pthread_mutex_init(&table_lock[i], NULL);
  // ...
}
```

### barrier

当前 part 要求实现一个 barrier, 即所有进程都运行到 barrier 后, 各个进程才能运行后续指令

barrier 指令需要借助 pthread 本身的 wait-awake 机制实现

```c
pthread_cond_wait(&cond, &mutex);  // go to sleep on cond, releasing lock mutex, acquiring upon wake up
pthread_cond_broadcast(&cond);     // wake up every thread sleeping on cond
```

调用 pthread_cond_wait 之前需要当前线程持有锁 mutex, 调用后会让当前线程阻塞在 cond, 同时释放锁 mutex, pthread_cond_broadcast 会让唤醒所有阻塞于 cond 的线程, 被唤醒的线程需要再次获取锁 mutex 后才能继续运行

为了校验 barrier 是否起到作用, 这里使用了 round 计数

```c
// barrier.c

static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread++;
  if (bstate.nthread == nthread) {
    bstate.round++;
    bstate.nthread = 0;
    pthread_cond_broadcast(&bstate.barrier_cond);
  }
  else pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

nthread 的作用是用来统计阻塞在 cond 的线程数, 所以当一轮 barrier 结束后需要重置 nthread

## networking

这里要求实现的是一个网卡驱动, xv6 搭建好了整体框架, 包括整体的网络协议栈, qemu 模拟的网卡是 82540EM, xv6 中将网卡抽象为 E1000, 驱动的目的是实现和 qemu 网卡的通信, 因此需要实现两个函数: tx, rx, 分别用来向 E1000 发送数据和从 E1000 接收数据

这里涉及和网卡通信, 因此需要明确 E1000 读写数据的方式, xv6 提供了 82540x 系列的[手册](https://pdos.csail.mit.edu/6.828/2022/readings/8254x_GBe_SDM.pdf), 这个 lab 的难点在于资料实在 "太详细" 了, 不太容易找到重点

qemu 模拟的网卡和 xv6 之间通过 DMA 的方式传输数据, 简单来说, 驱动程序会为网卡配置好一段内存, 以此作为 buffer, 当 xv6 写对应的内存时, 即为 xv6 向网卡写数据, 当 xv6 读对应的内存时, 即为 xv6 从网卡读入数据

此外网卡作为一个独立的外设, 具有自己的寄存器, 可以用来控制网卡的行为, xv6 通过读取 PCI 总线, 获取到网卡, 并将其寄存器映射到 xv6 自身的地址空间中, 这部分地址对应了 xv6 中地址小于 0x80000000 的部分, 当 xv6 kernel 读写这部分内存时, 即修改了网卡的行为

一般而言处理器处理数据的速度快于数据经过网络传输到内存的速率的, 因此 xv6 用来保存数据的 buffer 大小都不只一个数据包的大小, E1000 使用 descriptor 表示一个缓存单元, 并且发送缓存和接收缓存之间是独立的, 在 xv6 中分别使用结构体 tx_desc 和 rx_desc 抽象, 在网卡手册中 3.2.3 描述了 receive descriptors, 3.3.2 介绍了 transmit descriptors

descriptor 本身并不直接包括缓存, 其包括了一个 buffer 字段, 用来指向保存了缓存内容的地址, 此外在这个 lab 中还需要关注 descriptor 的 status 字段, 其表示了网卡对于 descriptor 的使用情况

在 descriptor 的 status 字段中, 存在一个 DD 字段 (descriptor done), 用来表示 E1000 是否已经完成了对 descriptor 的处理

### transmit descriptor

参考手册 3.4 可知, E1000 的寄存器 TDBAL 和 TDBAH 保存了 transmit descriptor 的起始地址, 分别表示地址的低 32 位和高 32 位

>   对于 E1000 而言 transmit descriptor 是一个连续的数组

寄存器 TDH 和 TDT 分别表示了起始 descriptor 的偏移量和结尾 descriptor 的偏移量, 因此最后一个可用的 transmit descriptor 的地址应为 \${TDBAL} + 16 * \${TDT}

>   一个 transmit descriptor 的大小为 16 Bytes

对于 transmit descriptor 而言, xv6 kernel 负责向其中填充数据, 而 E1000 从中获取数据, 因此 transmit descriptor 是否可用等价于 xv6 kernel 是否还能向 buffer 写入数据, 当 transmit descriptor 满了之后, xv6 kernel 不能再向其中写入数据, 初始化的时候, 将 TDH 和 TDT 都初始化为 0, 表示此时 xv6 kernel 没有使用任何的缓存, 因此初始化的时候 xv6 并没有初始化数组 tx_mbufs, 看一下 e1000_transmit 的入参可知, 每次上层都会通过传入一个参数 mbuf 实现数据的传递

### receive descriptor

参考手册 3.2.6 可知, E1000 的寄存器 RDBAL 和 RDBAH 保存了 receive descriptor 的起始地址, 分别表示地址的低 32 位和高 32 位, 类似的还有寄存器 RDH 和 RDT

对于 receive descriptor 而言, xv6 kernel 负责从中获取数据, 而 E1000 向其中写入数据, 在初始化的时候, 将 RDH 初始化为 0, RDT 初始化为最大值, 表示此时 xv6 提供的 receive buffer 没有被写入, 都是可以被 E1000 使用的, 因此初始化的时候 xv6 将整个数组 rx_mbufs 都赋予了初值

### ring

descriptor 并不是使用以此就丢弃的, E1000 认为 descriptor 是一个连续的数组, 通过 TDH(RDT) 和 TDT(RDT) 描述了一个范围, 实际在使用的时候, 将 descriptors 看成一个循环数组

从命名上就能看出来, xv6 分别称其为 tx_ring 和 rx_ring

### solution

hint 部分基本上就是代码的逻辑了, 照着写就行了:

#### transmit

-   First ask the E1000 for the TX ring index at which it's expecting the next packet, by reading the `E1000_TDT` control register.
-   Then check if the the ring is overflowing. If `E1000_TXD_STAT_DD` is not set in the descriptor indexed by `E1000_TDT`, the E1000 hasn't finished the corresponding previous transmission request, so return an error.
-   Otherwise, use `mbuffree()` to free the last mbuf that was transmitted from that descriptor (if there was one).
-   Then fill in the descriptor. `m->head` points to the packet's content in memory, and `m->len` is the packet length. Set the necessary cmd flags (look at Section 3.3 in the E1000 manual) and stash away a pointer to the mbuf for later freeing.
-   Finally, update the ring position by adding one to `E1000_TDT` modulo `TX_RING_SIZE`.
-   If `e1000_transmit()` added the mbuf successfully to the ring, return 0. On failure (e.g., there is no descriptor available to transmit the mbuf), return -1 so that the caller knows to free the mbuf.

```c
int
e1000_transmit(struct mbuf *m)
{
  //
  // Your code here.
  //
  // the mbuf contains an ethernet frame; program it into
  // the TX descriptor ring so that the e1000 sends it. Stash
  // a pointer so that it can be freed after sending.
  //

  acquire(&e1000_lock);
  
  // ask next index by reading TDT
  uint32 next_idx = regs[E1000_TDT];
  struct tx_desc *cur = &tx_ring[next_idx]; 

  // next descriptor in ring is not ready, just report an error
  if (!(cur->status & E1000_TXD_STAT_DD)) {
    release(&e1000_lock);
    return -1;
  }

  // free old buf
  if (tx_mbufs[next_idx]) mbuffree(tx_mbufs[next_idx]);
  
  // fill descriptor by set head, length, cmd
  tx_mbufs[next_idx] = m;
  cur->addr = (uint64)m->head;
  cur->length = m->len;
  cur->cmd = E1000_TXD_CMD_RS | E1000_TXD_CMD_EOP;
  
  // update by add and modulo
  next_idx = (next_idx + 1) % TX_RING_SIZE;
  regs[E1000_TDT] = next_idx;

  release(&e1000_lock);
  return 0;
}
```

设置 descriptor 的 cmd 字段, 其中 RS 是必须要设置的, 可以保证 E1000 一定会正确设置 status 中的 DD 字段; xv6 默认不需要支持大 packet (一个 packet 多个 descriptor), 因此这里直接无脑设置 EOP 即可

要注意的是, 网卡一定是要支持多进程的, 因此 e1000_transmit 是可能出现线程公平性问题的, 否则可能出现后一个进程的 mbuf 覆盖掉前一个进程的 mbuf 的情况 (比如前一个进程 update next_idx 之前, 下一个进程就进入并重新分配了 mbuf)

#### receive

-   First ask the E1000 for the ring index at which the next waiting received packet (if any) is located, by fetching the `E1000_RDT` control register and adding one modulo `RX_RING_SIZE`.
-   Then check if a new packet is available by checking for the `E1000_RXD_STAT_DD` bit in the `status` portion of the descriptor. If not, stop.
-   Otherwise, update the mbuf's `m->len` to the length reported in the descriptor. Deliver the mbuf to the network stack using `net_rx()`.
-   Then allocate a new mbuf using `mbufalloc()` to replace the one just given to `net_rx()`. Program its data pointer (`m->head`) into the descriptor. Clear the descriptor's status bits to zero.
-   Finally, update the `E1000_RDT` register to be the index of the last ring descriptor processed.
-   `e1000_init()` initializes the RX ring with mbufs, and you'll want to look at how it does that and perhaps borrow code.
-   At some point the total number of packets that have ever arrived will exceed the ring size (16); make sure your code can handle that.

```c
static void
e1000_recv(void)
{
  //
  // Your code here.
  //
  // Check for packets that have arrived from the e1000
  // Create and deliver an mbuf for each packet (using net_rx()).
  //
 
  // ask next index by read RDT
  for (uint32 next_idx = regs[E1000_RDT] + 1; ; next_idx++) {
    if (next_idx == RX_RING_SIZE) next_idx = 0;
    struct rx_desc *cur = &rx_ring[next_idx];
	// next buffer has not been filled
    if (!(cur->status) &E1000_RXD_STAT_DD) return;
	// update m->len and deliver mbuf
    struct mbuf* m = rx_mbufs[next_idx];
    m->len = cur->length;
    net_rx(m);
    // allocate a new mbuf (when mbuf is given to net_rx, old mbuf is released)
    rx_mbufs[next_idx] = mbufalloc(0);
    // program data pointer into descriptor and clear status bits
    cur->addr = (uint64)rx_mbufs[next_idx]->head;
    cur->status = 0;
    // update E1000_RDT
    regs[E1000_RDT] = next_idx;
  }
}
```

E1000 可能存在一次写入多个 mbuf 的情况, 因此在将 mbuf 递给 xv6 的时候需要使用循环保证每个被 E1000 写入的 mbuf 都可以被呈递给 xv6 kernel

## locks

这个 lab 需要实现的是锁的拆分, 通过将 giant-lock 拆分为 fine-grained lock, 可以显著提升并发性能

### memory allocator

xv6 kernel 的内存分配器只有一个 lock, 无论分配还是释放 block, 都需要获取相同的 block, 当多个进程试图获取获取内存时, 会产生严重的效率问题

xv6 解决这个问题的思路是为每个处理器核心分配一个 lock, 这样只有运行在同一个同一个处理器核心上的程序才需要通过加锁避免竞争, 对于同时运行在不同处理器上的进程而言, 其完全是并行分配内存的

然而这种构想是很美好的, 如果将内存均分给每个处理器, 那么必然会出现某个处理器的内存耗尽的情况, 此时需要从别的处理器核心 "借点" 内存, 为了避免竞争, 需要获取其他处理器核心的锁, 在这种多个锁的环境中, 要注意加锁顺序, 不要出现死锁

xv6 默认使用单链表记录物理页, 这里将其拆分为多个链表, 每个链表对应一个处理器核心, 且每个链表都有一个锁, 用来避免同一处理器核心上进程的竞争

```c
// kalloc.c

struct mem_list {
  struct spinlock lock;
  struct run *freelist;
};

struct {
  // giant lock is used only if current cpu list is empty
  // struct spinlock giant_lock;
  struct mem_list list[NCPU];
} kmem;

void
kinit()
{
  for (int i = 0; i < NCPU; i++) initlock(&kmem.list[i].lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}

void
freerange(void *pa_start, void *pa_end)
{
  char *p = (char*)PGROUNDUP((uint64)pa_start);

  for (char *cur = p; cur < (char*)pa_end; cur += PGSIZE) {
    uint64 idx = ((cur - p) / PGSIZE) % NCPU;
    struct run *r = (struct run*)cur;
    r->next = kmem.list[idx].freelist;
    kmem.list[idx].freelist = r; 
  }
}

```

>   内存初始化时, 通过取模运算, 将各个物理页分配给不同的处理器核心

分配内存的时按照参考手册的说法, 首先需要阻塞处理器上的中断, 然后才能安全的调用 cpuid() 获取当前核心的编号, 然后按照一般流程加锁获取物理页

当前处理器上的物理页全被分配完时, 需要向其他处理器 "借" 物理页, 在向其他核心 "借" 之前, 需要释放当前核心的锁, 否则可能出现分配在两个处理器上的进程相互加锁从而死锁的情况

```c
// kalloc.c
void *
kalloc(void)
{
  struct run *r;

  push_off();
  int cpu = cpuid();
  pop_off();
  
  for (int i = 0; i < NCPU; i++, cpu++) {
    if (cpu == NCPU) cpu = 0;
    
    acquire(&kmem.list[cpu].lock);
    r = kmem.list[cpu].freelist; 
    if (r) kmem.list[cpu].freelist = r->next;
    release(&kmem.list[cpu].lock);

    if (r) {
      memset((char*)r, 5, PGSIZE); // fill with junk
      return (void*)r;
    }
  }

  // when it returns 0, kernel may still has memory 
  return (void*)r;
}
```

从写法上直接优化为一个 for 循环, 每轮循环获取一个锁, 当前处理器核心仅仅起到确定遍历起点的作用

回收物理内存是类似的, 直接将当前物理页连接到对应核心的链表上即可

```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;
  push_off();
  int cpu = cpuid();
  pop_off();
  
  acquire(&kmem.list[cpu].lock);
  r->next = kmem.list[cpu].freelist;
  kmem.list[cpu].freelist = r;
  release(&kmem.list[cpu].lock);
}
```

个人认为, 这里在回收内存时, 并不需要将物理内存地址取模放在对应核心的链表上, 因为进程和处理器核心并不是完全绑定的, 每个进程会随机的运行在某个核心, 即便是同一个进程, 释放内存的时候, 也可以将物理页随机分配到不同核心上

### buffer cache

xv6 的文件系统在内存中对文件进行了缓存, 当某个进程试图写文件时, 并不会直接写道硬盘上, 首先会写在内存中, xv6 使用 bcache 抽象缓存对象

默认的情况下, 所有的 block cache 公用一个缓存, 每个 block 写入内存都需要获取相同的锁, xv6 解决的思路和上面 memory allocator 是类似的, 将 block 分配到不同的 bucket 中, 每个 bucket 还是使用双向链表管理各个 block, 每个 bucket 使用一个 lock 管理避免 bucket 内部的 block 的竞争

```c
// bio.c
struct bucket {
  struct spinlock lock;
  struct buf head;
};

struct {
  struct buf buf[NBUF];
  struct bucket buckets[NBUCKET]; 
  struct spinlock giant_lock;
} bcache;

void
binit(void)
{
  struct buf *b;

  // initialize giant lock, in case of deadlock
  initlock(&bcache.giant_lock, "bcache_giant");

  // initialize bucket
  for (int i = 0; i < NBUCKET; i++) {
    // locks
    initlock(&bcache.buckets[i].lock, "bcache");
    // linked list
    bcache.buckets[i].head.prev = &bcache.buckets[i].head;
    bcache.buckets[i].head.next = &bcache.buckets[i].head;
  }

  b = bcache.buf; 

  // insert blocks
  for (int i = 0; i < NBUF; i++, b++) {
    int idx = i % NBUCKET;
    initsleeplock(&b->lock, "buffer");
    b->next = bcache.buckets[idx].head.next;
    b->prev = &bcache.buckets[idx].head;
    bcache.buckets[idx].head.next->prev = b;
    bcache.buckets[idx].head.next = b;
  }
}
```

函数 bget 会从指定 dev 加载 blockno 到内存中, 可以通过对 blockno 取模找到对应的 bucket, 每个 bucket 中保存了一个双向链表, 和 kalloc 不同的是, 同个 block 可能被引用多次, 因此这里首先遍历一次双链表查看是否缓存过 block 了, 如果没有缓存过, 则需要从 bucket 中找到一个空闲的 block 作为缓存

不过还是会出现类似上面某个核心物理页不足的情况, 当某个 bucket 中的 block 已经全部被分配出去了, 则需要从其他 bucket 中 "借" block, 类似的也需要避免死锁的情况

```c
// bio.c

static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  uint64 idx = blockno % NBUCKET;

  acquire(&bcache.buckets[idx].lock);

  // check if current block has already cached 
  for (b = bcache.buckets[idx].head.next; b != &bcache.buckets[idx].head; b = b->next) {
    if (b->dev == dev && b->blockno == blockno) {
      b->refcnt++;
      
      // disconnect block
      b->next->prev = b->prev;
      b->prev->next = b->next;

      // connect to head
      b->next = bcache.buckets[idx].head.next;
      b->prev = &bcache.buckets[idx].head;
      bcache.buckets[idx].head.next->prev = b;
      bcache.buckets[idx].head.next = b;

      // release resource
      release(&bcache.buckets[idx].lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // allocate a block to cache
  for (b = bcache.buckets[idx].head.prev; b != &bcache.buckets[idx].head; b = b->prev) {
    if (b->refcnt == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      
      // disconnect block
      b->next->prev = b->prev;
      b->prev->next = b->next;

      // connect to head
      b->next = bcache.buckets[idx].head.next;
      b->prev = &bcache.buckets[idx].head;
      bcache.buckets[idx].head.next->prev = b;
      bcache.buckets[idx].head.next = b;
      
      // release resource
      release(&bcache.buckets[idx].lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  release(&bcache.buckets[idx].lock);

  // steal a block from other bucket
  acquire(&bcache.giant_lock);
  uint64 steal = idx + 1;
  for (int i = 0; i < NBUCKET; i++, steal++) {
    if (steal == NBUCKET) steal = 0;
    acquire(&bcache.buckets[steal].lock);
    for (b = bcache.buckets[steal].head.prev; b != &bcache.buckets[steal].head; b = b->prev) {
      if (b->refcnt == 0) {
        b->dev = dev;
        b->blockno = blockno;
        b->valid = 0;
        b->refcnt = 1;
        
        // disconnect block
        b->next->prev = b->prev;
        b->prev->next = b->next;

        acquire(&bcache.buckets[idx].lock);
        // connect to head
        b->next = bcache.buckets[idx].head.next;
        b->prev = &bcache.buckets[idx].head;
        bcache.buckets[idx].head.next->prev = b;
        bcache.buckets[idx].head.next = b;
        release(&bcache.buckets[idx].lock);
        
        release(&bcache.buckets[steal].lock);
        release(&bcache.giant_lock);
        acquiresleep(&b->lock);
        return b;
      }
    }
    release(&bcache.buckets[steal].lock);
  }

  release(&bcache.giant_lock);
  
  panic("bget: no buffers");
}
```

整体实现上看起来比较麻烦, 但大部分代码都在处理连接节点到双链表以及从双链表中分离节点, 这里使用 LRU 管理双链表, 尽可能利用 space locality (csapp), 减少后续查询同一个 block, 找寻空闲 block 的开销

这里要注意的是, 当从其他 bucket "借" block 时, 还需要将借来的 block 连接到当前 bucket 中, 为了保证数据的一致性, 需要当前进程同时持有两个 bucket 的锁, 但因为每个  bucket 时平等的, 不同进程获取 bucket 的锁的顺序可能存在交叉, 因此当需要从其他 block "借" 锁的时候还需要使用 bcache 提供的 giant_lock, 这样每个进程在 "借" block 的时候都会被强制先获取 giant_lock (保证获取锁的顺序)

不过这里要注意一个细节, 在获取 giant_lock 只要首先要释放已经获取的锁, 否则会出现获取 giant_lock 和 bucket_lock 存在交叉导致死锁的情况

释放 block 的时候需要根据 block 从属的 bucket, 将 block "归还" 到对应的 bucket 中

```c
// bio.c

void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  uint64 idx = b->blockno % NBUCKET;

  acquire(&bcache.buckets[idx].lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // disconnect block
    b->next->prev = b->prev;
    b->prev->next = b->next;
    
    uint64 ori = (b - bcache.buf) % NBUCKET;
    if (ori == idx) {
      b->prev = bcache.buckets[idx].head.prev->prev;
      b->next = bcache.buckets[idx].head.prev;
      bcache.buckets[idx].head.prev->prev->next = b;
      bcache.buckets[idx].head.prev->prev = b;
      release(&bcache.buckets[idx].lock);
      return;
    }
    release(&bcache.buckets[idx].lock);

    acquire(&bcache.buckets[ori].lock);
    b->prev = bcache.buckets[ori].head.prev->prev;
    b->next = bcache.buckets[ori].head.prev;
    bcache.buckets[ori].head.prev->prev->next = b;
    bcache.buckets[ori].head.prev->prev = b;
    release(&bcache.buckets[ori].lock);
    return;
  }
  release(&bcache.buckets[idx].lock);
}
```

为了充分利用 space locality, 这里将释放的 block 放在了尾节点的位置

此外, 尽管 xv6 lab 并没有直接要求, 这里还是对 bpin 和 bunpin 进行了修改, 避免使用 giant lock

```c
// bio.c

void
bpin(struct buf *b) {
  uint64 idx = b->blockno % NBUCKET;
  acquire(&bcache.buckets[idx].lock);
  b->refcnt++;
  release(&bcache.buckets[idx].lock);
}

void
bunpin(struct buf *b) {
  uint64 idx = b->blockno % NBUCKET;
  acquire(&bcache.buckets[idx].lock);
  b->refcnt--;
  release(&bcache.buckets[idx].lock);
}
```

## file system

qemu 调用 mkfs 初始化 xv6 的文件系统, 在这个 lab 的测试中会在文件系统中添加/删除若干文件, 为了不影响后续测试, 最好在进行下一次调试之前先 `make clean` 清空之前保存的临时文件 

### Large files

xv6 使用 inode 表示一个文件

```c
// fs.h
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT + 1];   // Data block addresses
};
```

inode 使用 12 个 direct pointer 和一个 indirect pointer 存储文件内容, xv6 book 的 fig 8.3 描述了存储方式:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/09/16/19:36:43:file_representation_on_disk.png)

所以 xv6 默认支持最大的文件大小为 12 + 256 => 268 block 

为了支持更大的文件大小, 这里 xv6 要求实现 double-indirect pointer, 一个 double-indirect pointer 指向一个 block, 其存储内容为 indirect pointer, 而其中每个 indirect pointer 指向一个 block, 这样一个 double-indirect pointer 可以索引 256 * 256 => 65536 block

xv6 要求不能修改 inode 本身的大小, 因此这里需要借出一个 direct pointer, 将其变为一个 double-indirect pointer, 此时一个 inode 可以支持最大的文件大小为 11 + 256 + 65536 => 65803 block

为了让一个文件支持更多 block, 需要修改 inode 本身, 包括 inode on disk 和 inode in memory

```c
// fs.h
// modified for lab: fs
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define NDINDIRECT (NINDIRECT * NINDIRECT)
#define MAXFILE (NDIRECT + NINDIRECT + NDINDIRECT)

// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+2];   // Data block addresses
};

// file.h
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  // modified for lab: fs
  uint addrs[NDIRECT+2];
};
```

此外还需要修改 bmap, 这是一个映射函数, 负责返回 inode 中某个 block 在内存中的地址, 并在 block 尚未进行映射的时候在内存中分配一个 block

```c
// fs.c
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[bn] = addr;
    }
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT] = addr;
    }
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }
  
  // added for lab: fs
  bn -= NINDIRECT;

  if (bn < NDINDIRECT) {
    // load double-indirect block, allocating if necessory
    if ((addr = ip->addrs[NDIRECT + 1]) == 0) {
      addr = balloc(ip->dev);
      if (addr == 0) return 0;
      ip->addrs[NDIRECT + 1] = addr;
    }
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    uint div = bn / NINDIRECT;
    
    // load indirect block, allocating if necessory
    if ((addr = a[div]) == 0) {
      addr = balloc(ip->dev);
      if (addr == 0) return 0;
      a[div] = addr;
      log_write(bp);
    }
    brelse(bp);
    
    uint modulo = bn % NINDIRECT;
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    // load block, allocating if necessory
    if ((addr = a[modulo]) == 0) {
      addr = balloc(ip->dev);
      if (addr == 0) return 0;
      a[modulo] = addr;
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

### symbolic links

这部分需要让 xv6 支持符号链接, 符号链接不会修改 inode 的引用计数, 可以认为符号链接是一个文本文件, 该文本中保存了到实际文件的路径, 因此符号链接也是可以嵌套的, 特别的在这部分中 xv6 要求嵌套层数不超过 10 层

xv6 需要实现一个 syscall => sys_symlink, 具有两个参数 target 和 path, 即在 path 中创建一个到 target 的符号链接, 在调用 syscall 的时候 path 和 target 都不要求必须存在

在实现这个 syscall 之前, 需要创建添加两个宏定义:

```c
// stat.h
// added for lab: fs
#define T_SYMLINK 4   // this file type is a symbolic link

// fcntl.h
// added for lab: fs
#define O_NOFOLLOW 0x800 // this flag used for symbolic link file, if this flag is set in `open`, target file is symbolic link file itself
```

添加了一个文件类型, 同时添加了一个访问控制位, 具体的作用可以查看 lab 的 hints 部分

syscall 的实现并不难, 首先获取 path 的 inode, 然后将 target 写进入就行了

```c
// sysfile.c 
// added for lab: fs
// create symbolic link at path to target
// just allocate a block and write path into the block
uint64 sys_symlink(void) {
  char target[MAXPATH], path[MAXPATH];
  struct inode *ip;
  int n, i = 0, r;

  if ((n = argstr(0, target, MAXPATH)) < 0) return -1;
  if ((n = argstr(1, path, MAXPATH)) < 0) return -1;
  
  begin_op();
  if ((ip = create(path, T_SYMLINK, 0, 0)) == 0) {
    end_op();
    return -1;
  }

  n = strlen(target);
  while (i < n) {
    r = n - i;
    if ((r = writei(ip, 0, (uint64)target + i, i, r)) > 0) i += r;
  }

  iunlockput(ip);
  end_op();
  return 0;
}
```

首先使用 create 获取对应的 inode, 这个函数不会无脑创建一个 inode, 会先检查 inode 是否已经存在了, 只有在 inode 不存在时才会创建新的 inode

>   xv6 提供的 create 处理了很多边界情况, 直接用就行了, 这里最好不要再造轮子了

从 create 返回的 inode 已经被加过锁了, 因此后面可以直接将 target 写入 inode content, 这里使用 while 处理 short count 的情况, 避免写文件操作被中断的情况

释放锁的时候使用了 iunlockput, 彻底释放 inode in memory, 这是因为在很多情况下创建符号链接和读取符号链接并不在一个程序内, 很多程序仅设置一个符号链接就结束了

由于需要修改文件系统, 这里一定要使用 begin_op 和 end_op 修改部分包裹起来 (xv6 的事务声明和提交)

当其他程序试图打开一个符号链接文件时, 程序试图打开的链接到的文件, 因此这里需要修改 sys_open 使其可以打开正确的文件

```c
// sysfile.c
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  argint(1, &omode);
  if((n = argstr(0, path, MAXPATH)) < 0)
    return -1;

  begin_op();

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
      end_op();
      return -1;
    }
    ilock(ip);
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
  }

  if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
    iunlockput(ip);
    end_op();
    return -1;
  }
  
  // added for lab: fs
  if (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
    // inode type is symbolic, follow bits is not set
    char buff[MAXPATH];
    for (int i = 0; i < SYMLINK_DEPTH && ip->type == T_SYMLINK; i++) {
      int off = 0, r;
      while ((r = readi(ip, 0, (uint64)buff + off, off, MAXPATH)) > 0) off += r;
      iunlockput(ip);
      if ((ip = namei(buff)) == 0) {
        end_op();
        return -1;
      }
      ilock(ip);
    }

    if (ip->type == T_SYMLINK) {
      iunlockput(ip);
      end_op();
      return -1;
    }
  }

  if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
    if(f)
      fileclose(f);
    iunlockput(ip);
    end_op();
    return -1;
  }

  if(ip->type == T_DEVICE){
    f->type = FD_DEVICE;
    f->major = ip->major;
  } else {
    f->type = FD_INODE;
    f->off = 0;
  }
  f->ip = ip;
  f->readable = !(omode & O_WRONLY);
  f->writable = (omode & O_WRONLY) || (omode & O_RDWR);

  if((omode & O_TRUNC) && ip->type == T_FILE){
    itrunc(ip);
  }

  iunlock(ip);
  end_op();

  return fd;
}
```

当 open 的文件是一个符号链接文件, 并且没有设置 NOFOLLOW bit, 那么此时程序需要打开的就是链接到的文件, 由于链接可以嵌套, 这里需要循环查找

当进行查找时一定要获取到对应 inode 的 lock, 在查找下一级文件时, 需要先释放上一级 inode 的 lock

## mmap

>   最后一个 lab 了 !!!

这个 lab 要求实现两个 syscall:

*   `void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset)`: xv6 并不要求实现一个功能完整的 mmap, 这里仅仅要求实现一个最简的 mmap
    *   addr: 在 linux 中 mmap 会尝试将 fd 对应的文件内容映射到 addr 对应的位置, 而在 xv6 中 addr 不起任何作用, xv6 kernel 负责从可用虚拟地址中任意合适的位置映射
    *   length: 映射内容的大小
    *   prot: 本身是一个 bit mode, 表示了对应映射区域的访问权限
    *   flags: 默认支持两个 flag: MAP_SHARED 和 MAP_PRIVATE, 用来表示是否会将修改结果同步到文件中, 当 flag 被设置为 MAP_PRIVATE 时, 此时 prot 和之前 open 文件时使用的控制位无关, 即就算使用 RDONLY open 了文件, 映射到的区域也是可写的 (具体行为和 prot 有关, 但不管如何不会同步到文件中)
    *   fd: 将 fd 对应的文件内容映射到内存中
    *   offset: xv6 保证了 offset 在测试用例中一定为 0, 因此这个参数和 addr 一样都是无用的 
*   `int munmap(void *addr, size_t length)`: 取消从 addr 处开始, 大小为 length 的内存映射, xv6 保证取消映射的区域一定位于开头或结尾, 在取消映射完成后不会出现空洞的情况

mmap 的映射是 lazy allocation, 仅仅是进行了标记, 并不会占用实际的物理内存, 而在后续访问到相应内存后触发 load-page-fault 才会进行内存分配

为了记录当前进程使用的映射区域, 需要添加额外的结构体记录, lab 建议可以一切从简, 为每个进程分配固定大小的结构体数组记录

```c
// param.h
// added for lab: mmap
#define NVMA         16    // maximum number of virtual memory area	

// proc.h
// added for lab: mmap
struct vma {
  int valid;
  uint64 addr;
  uint64 file_start;
  uint64 length;
  uint access;
  uint writeback;
  struct file *ofile;
};

// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

  // added for lab: mmap
  struct vma vmas[NVMA];       // virtual memory areas of process
};
```

>   lab 建议大小为 16 的数组就可以了

由于在结构体 proc 中添加了结构体 vma, 为了避免出现非法值的情况, 需要在初始化进程的时候要为每个 vma 的 valid 赋为 0

### mmap

由于需要将文件映射到虚拟地址空间中, 低地址处已经被代码段, 数据段, 栈, 可扩展的堆占用了, 因此这里选择将文件映射到内存的高地址处

内存地址最顶层分别为 trampoline 和 trapframe 各占一个 page, 这里将文件映射到 trapframe 以下, 虚拟内存空间资源可比物理内存资源大的多了, 因此这里在进行映射的时候直接遍历所有的 vma, 然后将当前区域映射到最靠前的 vma 之下

```c
// sysfile.c
uint64 sys_mmap(void) {
  int length, prot, flags, fd;
  argint(1, &length); 
  argint(2, &prot);
  argint(3, &flags);
  argint(4, &fd);

  struct proc *p = myproc();
  struct file *f = p->ofile[fd];

  int access_bits = PTE_U;
  int writeback = 0;
  if (flags == MAP_SHARED) {
    if (prot & PROT_WRITE) {
      if (!f->writable) return -1;
      access_bits |= PTE_W;
    }

    if (prot & PROT_READ) {
      if (!f->readable) return -1;
      access_bits |= PTE_R;
    }
    writeback = 1;
  } else {
    if(prot & PROT_WRITE) access_bits |= PTE_W;
    if (prot & PROT_READ) access_bits |= PTE_R;
  }

  int idx = -1;
  uint64 end_addr = MAXVA - (PGSIZE << 1);
  for (int i = 0; i < NVMA; i++) {
    if (!p->vmas[i].valid) idx = i;
    else if (p->vmas[i].addr < end_addr) end_addr = p->vmas[i].addr;
  }
  if (idx < 0) return -1;

  // round up division 
  uint page_cnt = (length + PGSIZE - 1) / PGSIZE;
  length = page_cnt * PGSIZE;
  p->vmas[idx].valid = 1;
  p->vmas[idx].addr = end_addr - length;
  p->vmas[idx].file_start = p->vmas[idx].addr;
  p->vmas[idx].length = length;
  p->vmas[idx].ofile = filedup(p->ofile[fd]);
  p->vmas[idx].access = access_bits;
  p->vmas[idx].writeback = writeback;
  return p->vmas[idx].addr;
}
```

这里的 access bit 是为了服务后续处理异常时进行页表映射的, 这里将 prot 翻译为 PTE 的权限位, lab 提示要调用 filedup 增加文件索引, 避免文件被释放

触发 load-page-fault 后需要将文件内容加载到内存中

```c
// trap.c
// added for lab: mmap
static void usertrap_excaption_handler() {
  struct proc *p = myproc();
  printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
  printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
  setkilled(p);
}

static int load_page_fault_handler() {
  struct proc *p = myproc();
  // when load page fault araise, stval stores virtual address cause this fault
  uint64 addr = r_stval();
  addr = PGROUNDDOWN(addr);

  int i;
  for (i = 0; i < NVMA; i++) {
    if (!p->vmas[i].valid) continue;
    uint64 end = p->vmas[i].addr + p->vmas[i].length;
    if (addr >= p->vmas[i].addr && addr < end) break;
  }
  if (i == NVMA) return -1;
  
  uint64 page = (uint64)kalloc();
  memset((char*)page, 0, PGSIZE);
  if (!page) return -1;
  if (mappages(p->pagetable, addr, PGSIZE, page, p->vmas[i].access) < 0) return -1;
  if (load_vma(p->vmas[i].ofile, addr - p->vmas[i].file_start, addr) < 0) return -1; 
  return 0;
}

//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();

  uint64 scause = r_scause();

  if(r_scause() == 8){
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    intr_on();

    syscall();
  } else if (scause == 0xd) {
    if (load_page_fault_handler() < 0) usertrap_excaption_handler();
  }
  else if ((which_dev = devintr()) != 0) {
    // ok
  } else usertrap_excaption_handler();

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

首先需要校验地址的合法性, 只有是访问虚拟内存区域导致的异常才会被正常的处理, 其余的地址导致的异常都需要将进程终止掉

要注意的是 kalloc 分配一个物理页后, 会写入 junk data (0x5), 而在 mmaptest 中进行校验时, 对于超出的文件大小的部分, xv6 希望访问到的是空字节 0

最后调用 load_vma 将文件加载到内存中, 这个函数的实现放在了 file.c 中

```c
// file.c
// added for lab: mmap
/**
 * load a page from file @param: f into @param: va
 * page has offset @param: offset
 * returns -1 on load fault
*/
int load_vma(struct file* f, uint64 offset, uint64 va) {
  struct inode *inode = f->ip;
  int i = 0, r;
  ilock(inode);
  while (i < PGSIZE) {
    r = PGSIZE - i;
    r = readi(inode, 1, va + i, offset + i, r);
    if (r == 0) break;
    if (r < 0) return -1;
    i += r;
  }
  iunlock(inode);
  return 0;
}
```

由于需要读文件, 为 inode 加了锁, 同时为了处理 short count, 使用 while 循环读文件

### munmap

munmap 需要进行资源释放, 必要时还需要将内存写回文件, 由于程序终止时需要释放程序占用的所有空间, 包括虚拟内存空间, 因此这里对 sys_munmap 进行了抽取

```c
// sysfile.c
/**
 * int mummap(void *addr, size_t length);
 * @param: addr   => start address of unmaped area 
 * @param: length => unmapped area size
*/
uint64 sys_munmap(void) {
  uint64 addr;
  int length;
  
  argaddr(0, &addr);
  argint(1, &length);

  return munmap(addr, length);
}
```

munmap 的实现放在了 proc.c 中, exit 会调用 munmap 释放占用的空间

```c
// riscv.h
// added for lab: mmap
#define PTE_D (1L << 7) // dirty bit shows whether current page has been writen

// proc.c
int munmap(uint64 addr, int length) {
  struct proc *p = myproc();
  int idx = 0;
  for (; idx < NVMA; idx++) {
    if (!p->vmas[idx].valid) continue;
    uint64 end = p->vmas[idx].addr + p->vmas[idx].length;
    if (p->vmas[idx].addr <= addr && addr < end) break;
  }

  if (idx == NVMA) return -1;
  // unmap will just from the start or the end
  if (addr != p->vmas[idx].addr) addr = PGROUNDDOWN(addr);
  uint page_cnt = (length + PGSIZE - 1) / PGSIZE;
  length = page_cnt * PGSIZE;
  
  uint64 cur = addr;
  pte_t *pte;

  for (int i = 0; i < page_cnt; i++, cur += PGSIZE) {
      pte = walk(p->pagetable, cur, 0);
      if (*pte == 0) continue; 
      if (*pte & PTE_D && p->vmas[idx].writeback) {
        if (store_vma(p->vmas[idx].ofile, cur - p->vmas[idx].file_start, cur) < 0) return -1;
      }
      uvmunmap(p->pagetable, cur, 1, 1);
  }

  if (addr == p->vmas[idx].addr) p->vmas[idx].addr += length;
  p->vmas[idx].length -= length;

  if (p->vmas[idx].length == 0) {
    p->vmas[idx].valid = 0;
    fileclose(p->vmas[idx].ofile);  
  }
  return 0;
}
```

xv6 建议在写回内存的时候仅仅写回那些被修改过的 page, 因此这里使用到了 PTE 中的 dirty bit, 类似于 load_vma 这里使用 store_vma 实现从内存写回文件的操作

在当前虚拟内存区域为空的时候需要减少当前文件的引用计数, 避免文件占用导致的内存泄漏

```c
// file.c
/**
 * store a page from @param: va into file @param: f
 * page has offset @param: offset 
 * returns -1 on storage fault
*/
int store_vma(struct file *f, uint64 offset, uint64 va) {
  struct inode *inode = f->ip;
  int max = ((MAXOPBLOCKS-1-1-2) / 2) * BSIZE;
  int i = 0, r;
  while (i < PGSIZE) {
    r = PGSIZE - i;
    if (r > max) r = max;
    begin_op();
    ilock(inode);
    r = writei(inode, 1, va + i, offset + i, r);
    iunlock(inode);
    end_op();
    if (r <= 0) break; 
    i += r;
  }
  if (r <= 0) return -1;
  return 0; 
}
```

在实现 store_vma 的时候根据 lab hint 可知一次写入文件的大小最好不要超过一个 block 大小, 而物理内存以 page 为单位保存数据, 因此这里参考了 filewrite 的写法, 计算一次写入的最大大小 max, 在写入之前调用 begin_op 开启事务, 写入结束后调用 end_op 提交事务

### fork

子进程需要继承父进程的 vma, 因此这里需要修改 fork 函数

```c
// proc.c
// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;

  // added for lab: mmap
  for (int i = 0; i < NVMA; i++) {
    if (p->vmas[i].valid) {
      np->vmas[i].valid = 1;
      np->vmas[i].addr = p->vmas[i].addr;
      np->vmas[i].file_start = p->vmas[i].file_start;
      np->vmas[i].length = p->vmas[i].length;
      np->vmas[i].ofile = filedup(p->vmas[i].ofile);
      np->vmas[i].access = p->vmas[i].access;
      np->vmas[i].writeback = p->vmas[i].writeback;
    }
  }

  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  release(&np->lock);

  acquire(&wait_lock);
  np->parent = p;
  release(&wait_lock);

  acquire(&np->lock);
  np->state = RUNNABLE;
  release(&np->lock);

  return pid;
}
```

当然可以在修改之前先 merge 之前 cow lab 的修改, 这样可以更高级的 lazy allocation