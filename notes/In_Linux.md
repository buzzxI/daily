# Ubuntu 的坑

## 换源

借助阿里镜像源：[阿里巴巴开源镜像站-OPSX镜像站-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/mirror/)

修改的文件为 `/etc/apt/sources.list`

将所有的 `http://archive.ubuntu.com` 修改为 `http://mirrors.aliyun.com`

>   建议是是备份原来的文件，并直接从镜像站上把他写好的文件写进入
>
>   ```shell
>   # cp -p sources.list sources.list.bk
>   ```

## 编译出错

*   编译出现fatal error: bits/libc-header-start.h: No such file or directory，这个真的不赖我，Ubuntu 默认就缺少库文件；不过也很好解决

    ```shell
    $ sudo apt update install gcc-multilib
    ```

## oh-my-zsh

使用 zsh 的美化 wsl 的目的是：平时使用 root 用户的时候没有高亮，有的时候信息多了找不到关键点，实在受不了了，也不想自己优化了，就直接用 zsh 了

首先需要下载 zsh，这个好说直接 apt install 就行

看 oh-my-zsh，官网在：https://ohmyz.sh/，就一个命令，其实是下载一个 .sh 脚本，尝试过直接下，下不动；直接到 github 上复制下来，自己运行

美化的主题使用的是 (~~[Powerlevel9k](https://github.com/Powerlevel9k/powerlevel9k)~~)[powerlevel10k](https://github.com/romkatv/powerlevel10k)

>   升级了

## 瞎写文件名

一不小心创建了一个名字里带有 '-' 的文件

```shell
$ ls                                                                  
-Og  dump_file  test.c  test.s
```

好吧，一看就是在 gcc -o 的时候使用了参数 -Og

现在的问题在于这个文件创建好了之后，就删不掉了 rm -rf -Og，他会说 rm 命令没有参数 -Og

这个时候删除就必须带上路径了
```shell
$ rm -rf ./-Og
```

# 文件权限

## 用户和用户组

linux中的文件权限针对三类对象：`owner`(所属用户)、`group`(所属组)、`other`(其他)

一个用户可以属于多个用户组

关于 linux 系统下的用户信息，保存在 /etc/passwd 中；用户的密码信息保存在 /etc/shadow 下；用户组信息保存在 /etc/group 下

为了查看文件的属性信息：

```shell
buzz@buzz:~$ ls -al
total 76
drwxr-xr-x 7 buzz buzz  4096 Jul  3 22:53 .
drwxr-xr-x 3 root root  4096 Jun 29 00:09 ..
-rw------- 1 buzz buzz  6803 Jul  3 23:07 .bash_history
-rw-r--r-- 1 buzz buzz   220 Jun 29 00:09 .bash_logout
-rw-r--r-- 1 buzz buzz  3771 Jun 29 00:09 .bashrc
drwxr-xr-x 3 buzz buzz  4096 Jun 29 00:09 .cache
drwx------ 3 buzz buzz  4096 Jun 29 00:09 .config
-rw------- 1 buzz buzz    28 Jul  1 18:03 .lesshst
-rw-r--r-- 1 buzz buzz   807 Jun 29 00:09 .profile
-rw-r--r-- 1 buzz buzz     0 Jun 29 00:11 .sudo_as_admin_successful
-rw------- 1 buzz buzz 16900 Jul  3 22:53 .viminfo
-rw-rw-r-- 1 buzz buzz    14 Jun 30 21:21 .vimrc
```

## 文件权限的意义

每个对象对文件的权限通过三个字符表示：`r`读，`w`写，`x`执行

这种说法太宽泛了，在 linux 中一切都是文件，包括目录，而显然目录具有执行权力，这是不符合逻辑的

对于普通的文件，比如文本文件、二进制文件而言，其内部存储的是实际的数据：

*   r：表示可以读取这个文件中的数据

*   w：表示可以修改这个文件中的数据(就是编辑)

*   x：表示这个文件可以被执行

    >   在 win 下如果文件的后缀为 .exe，.bat 就表示这个文件可以被执行，但在 linux 中文件是否可以执行并不取决于文件名后缀，而是某用户是否具有某文件的 x 权限

要注意的是 w 表示可以对文件内的数据进行编辑，并不代表可以删除这个文件，是否可以新建或删除这个文件，并不取决于某用户对当前文件是否具有 w 权限，这一点很重要，后面还会再提

对于目录(文件)，可以认为这个文件中记录的是目录下的文件名，所以：

*   r：表示可以读取目录下的文件名，理解为可以查询某个目录下的文件

*   w：表示可以修改目录下的文件名，理解为可以在某个目录下新建文件、删除文件、修改文件名、移动文件或目录的位置

    >   w 意味着可以对目录下文件名进行编辑操作

*   x: 表示可以用户是否可以进入当前目录(就是 cd 是否可以进入当前目录)

举一个例子：root 用户在 /home/buzz 下创建了目录 root_dir

```shell
root@buzz:/home/buzz# mkdir root_dir
root@buzz:/home/buzz# ll | grep root_dir
drwxr-xr-x 2 root root  4096 Jul  4 17:15 root_dir/
```

可以看到这个目录是允许其他用户读取，进入的，但是未开放 w 权限，即其他用户不可以在 root_dir/ 下新建文件，也不可以删除原来的文件，不可以重命名或复制文件

更进一步的，在这个目录下新建文件 test.txt

```shell
root@buzz:/home/buzz# cd root_dir/
root@buzz:/home/buzz/root_dir# touch test.txt
root@buzz:/home/buzz/root_dir# ll | grep test.txt
-rw-r--r-- 1 root root    0 Jul  4 17:20 test.txt
```

可以看到新建的文件不能被其他用户修改，但可以被读取

当切换为 buzz 时：

```shell
# 具有 x 权限，可以进入 root_dir 目录
buzz@buzz:~$ cd root_dir/
# 不具有 w 权限，不可以新建文件
buzz@buzz:~/root_dir$ touch buzz_file
touch: cannot touch 'buzz_file': Permission denied
# 不具有 w 权限，不可以删除原有的文件
buzz@buzz:~/root_dir$ rm -rf test.txt
rm: cannot remove 'test.txt': Permission denied
# 不具有 w 权限，不可以重命名文件(移动文件也不行)
buzz@buzz:~/root_dir$ mv test.txt buzz.txt
mv: cannot move 'test.txt' to 'buzz.txt': Permission denied
# 不具有 test.txt 的 w 权限，不可以写入数据
buzz@buzz:~/root_dir$ cd ..
buzz@buzz:~$ vim buzz.txt
buzz@buzz:~$ cat buzz.txt >> root_dir/test.txt
bash: root_dir/test.txt: Permission denied
```

但是一个比较有意思的地方是，因为 root_dir 本身位于 /home/buzz 目录下，即 buzz 对 root_dir 具有 w 权限，所以 buzz 用户可以删除这个文件夹

>   这里实验了一下，发现 ubuntu 下不可以删除，但是 centos 下可以删除 ？？？

当开放目录供他人查看时，应当给予目录 r 和 x 权限，但不应该基于 w 权限，才可以保证正常的浏览，上面 root 用户创建目录时赋予了 x 权限，如果现在回收：

```shell
root@buzz:/home/buzz# chmod 744 root_dir/
root@buzz:/home/buzz# su buzz
buzz@buzz:~$ ls -l root_dir/
ls: cannot access 'root_dir/test.txt': Permission denied
total 0
-????????? ? ? ? ?            ? test.txt
```

虽然有读的权限，但是没有 x 的权限，ls 的结果居然是一堆问号，为了读取(或修改或执行)目录下的文件，至少对这个目录具有 x 权限，只有进入这个目录才可以对文件进行操作

这里进一步的，只要对文件夹具有 x 权限，那么即便没有 r 权限，在进入文件夹后还是可以通过 ls 获取文件夹中的文件信息

## 修改文件的属性

文件的属性包括：文件的权限信息、文件所属的用户、文件所属的用户组

分别通过：`chmod`、`chown`、`chgrp`进行修改

### 修改用户组

要注意的是，修改当前文件的属于其他的用户组时，需要保证用户组的信息已经在 /etc/group 中，这个命令默认不会创建新的用户组

>   chgrp 命令需要具有 root 权限使用

一般的格式如下：

```shell
chgrp [-R] [新的用户组] [文件]
```

其中参数 -R 表示递归修改，如果需要修改的是文件夹的所属的用户组，那么就需要使用这个参数了

举例来说：

```shell
$ sudo chgrp users test.txt
```

### 修改所属用户

类似的需要保证，修改的用户信息已经在 /etc/passwd 中

>   这个命令也需要 root 权限

一般的格式如下：

```shell
chown [-R] [新的用户] [文件]
```

>   参数就不解释了，和上面的一样

命令 chown 可以同时修改用户组和用户信息，一般的格式如下：

```shell
chown [-R] [新的用户.新的用户组] [文件]
```

举例来说：

```shell
$ sudo chgrp root.root test.txt
```

>   将文件的所属用户组修改为 root，同时文件的所属用户也变为了 root
>
>   更进一步的，如果写成 .root 那么将仅修改文件的所属用户组到 root(此时不改变文件的所属用户)

### 修改文件权限

#### 使用数字修改文件权限

* `r`表示4
* `w`表示2
* `x`表示1

>   所以一个用户对文件的最高权限表示为7

举例来说：

```shell
$ chmod 777 test.sh
```

给文件所属用户，文件所属组，及其其他人都赋予了最高权限

#### 使用符号修改文件权限

文件权限针对的三类用户：文件所属用户、文件所属用户组的用户、其他用户，可以通过 u、g、o 三个字符区分

此外使用 a 表示所有的人，那么 chmod 具有如下的形式：

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/chmod_symbol.png)

举例来说：

```shell
# 为文件所属用户配置 rwx 权限，而所属用户组和其他人配置 rx 权限
$ chmod u=rwx,go=rx test.txt 
# 为所有人配置文件的可执行权限
$ chmod a+x test.sh
```

# 目录

## 目录的简单操作

相对路径和绝对路径就不说了，值得注意的是，如果在 '/' (根目录下)，ls -al 也是可以看到 '.' 和 '..' 两个目录的，然而这两个目录的信息完全一致，所以在 linux 中 '/' 下的 '.' 和 '..' 是同一个目录，代表当前目录

*   cd：切换目录

    'cd -' 表示切换到上一次的目录中，返回上一次 cd 前的目录中，连续的 cd - 就是在两个目录中来回切换:laughing: 

*   pwd：显示当前目录(全路径)

    可选参数：-P，如果当前目录是通过软链接进入的 pwd 会打印全路径，但 pwd -P 会打印实际链接到文件的路径

    ```shell
    $ cd ~
    $ mkdir test
    # 创建 test 目录的一个软链接
    $ ln -s test soft_link
    $ ls -al
    lrwxrwxrwx  1 buzz buzz     4 Jul 14 12:25 soft -> test
    drwxr-xr-x  2 buzz buzz  4096 Jul 14 12:26 test
    $ cd soft_link
    # 不加参数 -P 打印的是实际路径
    $ pwd
    /home/buzz/soft
    # 添加参数 -P 打印的是链接到文件的路径
    $ pwd -P
    /home/buzz/test
    ```

*   mkdir：创建目录

    可选参数：

    *   -p：递归创建目录，比如目录下需要创建子目录
    *   -m：设置目录权限，不使用这个参数时使用默认的新建目录的权限，具体的参考 umask

## 目录管理

FHS：FileSystem Heirarchy Standard

FHS 针对目录树架构，定义三层目录下放置的数据：

*   /(根目录)：存放启动系统相关数据
*   /usr(unix software resource)：存放软件安装、执行相关数据
*   /var(variable)：存放系统运行相关数据



# 软链接接和硬链接

先说一下linux中的文件系统，在linux中，文件分为两部分，元数据（metadata）和用户数据（userdata）

其中元数据是文件的附加属性，包含了索引节点（Inode）、文件大小、文件创建时间、文件所有者的信息

然而，上面这么多的信息，就是不包含文件名，文件名主要是为了方便用户使用，真正定位一个文件的其实是索引节点号

> 有点类似ip地址和域名的关系

用户数据是文件数据块，即二进制格式的数据块，记录着文件的真实内容

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/linux_metadata_userdata.png)

如果希望查看文件的节点索引号，使用命令

```shell
ls -i [fileName]
```

比如：

```shell
$ vim test.sh
$ cat test.sh
#!/bin/bash

echo "this is shell script"
$ ls -i test.sh
134241173 test.sh
```

## 硬链接

在linux中允许多个文件名含有同一个索引号

硬链接为通过索引节点号进行的链接，通过指令`ln`可以为文件创建硬链接

```shell
ln [original file] [link file]
```

比如：

```shell
$ ln test.sh hardlink.sh
$ ls -il
total 8
134241173 -rwxrwxrwx 2 buzz buzz 42 Mar 31 11:01 hardlink.sh
134241173 -rwxrwxrwx 2 buzz buzz 42 Mar 31 11:01 test.sh
```

可以看到使用硬链接创建的文件`hardlink.sh`具有和源文件`test.sh`相同的节点索引号

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/linux_hardlink.png)

如上图，既然文件节点索引号都一样了，文件数据肯定也是一样

注意到上面在执行了命令：`ls -il`后，表示文件权限后面多了一个2，这表示当前的硬链接数量

我们删除文件，其实删除的就是硬链接，当一个文件的硬链接数量变为0时，这个文件将被删除，只有当最后一个连接被删除后，文件的数据块及目录的连接才会被释放。

硬连接的作用是允许一个文件拥有多个有效路径名，这样用户就可以建立硬连接到重要文件，以防止“误删”的功能。

## 软链接(符号链接)

软链接相当于创建了一个文件，这个文件的内容就是源文件的节点索引号，通过下面的命令创建软链接

```shell
ln -s [original file] [link file]
```

比如：

```shell
$ ln -s test.sh softlink.sh
$ ls -il
total 4
134241172 lrwxrwxrwx 1 buzz buzz  7 Mar 31 11:19 softlink.sh -> test.sh
134241173 -rwxrwxrwx 1 buzz buzz 42 Mar 31 11:19 test.sh
```

可以看到二者的节点索引是不同的，且源文件大小为42Bytes，而软链接的文件大小仅为7Bytes

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/linux_softlink.png)



和硬链接最大的区别在于如果我们删除了源文件，文件就真的被删除了，此时软链接是无效的

```shell
$ rm -rf test.sh
$ cat softlink.sh
cat: softlink.sh: No such file or directory
```

软链接主要应用在两个方面：

- 一是方便管理，例如可以把一个复杂路径下的文件链接到一个简单路径下方便用户访问；
- 另一方面就是解决文件系统磁盘空间不足的情况。例如某个文件文件系统空间已经用完了，但是现在必须在该文件系统下创建一个新的目录并存储大量的文件，那么可以把另一个剩余空间较多的文件系统中的目录链接到该文件系统中，这样就可以很好的解决空间不足问题；

## 目录链接

上面的操作都是创建关于文件的软链接和硬链接

其实关于目录也可以创建软连接（注意这里不能创建硬链接）

```shell
$ mkdir tempdir
$ ln -s tempdir linkdir
$ cd ./tempdir
$ vim test.sh
$ cat test.sh
#!/bin/bash

echo "buzz is shit"
$ cd ..
$ cd linkdir
$ ./test.sh
buzz is shit
```

我们在源文件夹中创建了一个文件`test.sh`，而在软链接指向的`linkdir`中可以查看文件`test.sh`

## 区别

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/linux_hardlink_softlink.png)

## win下的链接

现在有这样一个需求：

我原来新建了一个`learn_design_pattern`的项目，并在里面新建了一个`README`文档，就是笔记

现在我希望把这个笔记同步到我的`note`文件夹中，同时希望可以实现**同步修改**，不管是我直接在`note`文件夹中写，还是在项目的文档中写，我都希望可以直接同步

第一个思路是使用快捷方式，这个应该行，但是问题出现在我的`note`文件夹中设置了坚果云同步，不确定同步快捷方式是否会出现什么问题

既然软链接不行，那就试试硬链接吧，在`win`中使用：`mklink /?`查看有关链接的指令，具体的参数解释很详细了：

```shell
$ mklink /?
创建符号链接。

MKLINK [[/D] | [/H] | [/J]] Link Target

        /D      创建目录符号链接。默认为文件
                符号链接。
        /H      创建硬链接而非符号链接。
        /J      创建目录联接。
        Link    指定新的符号链接名称。
        Target  指定新链接引用的路径
                (相对或绝对)。
```

# 复制

默认的话只能复制文件，如果希望复制文件夹以及文件夹内部的所有文件：

```shell
cp -r [source] [destination]
```

其他的一些参数：

*   -i：添加这个参数后，复制的时候，如果目标文件 [destination] 已经存在，则会询问是否进行覆盖

*   -p：连同文件的属性(创建时间、权限、用户)一起复制过去(文件备份)

*   -d：如果文件是是一个链接(软链接或硬链接)，则将链接复制过去，而不是复制文件

*   --preserve=all：在 -p 的基础上赋予了一些其他属性

    >   其他属性：SELinux 属性、links、xattr

*   -a：等效于 -dr --preserve=all

*   -l：进行硬链接

*   -s：进行符号链接(软链接)

# Vim

## vimrc

这个位置不是我乱说的，具体的进入 vim 的命令模式使用命令 `:version` 可以查看

```shell
system vimrc file: "$VIM/vimrc"
     user vimrc file: "$HOME/.vimrc"
 2nd user vimrc file: "~/.vim/vimrc"
      user exrc file: "$HOME/.exrc"
       defaults file: "$VIMRUNTIME/defaults.vim"
  fall-back for $VIM: "/usr/share/vim"
```

在 wsl Ubuntu-18.04 中默认是没有环境变量 $VIM 的，在没有额外的配置的情况下 \$VIM 表示为 /usr/share/vim

```shell
root@buzz:/usr/share/vim# ll
total 20
drwxr-xr-x   5 root root 4096 May  5 23:51 ./
drwxr-xr-x 110 root root 4096 May  6 00:13 ../
drwxr-xr-x   3 root root 4096 May  5 23:51 addons/
drwxr-xr-x   2 root root 4096 May  5 23:51 registry/
drwxr-xr-x  17 root root 4096 May  5 23:51 vim80/
lrwxrwxrwx   1 root root    8 Apr 11  2018 vimfiles -> /etc/vim/
lrwxrwxrwx   1 root root   14 Jan 20 10:47 vimrc -> /etc/vim/vimrc
lrwxrwxrwx   1 root root   19 Jan 20 10:47 vimrc.tiny -> /etc/vim/vimrc.tiny
```

这个目录下可以看到一堆符号链接，要么指向了 /etc/vim 下的某个文件(或者是这个目录本身)

修改配置文件，最关键的地方需要修改的是 tab 的缩进，因为是在 linux 下使用 vim 就不修改文件编码格式了(反正都是 utf-8)

```shell
" 设置 tab 为 4 个空格
set tabstop=4
" 设置使用空格替换tab
set expandtab

" 设置语法高亮，默认的话其实不需要设置，因为上面已经设置好了
" syntax on

" 设置允许使用鼠标(其实很难用)
" set mouse=a

" 设置高亮当前行(其实也很难用)
" set cursorline

" 设置全文检索的时候忽略大小写
set ignorecase

" 设置搜索时高亮被搜索的文本
set hlsearch

" 设置自动缩进
set autoindent
```

## vim相关指令

* 复制当前行：`yy`

* 剪切当前行：`dd`

* 删除当前光标下的字符：`x`

* 替换当前光标下的字符：`r`

* 进入替换模式：`R`，在替换模式下，默认会替换光标处的字符，随着输入的进行，光标后移

* 从当前光标的字符处插入：`i`(此后进入插入模式)

* 从当前光标的下一个字符处插入：`a`(此后进入插入模式)

* 粘贴复制的内容：`p`

* 复制若干行：此时需要先进入可视模式（按`v`），然后移动光标选择需要复制的行，按`y`会完成复制；如果希望提前退出，直接`esc`即可

* 撤销：`u`（可以看到，撤销还是很危险的）

* 恢复撤销：`Ctrl + r`

* 全选：本来是`Ctrl + a`即可以全选了，这里使用等效的方式

    * `gg`来到文档开头
    * `G`来到文档结尾

    故可以先`gg`来到文档开头，然后`v`进入可视模式，然后`G`来到文档结尾，即可实现全选

    随后`y`表示复制；`d`表示删除
    
* <kbd>ctrl + f</kbd>：下一页、<kbd>ctrl + b</kbd>：上一页；这两个命令适合大范围移动的时候使用

* 十六进制的格式

    *   查看文件：`:%!xxd`
    *   恢复文件：`:%xxd -r`

    这个需求挺奇怪的，主要是有的文件时二进制的，不可读，转成十六进制好得能分清字节了
    
* 移动到下一个单词：`w`

* 移动到上一个单词：`b`

* 全局搜索：`/`

## 多文件编辑

开启多个文件：

```shell
$ vim [file 1] [file2] [file3] ...
```

在命令上使用：

*   :file：这个命令可以查看已经打开的文本

*   :n：这个命令可以切换到下一个文件

*   :N：这个命令可以切换到上一个文件

    >   有点类似在搜索模式下 n 和 N 的作用

## 多窗口编辑

这个多窗口不仅仅可以针对多个文件实现多窗口，如果只有一个文件，但是这个文件很大，也可以开多个窗口，一个看开头的部分，一个看结尾的部分

进入命令行模式，并使用命令：

```vim
:sp [filename]
```

这个 filename 是一个可选项，如果不写，默认打开的窗口为当前的文件

打开窗口后，通过 <kbd>ctrl + w + j</kbd> 切换下一个窗口，通过 <kbd>ctrl + w + k</kbd> 切换回上一个窗口

>   实测按住 ctrl 后 双击 w 也可以实现切换

默认的 :sp 在上方水平切分窗口，为了实现左右切分，在命令行模式使用：

```vim
:vs(或者 :vsplit) [filename]
```

此外在 vim 中甚至可以使用 terminal

```vim
:terminal
```

>   使用 `:help :terminal` 查看该命令的更多参数

# 打包和压缩

一般而言，文件后缀为 .tar 的为被 tar 打包过的文件(并未压缩)

在 linux 中常见的三种压缩，因为压缩文件：

*   .gz：使用 gzip 进行压缩，一些参数：

    *   -c：将压缩数据输出到屏幕上(简单的 -c 参数会导致不生成 .gz 文件了，因为压缩的结果全都打印出来了)

        所以实际使用的时候：

        ```shell
        $ gzip -c test_file > test_file.gz
        ```

        >   -c 表示将压缩文件的数据打印展示
        >
        >   \> 表示将压缩的数据重定向到 test_file.gz
        >
        >   这两个一起使用，看起来跟两个都没用一样，其实不然
        >
        >   因为 gzip 命令默认将原文件压缩到 .gz 文件中，这导致了压缩后原文件就不见了
        >
        >   同时使用 -c 和 \> 可以使得压缩后原文件还保留

    *   -v：显示原文件/压缩文件的压缩比信息

    *   -[数字 1-9]：这个是压缩等级，压缩等级越高，压缩比越高，但是压缩的速度越慢

    *   -t：校验压缩文件的完整性

    *   -d：解压缩 .gz 文件

    对于压缩后的文件，可以使用 zcat 代替 cat 查看文件内容(当然也可以使用 zless 和 zmore 分别代替 less 和 more)；linux 中使用 grep 可以搜索文本文件内容，而对于压缩后的文件可以使用 zgrep 进行搜索

*   .bz2：使用 bzip2 进行压缩，其参数和 gzip 差不多

    *   -c：不解释了，一个作用
    *   -v：不解释了，一个作用
    *   -[数字 1-9]：不解释了，一个作用
    *   -t：不解释了，一个作用
    *   -d：不解释了，一个作用
    *   -k：保留原始文件(终于不需要 -c 然后再重定向了)

    如果此时希望查看文件内容，使用bzcat(其他的也是一样的)

*   .xz：使用 xz 进行压缩，参数也是类似的

    *   -c：不解释了，一个作用
    *   -v：不解释了，一个作用
    *   -[数字 1-9]：不解释了，一个作用
    *   -t：不解释了，一个作用
    *   -d：不解释了，一个作用
    *   -k：不解释了，一个作用
    *   -l：列出压缩文件的信息

    此时如果希望查看文件内容，使用xzcat

从 gzip 到 bzip2 到 xz 文件的压缩比是逐渐变大的，然而代价就是压缩时间的增加，所以一般 gzip 就行了

要注意的是上面的三个命令是用来压缩文件的，不是压缩目录的，所以实际中更多的是先打包(使用tar) 然后压缩

tar 打包的参数：

*   -c：创建打包文件
*   -t：查看打包文件的内容(内容主要是文件名)
*   -x：解包

可以看到 -c -t -x 三者对应了文件的不同时期(创建打包文件、查看打包文件的内容、解包)，所以这三个不可以同时使用

当然 tar 命令还有一些"公共"的命令：

*   -v：查看正在被处理的文件名
*   -f：后面紧跟需要处理的打包文件名，举例来说，如果 -cf 同时使用，-f 指代的就是创建打包文件后的文件名；如果 -tf 同时使用，-f 指代的就是要查看的打包文件的文件名；-xf 同时使用，-f 指代的就是需要解包的打包文件名

此外上面也提到了，一般打包和压缩同时进行，通过参数选择不同的压缩方式

*   -z：选择 gzip 压缩
*   -j：选择 bzip2 压缩
*   -J：选择 xz 压缩

针对上面打包文件的三个时期，一般而言有以下的命令：

```shell
# 创建压缩包
$ tar -cvz -f [生成的 tarball 文件，以 .tar.gz 结尾] [待打包压缩的文件]
# 比如
$ tar -cvz -f test.tar.gz test
test/
test/p_file
test/p1/
test/p1/p2/
test/p1/p2/p2_file
test/p1/p1_file

# 查看压缩包内的内容
$ tar -tvz -f [待查看的 tarball 文件，以 .tar.gz 结尾]
# 比如(如果去掉 -v 参数，那么仅仅会列出文件名，而无文件的权限信息)
$ tar -tvz -f test.tar.gz
drwxr-xr-x buzz/buzz         0 2022-07-05 20:07 test/
-rw-r--r-- buzz/buzz        15 2022-07-05 19:47 test/p_file
drwxr-xr-x buzz/buzz         0 2022-07-05 19:46 test/p1/
drwxr-xr-x buzz/buzz         0 2022-07-05 19:46 test/p1/p2/
-rw-r--r-- buzz/buzz        16 2022-07-05 19:46 test/p1/p2/p2_file
-rw-r--r-- buzz/buzz        16 2022-07-05 19:46 test/p1/p1_file

# 解压缩
$ tar -xvz -f [待解压的 tarball 文件，以 .tar.gz 结尾]
# 比如
$ tar -xvz -f test.tar.gz
test/
test/p_file
test/p1/
test/p1/p2/
test/p1/p2/p2_file
test/p1/p1_file
```

>   有的在写压缩的时候使用的参数是 -cvzf；解压缩的时候使用的参数是 -xvzf
>
>   这里为了明确待操作的打包文件，故意将 -f 单独写出

## 进阶操作

### 其他参数

其他的一些参数：

*   -p(该参数在打包压缩的时候使用)：保留原文件的属性(权限信息)
*   -P(该参数在打包压缩的时候使用)：保留原文件的绝对路径(这个操作很危险，此时解压)
*   -C(该参数在解包解压缩的时候使用)：-C 后面跟着一个路径，表示将文件在特定的位置解压缩

### 解压部分文件

修正后的命令如下：

```shell
$ tar -xvz -f [tarball 文件] [待解压的文件名]
```

一般的为了知道需要解压那些文件，会先通过 tar -tvz -f 查看一下压缩包内的文件，选择需要的文件，将其路径填入上面的待解压的文件名

### 打包部分文件

通过参数 --exclude=[路径名] 排除那些不想被打包的文件名



# C 的编译过程

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/compile_c_program.png)

由上图可以看到一共四个阶段：

*   预处理过程：c pre-processor(cpp) 根据文件开头的 `#` 修改原程序，比如 #include<stdio.h> 就是把头文件 stdio.h 放入原来的文本文件中，形成了 hello.i
*   编译过程：c compiler(ccl) 将 hello.i 编译为汇编代码得到 hello.s
*   汇编过程：assembler(as) 将 hello.s 汇编为机器指令得到 hello.o，这是一种可重定位目标程序(relocatable object program)
*   链接阶段：linker(ld) 将标准库中的函数(比如 printf 函数) 对应的.o 文件(可重定位目标文件 printf.o) 合并到当前的程序中，得到可执行目标文件

可以手动控制 gcc 的编译过程：

``` shell
$ gcc -S Hello.c # 仅编译，不汇编，不链接；生成 Hello.s 汇编文件，汇编器可以对这个文件进行汇编得到可重定位目标文件
$ gcc -c Hello.c # 完成编译，汇编，但不链接；生成 Hello.o 可重定位目标文件，链接器可以对这个文件进行链接得到可执行目标文件
$ gcc -o Hello Hello.c # 生成可执行目标文件(二进制文件)，名字为 Hello
```

>   ubuntu-18.04 下 .h 文件位于 /usr/include 目录下，链接文件 .so/.a 位于 /lib(或 /lib64) 下

## 链接外部程序

比如现在要使用 linux 系统下自带的 math 库，定义 test.c 如下：

```c
#include<stdio.h>
#include<math.h>

int main(void){
    float val = sin(3.14 / 2);
    printf("%f\n", val);
}
```

test.c 计算了 $\sin(90\degree)$，并将其输出

默认的，直接编译：

```shell
$ gcc -o test test.c
```

这样做不会报错，gcc 会将当前 main 所需要的函数库在链接阶段整合到 test.o 中，所以并不会报错

然而事实上调用 sin 函数位于 libm.so 内，在链接阶段，最好手动加入这个库，这个库默认位于 /lib(或 /lib64) 内，所以正确的编译命令应该是：

```shell
$ gcc -o test test.c -lm -L/lib -L/lib64
```

其中参数 -lm 中 -l 表示(libary) 加入某个函数库，m 表示加入 libm.so 这个函数库；-L 后面跟着路径，表示在对应路径下找函数库

linux 中默认的头文件位于 /usr/include 下，当变更头文件的地址时，也需要使用额外的参数进行修正

>   默认的不填写参数相当于 [-I/usr/include]

# make 和 configure

先说 make，执行 make 指令，程序会在当前目录下寻找 Makefile，这个文件中记录了程序如何进行编译，稍微大一点的程序都不可能只有一个 .c 文件，那么最终编译得到一个可执行目标文件的时候，也许需要链接一堆 .o 文件，有了这个 Makefile，可以通过类似脚本的形式，每次编译的时候直接 make 就行了，不需要手动输入大量的命令

如果 makefile 仅仅是一个脚本，那么 .sh 文件也可以做到，make 的第二个作用是当文件发生变更时，make 仅会更新那些发生了变更的文件(增量编译)，而不再进行全局编译(节省时间)

一般的，当软件以源码的形式安装时，下载的压缩包中并不会直接包含 makefile，而是包括一个 configure.sh 的脚本文件，这个脚本用来检测当前环境中：

*   是否具有合适的编译器可以编译当前软件的源代码
*   是否具有当前软件依赖的函数库和依赖的其他软件

因为不同发行版的 linux，函数库的位置可能有些许差别，所以不同环境下使用 configure.sh 脚本生成的 makefile 可能存在差异

通过源码的形式(tarball)安装软件的流程如下：

1.   官网下载源码(tarball)：一般为.tar.gz 格式

     >   上面的 .tar.gz 表示使用了 tar 进行打包，使用 gzip 进行压缩，所以有的时候可以使用 .tgz 简写后缀
     >
     >   .gz 表示使用了 gzip 压缩，而近些年来发现 bzip2 和 xz 的压缩率更好，所以源码的压缩包的格式还可能是 .tar.bz2 或 .tar.xz

2.   将源码的压缩包解压(建议放到 /usr/local/src 下)

3.   gcc 进行编译得到可重定位目标文件 .o 文件

4.   gcc 对函数库、主程序、子程序链接得到可执行目标文件

5.   其他配置(比如配置环境变量 XXX_HOME 和 PATH)

上面的 3 和 4 一般可以使用 make 进行简化(make 之前通过运行 configure.sh 脚本获得 makefile)

## 使用 make

现在有一个特殊的需求，比如有个程序的源码的结构如下

```shell
.
├── bin
├── c_files
│   ├── fun1.c
│   ├── fun2.c
│   ├── fun3.c
│   └── hello.c
└── head_files
    └── hello.h
```

其中 c_files 保存着 .c 程序，head_files 保存着 .h 程序

fun1.c 为一个求解 sin 函数的程序，fun2.c 为一个求解 cos 函数的程序，fun3.c 仅打印输出；而在 hello.c 中依次调用了这三个函数，所有的函数声明写在了头文件 hello.h 中

现在需要得到可执行目标文件 hello，并将其置于 bin 目录下

首先是传统的命令行编译的方式：

```shell
# 先得到所有的可重定位目标文件
buzz@buzz:~/infamous_application/c_files$ gcc -c fun1.c
buzz@buzz:~/infamous_application/c_files$ gcc -c fun2.c
buzz@buzz:~/infamous_application/c_files$ gcc -c fun3.c
buzz@buzz:~/infamous_application/c_files$ gcc -c hello.c -I ../head_files
# 然后将这些文件链接起来得到可执行目标文件
buzz@buzz:~/infamous_application/c_files$ gcc -o ../bin/hello fun1.o fun2.o fun3.o hello.o -lm -L/lib
```

然后是使用 makefile 的方式，在 c_files 下编写 makefile

```makefile
install: hello.o fun1.o fun2.o fun3.o
    gcc -o ../bin/hello fun1.o fun2.o fun3.o hello.o -lm -L /lib
hello.o: hello.c
    gcc -c hello.c -I ../head_files
clean: *.o
    rm -rf *.o
```

为了得到可执行目标文件，仅需要 make install

对比两种方式，第二种方式显然更加方便，尽管在第一次配置时，需要配置 makefile 但后续升级时，如果不新增文件 那么修改源码后直接 make install 即可完成升级，再也不需要考虑文件之间的约束关系(反正都在 makefile 中配置好了)

## makefile 的简单语法

其一般的形式为：

```makefile
目标(target): 文件1 文件2
<tab> [命令]
```

这里面的目标可以是可执行文件、可重定位文件(比如上面的 hello.o)、或者是标签(比如上面的install 和 clean)

第二行命令前必须有一个 tab，注意这里必须是 tab 如果配置了使用 空格代替 tab 的话 makefile 是不识别的

如果目标是一个可重定位文件，不需要写对应的 .c 文件了，比如上面的：

```makefile
hello.o: hello.c
    gcc -c hello.c -I ../head_files
# 可以写成
hello.o:
    gcc -c hello.c -I ../head_files
```

在 makefile 中是可以定义变量的，比如上面的 makefile 可以改写为：

```makefile
OBJS = hello.o fun1.o fun2.o fun3.o
LIBS = -lm
LIB_DIR = -L /lib
HEAD_DIR = -I ../head_files
install: ${OBJS}
    gcc -o ../bin/hello ${OBJS} ${LIB_DIR}
hello.o:
    gcc -c hello.c ${HEAD_DIR}
clean: *.o
    rm -rf *.o
```

注意引用变量的时候使用：\${[变量名]}$ 的格式

## 使用 make 安装

通过源码(tarball)的方式安装软件，一般而言 linux 中的软件安装在 /usr 目录下，而用户自定义安装的软件放置于 /usr/local 下(只是一种习惯而已，方便管理)

# little trick

*   使用 man ascii 可以得到 ascii 字符码表

# 绕不开的 c

## typedef

这是一个关键字，用来**定义数据类型**

使用的时候和普通的变量定义很类似：

```c
typedef unsigned char * byte_pointer;
```

定义了类型 byte_pointer 他本质上就是 unsigned char *

```c
typedef int * int_pointer;
// 现在定义int 类型的指针有两种写法
int *p;
int_pointer p;
```

当然 typedef 还可以定义数组类型，不过写法有点独特

```c
typedef int row3_t[3]; // row3_t 相当于一个大小为 3 的数组
typedef row3_t * row3_ptr; // row3_ptr 是一个指向了大小为 3 的数组的指针
```

typedef 甚至可以用来定义函数

```c
typedef void(*sighandler_t)(int);
```

这里定义了一个指针 sighandler_t，这个指针指向一个函数，函数的方法返回值为 void，函数的参数为 int

## size_t

本质上是无符号的整数，我在实际操作的时候发现，这个相当于 unsigned long

size_t 是用来表示对象 (objects) 在内存中占用的字节大小的；因此操作符 sizeof 的返回值就是 size_t

## 结构体

看作是一个内部元素类型可互不相同的数组，比如定义一个结构体，表示一个矩形

```c
struct rect{
    long llx;	// low left x
    long lly;	// low left y
    unsigned long width;	// 宽
    unsigned long height;	// 高
    unsigned long color;	// 颜色
};
```

当我们使用结构体时：

```c
// 以 . 的方式访问结构体内部的变量
struct rect r;
r.llx = 0;
r.lly = 0;
r.width = 10;
r.height = 20;
// 使用类似数组的方式初始化结构体
struct rect r = {0, 0, 10, 20, 0xFFFFFF};
```

当然结构体和 typedef 一起使用，看起来更加简洁

```c
typedef struct {
    long llx;	// low left x
    long lly;	// low left y
    unsigned long width;	// 宽
    unsigned long height;	// 高
    unsigned long color;	// 颜色
} rect;
rect r = {0, 0, 10, 20, 0xFFFFFF};
```

结构体中可以通过一些骚操作压缩使用空间

```c
#include <stdio.h>

typedef struct {
    int x : 16,
    	y : 16;
} Emp;

int main() {
    Emp emp;
    emp.x = 10;
    emp.y = 20;
    char* ptr = &emp;
    int i;
    printf("0x");
    for (i = 0; i < sizeof emp; i++, ptr++) printf("%x", 0xff & *ptr);
	printf("\n"); // 0xa0140
    return 0;
}
```

注意到上面定义了一个结构体，具有 int 属性，而被两个变量 x 和 y 平分，其中 x 占用 16 bit，y 占用 16 bit，注意打印输出，看到属性 x 占用的是高 16 位，属性 y 占用的是低 16 位

>   16进制的输出一位对应了 8 bit，所以看到输出的 0 其实对应的是两个 0

## 指针

因为指针指向的是地址的值，所以我发现了，方法的返回值其实可以通过指针传递

```c
#include <stdio.h>
void doMulti(int a, int b, int *rst);
int main(void){
    int a = 10;
    int b = 20;
    int *c = &a;
    doMulti(a, b, c);
    printf("%d\n", *c);
    printf("%d\n", a);
    return 0;
}

void doMulti(int a, int b, int *rst) {
    *rst = a * b;
}
```

可以看到指针 c 指向了存储变量 a 的地址，如果在方法调用时，将指针作为参数传递，那么方法的结果值会保存在指针中，这一点是 java 所不具有的

此外指针还可以指向一个数组，或者说数组本身就是一个指针

### void 指针

看到了一种写法 void *，这其实就是 void 指针

```c
// void 指针可以指向任意类型的数据
int *a;
void *p = a;
```

### 结构体指针

使用之前[结构体](#结构体)中的矩形的那个例子，

```c
// 声明结构体
struct rect *rp;

// 使用结构体，注意必须加上括号
(*rp).width = 10;
// *rp.width 会被 gcc 认为是 *(rp.width)，显然是不对的
// 替代的写法
rp -> width = 10;
```

### 函数指针(function pointer)

顾名思义，就是一个指向了函数的指针，可以向这个指针传递参数，调用方法

举个例子说明一下吧：

```c
int max(int a, int b) {
	return a > b ? a : b;
}

int main(int argc, char** argv) {
	int (*func_ptr)(int, int) = &max;
	return func_ptr(1, 2);
}
```

这里定义了一个比较两个 int 大小的函数 max，同时定义了一个函数指针 func_ptr，在调用函数时，并没有直接调用 max 而是直接调用了函数指针

`int (*func_ptr)(int, int)` 声明了一个具有两个参数，且两个参数都是 int 类型，具有返回值类型为 int 类型的函数指针，这个函数指针名为 func_ptr

因为将 reference max 的赋给了 func_ptr，因此调用 func_ptr 就是调用 max

值得一提的是，因为有回调函数的存在，函数指针也可以作为一个参数传递到函数中，但是这也导致了回调地狱的出现 ...

>   如果一个语言没有 async await 的机制的话，还是慎用回调函数，不然一层套一层，真的很难调试





