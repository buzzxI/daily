# 基本环境配置

## shell

处于美观和易用性考虑, 这里使用 zsh 作为 shell, 使用 [oh-my-zsh](https://ohmyz.sh/) 和 [powerlevel10k](https://github.com/romkatv/powerlevel10k) 进行美化

```shell
$ sudo apt-get install zsh
```

oh-my-zsh 的默认安装是从 github 上 curl 一个 .sh 脚本并运行, 出于各种魔法原因, 不对本地网络进行特殊配置之前大概率是 curl 不动的

索性直接到 oh-my-zsh 的 [github](https://github.com/ohmyzsh/ohmyzsh) 主页中直接复制对应的 install.sh 然后运行

默认的 oh-my-zsh 已经安装了部分插件, 这里可以在 .zshrc 中额外启用常用插件

```shell
# .zshrc

plugins(git vi-mode command-not-found)
```

默认启用了 git, 这里额外启用了 vi-mode(通过 vim 的方式编辑 command) 和 command-not-found(如果输入指令找不到可以给出下载建议)

其他好用的插件比如自动高亮的 [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting) 和 自动补全的 [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions) 需要到 github 主页上手动 clone 后再进行配置, 官方文档具有详细的配置流程

安装 powerlevel10k 的时候选择 [基于 oh-my-zsh](https://github.com/romkatv/powerlevel10k#oh-my-zsh) 安装, 官方贴心的给出了 gitee 的镜像避免魔法问题

## basic command

用了这么多回 linux 其实常用的 command 应该也记得差不多了, 可以去 [Basic Shell Commands in Linux](https://www.geeksforgeeks.org/basic-shell-commands-in-linux/#) 上看一眼有没有不认识的

其余的 command 可以在 [command-not-found.com](https://command-not-found.com/) 上找到一个简单的例子, 比 help 和 man page 更加简洁直观

*   ag: 一个用来找关键字的命令, 默认从文件系统中找到包含特定 pattern 的文件和 pattern 在文件中的位置(带有高亮)
*   ~~[awk](https://www.tutorialspoint.com/awk/awk_quick_guide.htm): 数据处理~~ 建议使用 python 直接正则表达式处理
*   sed: 格式化

# SSH

以前都是使用 ssh 每次验证密码的方式登录，然后服务器的 mysql 就挂掉了，莫名奇妙的

使用 `ssh-keygen` 的方式生成密钥的方式，不仅安全，还可以免密码登录，五星好评

>   github 就是采用 ssh-key 的方式进行登录的，现在还能记得当时配置 github 的时候一脸懵逼的给 github 配置了一个公钥

## 简单猜一下 ssh 是怎么连接的

ssh 客户端可以主动链接到 ssh 服务端, 在客户端生成一对公钥和私钥, 将公钥配置给服务器

这样，每次当客户端发起连接请求时，服务器回响应一个随机的报文, 客户端收到报文后使用私钥进行加密，并重新发送给服务器

## 生成 sshkey

```shell
$ ssh-keygen -t [加密方式] -f [生成的公钥和私钥名称] -C [备注信息] 
```

一般的话使用 `rsa` 进行加密就好了，当然可以查阅其他的你喜欢的加密方式;

>   为了让一台设备可以通过 ssh 连接到多个服务端, 需要在客户端明确私钥, 此时 `-f` 参数不要省略

`-C` 参数，我在网上查看的配置 `github` 的 `ssh` 连接中，基本上都把这个字段写成了自己账户的邮箱，可能是 `github` 约定俗成的吧，毕竟他管理那么多公钥，收到加密消息后总不能使用所有的公钥进行解密吧...

举个例子吧：

```shell
# 首先这个是配置我到腾讯云的ssh连接，所以命名是使用的是tencent_cloud
$ ssh-keygen -t rsa -f ./tencent_keys/tencent_cloud
# 然后这个配置的是github的ssh，所以命名是github
$ ssh-keygen -t rsa -f ./github_keys/github -C '[这里填写具体的邮箱]'
```

它这个默认创建钥匙的目录是在当前用户的 `.ssh` 目录中的

> 所上面的路径也是相对于 `.ssh` 目录的

一个 ssh-keygen 指令会生成一个公钥私钥对, 这里的公钥是带有 `.pub` 后缀的

## 给服务器配置公钥

这里需要修改 `~/.ssh/authoried_key` 文件, 这个文件中容纳了当前用户 ssh 服务端上所有允许的公钥

>   要注意的是，这个 `authoried_keys` 的文件权限最好是 `600`，即当前用户可读可写
>
>   ```shell
>   $ chmod 600 authorized_keys
>   ```

配置好公钥之后最好把密码登录给禁掉:

```shell
$ sudo vim /etc/ssh/sshd_config

PermitRootLogin no #是否允许root登录
PasswordAuthentication no #是否允许密码登录
```

## 客户端管理多个私钥

当可以需要通过 pubkey 的方式连接多个设备时就需要配置了, 需要在客户端的 `.ssh` 目录下新建: `config` 文件, 基础格式如下: 

```shell
# .ssh/config
Host=[随便叫什么名，这里我使用符号server_alias表示]
HostName=[服务器的域名或者ip] # 这个很关键
Port=22 # 这个不写也行，反正ssh默认就是22号端口
User=[想要登录远程服务器的用户名]
IdentityFile=[前面存放私钥的地址] # 这个最关键
PreferredAuthentications=publickey
```

此后在连接 server_alias 时, 只需要:

```shell
$ ssh server_alias
```

如果是配置 github 的话，建议的写法如下: 

```shell
Host=github.com[就必须是这个名字]
HostName=github.com[就必须是这个名字]
PreferredAuthentications=publickey
IdentityFile=[前面存放私钥的地址]
```

测试一下通不通：

```shell
$ ssh -T git@github.com
```

>   事实上 config 不一定完全需要写成 `=` 可以通过类似 yaml 文件的写法, key 与 value 之间通过空格分隔

## reverse tunnel

简单来说, 反向代理可以实现像访问外网设备一样通过 ssh 访问内网设备, 并且不需要进行任何额外的配置, 但凡事总有前提, 至少需要一个具有公网 IP 的设备

>   大概的原理: 内网设备建立一个到具有公网 IP 的设备的 ssh tunnel, 尽管这个 tunnel 是内网设备主动向公网设备建立的, 而后续反而需要利用这个 tunnel 向内网设备通信, 因此这个 tunnel 被称为 `reverse tunnel`
>
>   这样当通过 ssh 连接到具有公网 IP 的设备上, 可以通过某些手段利用这个 tunnel, 反向建立到内网设备的连接

### 内网设备

首先建立 reverse tunnel, 这部分需要在内网设备上进行, 因为需要内网设备建立一个到公网 IP 的 ssh 连接, 这里还是使用 publickey 的方式

>   在内网设备上创建公钥私钥对, 同时完成配置, 至少要先保证可以让内网设备通过 ssh 连接到外网设备

随后通过一个特殊的命令建立一个 ssh reverse tunnel

```shell
# -N 表示建立连接后, 不开启 shell 程序
# -R 表示创建一个 reverse tunnel
$ ssh -N -R 2222:localhost:22 ucloud
```

简单解释以下各个参数:

*   `-N`: 建立 ssh 连接后不要开启 shell 程序
*   `-R`: 建立一个 reverse tunnel, 这个参数后必须要写上端口号; `2222:localhost:22` 表示将 remote server 的 2222 端口上的流量转发到本地的 22 端口上

>   autossh is needed

### 公网 IP

上面那个命令之后在公网设备上就可以利用这个 reverse tunnel 实现到私网设备的 ssh 连接了, 先尝试一下:

```shell
$ ssh -p 2222 [user_name]@localhost
```

上面的命令表示在公网设备上建立一个到自身 2222 端口的 ssh 连接, 由于刚才的命令已经将去往 2222 的流量转发到内网设备的 22 端口上了, 这样就可以反向连接了

要注意的是, 这里是通过 [user name]@[ip] 的方式建立 ssh 连接的, 因此还是需要输入密码的, 既不安全也麻烦, 这里还是一样将其优化为使用 publickey 的方式连接

这里需要在公网设备上建立公钥和私钥对, 配置公钥的方式和普通的 ssh 公钥相同, 唯一的不同在于在公网设备的 config 中有一点点不同

```shell
# .ssh/config from server
Host [alias]
	HostName localhost
    User [target user]                                                                                               
  	PreferredAuthentications publickey
    IdentityFile [path/to/identity/file]
    # important !!!
    ForwardAgent yes
```

>   注意到最后一行添加了 `ForwardAgent yes`, 表示允许进行转发, 这里是为了服务后续外网设备之间连接到内网设备准备的

此后在公网设备上就可以通过 host alias 连接到内网设备了

```shell
$ ssh -p 22222 raspberry
```

### 一般设备

终于到这一步了, 当一个设备需要访问内网设备, 首先让当前设备可以访问到外网设备, 还是一样的, 配置一个当前设备到外网设备的 publickey

```shell
# .ssh/config from my laptop
Host [server_alias]
	HostName [server ip]
	User [server user]
	Port 22
	PreferredAuthentications publickey
    IdentityFile [path/to/identity/file]
```

此外还需要配置一个当前设备到内网设备的 publickey, 配置公钥的方式和上面的 ssh 公钥基本类似, config 文件的写法有一点点不同

```shell
# .ssh/config from my laptop
Host [alias]
	HostName localhost
    User [target user]
    Port 2222
    PreferredAuthentications publickey
    IdentityFile [path/to/identity/file]
    ProxyJump [server_alias]
```

注意到这里添加了一个字段 ProxyJump, 其 value 为之前配置的 server Host, 表示设备建立到内网设备的 ssh 连接时, 以 server_alias 为中继

>   之前在 server 到内网设备的 ssh 连接的时候配置了允许 `ForwardAgent`, 即允许中继

现在通过 `ssh alias` 即可完成从当前设备到内网设备的 ssh 连接

### enhanced tools

当网络不稳定的时候, ssh tunnel 可能中断, 此时最好使用 `autossh` 进行断网后自动重连

```shell
# install autossh
$ apt-get update
$ apt-get install autossh
# use autossh
$ autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -N -R 22222:localhost:22 ucloud
```

*   `-M 0`: 这个是 chatgpt 教我写的, 不知道为啥
*   `ServerAliveInterval 30`: 表示私网设备每 30 s 向外网 server 发送一个心跳包
*   `ServerAliveCountMax 3`: 表示最多连续收不到 3 个心跳包, 超过了, 就重连

# 开启自启动

比如启动 clash, 比如启用 ssh reverse tunnel

首先写一个 shell 脚本, 用来启动各个程序

```shell
# startup_script.sh

#!/bin/zsh

# tmux for clash
tmux new -d -s clash '/root/clash/clash -d /root/clash'

# tmux for reverse tunnel
tmux new -d -s ssh-tunnel 'autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -N -R 22222:localhost:22 ucloud'
```

十分简洁明了, 这个脚本并没有特别高级, 就是简单的命令的堆叠 (不要忘了 `chmod +x` 为其赋予 可执行权限)

随后, 使用 systemctl 对运行的各个脚本进行管理

>   原理未知, 反正能用

在 `/etc/systemd/system` 目录下创建一个 `startup_script.service` 文件, 表示脚本服务

```shell
# startup_script.service
[Unit]
Description=Startup Script
After=network-online.target ssh.service

[Service]
Type=simple
ExecStart=/usr/local/bin/startup_script.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

*   Description: 当前 service 的描述, 随便写
*   After: 用来指定当前的服务的启动顺序在 network-online 和 ssh 之后, 保证当前 service 启动时, ssh 和 network 都已经准备好了
*   Type: simple, 这里写成 simple 就好
*   ExecStart: 这个对应的是执行脚本的路径
*   RemainAfterExit: 写成 yes 就好
*   WantedBy: 写成 multi-user.target

随后需要让 systemd 管理当前 service:

```shell
$ systemctl daemon-reload
$ systemctl enable startup_script.service
```

到这里其实就可以了, 脚本应该就可以开机启动了, 但还是有一些需要注意的:

*   手动启动服务: `systemctl start startup_script.service`
*   手动关闭服务: `systemctl start startup_script.service`
*   查看服务状态: `systemctl status startup_script.service`

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

先说一下 linux 中的文件系统，在 linux 中，文件分为两部分，metadata(文件的属性数据) 和 userdata(文件中实际保存的数据)

关于 metadata 更多的信息可以看 [metadata](./CSAPP.md#metadata)，可以看到 metadata 中是不包含文件名的，文件名主要是为了方便用户使用，真正定位一个文件的其实是索引节点号

> 有点类似ip地址和域名的关系

用户数据是文件数据块，即二进制格式的数据块，记录着文件的真实内容

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/linux_metadata_userdata.png)

如果希望查看文件的节点索引号，使用命令

```shell
ls -i [fileName]
```

比如：

```shell
$ ls -i echo.sh
134241173 echo.sh
```

## 硬链接

在 linux 中允许多个文件名含有同一个索引号, 这样创建的不同文件指向了一个相同的 v-node, 这样创建的就是硬链接

通过指令 ln 可以为文件创建硬链接

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

可以看到使用硬链接创建的文件 hardlink.sh 具有和源文件 test.sh 相同的节点索引号

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/linux_hardlink.png)

如上图，既然文件节点索引号都一样了，文件数据肯定也是一样

注意到上面在执行了命令: ls -il 后，表示文件权限后面多了一个2，这表示当前的硬链接数量

删除文件，其实删除的就是硬链接，当一个文件的硬链接数量变为 0 时，这个文件将被删除，只有当最后一个硬链接被删除后，文件的数据块及目录的连接才会被释放

硬链接的作用是允许一个文件拥有多个有效路径名，这样用户就可以建立硬链接到重要文件，以防止“误删”的功能。

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

可以看到二者的节点索引是不同的，且源文件大小为 42 Bytes，而软链接的文件大小仅为 7 Bytes

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

上面的操作都是创建关于普通文件的软链接和硬链接，其实关于目录也可以创建软连接(注意这里不能创建硬链接)

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

我们在源文件夹中创建了一个文件 test.sh ，而在软链接指向的 linkdir 中可以查看文件 test.sh

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

*   单词为单位移动: 
    *   上一个单词的开头: `b`
    *   下一个单词的开头: `w`
    *   下一个单词的结尾: `e`

* 插入:

    *   从当前光标字符插入: `i`
    *   从当前光标的下一个字符处插入：`a`

* 替换:

    *   替换当前字符: `r`
    *   替换当前字符的大小写: `~`
    *   替换到行尾(可以理解为从当前位置一直删到这一行结束, 随后进入插入模式): `C`

* 删除:

    *   剪切当前行: `dd`
    *   删除当前光标下的字符: `x`
    *   删除到行尾(和上面的删除到行尾不太一样, 这里结束删除后还是普通模式): `D`

* 复制一行: `yy`

* 粘贴复制的内容：`p`

* 撤销: `u`

* 恢复撤销: `Ctrl + r`

* 基于 vision 模式的各种神奇操作(`esc` 会退出 vision 模式):

    *   字符模式: `v` 使用最多的模式, 简单的选中
    *   行模式: `V` 选中了一行, 用的不太多
    *   块模式: <kbd>Ctrl + v(V)</kbd> 这个键位有的时候会有冲突, 比如在 windows terminal 中不管是什么模式都是直接复制, 所以需要改键, 我是直接删除了 terminal 中的 <kbd>Ctrl + V</kbd> 作为复制的快捷键

    *   复制若干连续的内容: 首先进入字符, 然后移动光标覆盖需要复制的部分, 最后按 `y`;
    *   剪切若干连续的内容: 前面的部分都一样, 只不过最后按 `d`
    *   全选: `gg` 来到文件开头(或者 `G` 来到文件结尾), 然后进入字符模式, 最后 `G` 来到文件结尾(或者 `gg` 来到文件开头)
    *   为连续的多行同时添加某个字符, 首先通过 <kbd>ctrl + v</kbd> 进入块模式选中多行, 然后使用 <kbd>shift + i</kbd>, 即可同时编辑多行(或者使用 <kbd>shift + a</kbd> 在当前位置的后面添加)

* <kbd>Ctrl + f</kbd>：下一页、<kbd>Ctrl + b</kbd>：上一页；这两个命令适合大范围移动的时候使用

* 替换: 在命令行模式下使用 `:[range]s/[from]/[to]/[flag]`, 其中 from 表示需要被替换的内容, to 表示需要替换为的内容, flag 一般写为 `g`, 至于 range 其取值有很多可能:

    *   `21`(一个行号): 表示只替换 21 行的内容

        >   因为行号从 1 开始, 因此 1 表示替换第一行

    *   `$`: 表示只替换最后一行的内容

    *   `21,25`: 表示替换从 21 行到 25 行的内容

    *   `.`: 表示当前行

    *   `.5`: 表示从当前行下的第五行, 所以 `.` 其实就是 `.0` 的意思

    *   `%`: 表示替换全文, 等价于 `1,%`

    *   `'<,'>s`: 当使用 vim 选中块之后, 会自动添加 `'<,'>`, 此后添加 flag 's' 即可进行替换

* 搜索: `/`, 随后使用 `n` 查看后一个匹配位置(`N` 查看前一个匹配位置), 默认情况下我都是开启了忽略大小写的, 在某些情况下, 如果希望分开大小写的情况, 可以通过命令行的方式:

    ```shell
    :set ignorecase!
    ```

    或者临时通过参数表示大小写敏感:

    ```shell
    /pattern\C # match 'pattern' only 
    ```

* pattern 统计, 有了搜索, 当然希望知道 pattern 到底出现了多少次, 这里需要使用的还是 `:s` 命令

    ```shell
    :%s/[pattern]//n
    ```

    其中 `%` 表示全局统计, 而将 flag 设置为 `n` 表示统计字数, 而不是执行替换, 此外类似 `/` 可以通过 `n`(N) 查看前一项和后一项

* 十六进制的格式, 这里主要是为了查看二进制文件

    *   查看文件：`:%!xxd`
    *   恢复文件：`:%!xxd -r`

    在上述指令中, `%` 表示整个文件的内容, `!` 表示调用一个外部程序, `xxd` 是一个用来将二进制文件转化为十六进制的工具, 其中参数 `-r` 是作用于 `xxd` 的, 即让 `xxd` 将十六进制转化为二进制
    
* 文本过滤: 比如需要仅保留符合某些字符匹配的文本, 此时可以通过命令过滤: `:g/[pattern]/d` 这里默认是在全局(g)删除掉(d)符合 pattern 的行, 如果希望保留, 那么直接取反即可: `:g!/[pattern]/d`

* 文本折叠: 

    *   启动折叠: `:set nowrap`
    *   取消折叠: `set wrap`

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

## registers

vim 使用 registers 保存被剪切的文本, vim 内置了多个 registers:

在不考虑 registers 时, 使用 `dd` 进行剪切, 使用 `yy` 进行复制, 使用 `p` 进行粘贴

在考虑到寄存器后, 使用 `"[register name]dd` 将剪切的结果保存特定名字的寄存器中, 使用 `"[register name]yy` 将复制的结果保存特定名字的寄存器中, 使用 `"[register name]p` 从特定名称的寄存器中粘贴 

*   numbers registers: 编号 0 ~ 9, 默认的最新被复制/剪切的文本被保存在 register 0 中, 而上一条记录保存在 register 1 中, 以此类推, 所以这里也很显然了, 默认一直使用的就是 register 0

    默认使用的命令: `dd` 其实等价于: `"0dd`

*   named registers: 编号 'a - z' 和 'A - Z', 比如使用 `"ayy` 将结果保存在寄存器 a 中, 使用 `"ap` 从寄存器 a 中复制结果

*   black hole registers: 顾名思义, 将会丢弃所有的结果, 因此一般使用的时候都是: `"_dd` 表示删除 (不剪切)

注意到上面的这些操作是不区分 visual mode 还是 normal mode 的, 在 visual mode 下, 通过 `"ad` 对选中的块进行剪切操作, 并将结果保存在寄存器 a 中

>   使用 `:register` 可以将各个寄存器中保存的 value 打印出来

## 奇淫技巧

*   在 windows 中习惯了使用 <kbd>Ctrl + s</kbd> 保存文件, 在切换到 vim 环境之后还是有这个习惯, 而在 linux 中 <kbd>Ctrl + s</kbd> 会让当前进程暂停, vim 中就直接卡死了, 这个时候不用直接退出当前的 terminal, 直接 <kbd>Ctrl + q</kbd> 就可以恢复当前进程了

*   有的时候使用 vim 重新打开刚刚编辑过的文件会直接回到文件顶部, 而不是回到上次编辑过的地方, 为了配置每次回到上次编辑过的地方, 可以修改 `/etc/vim/vimrc` 文件:

    ```shell
     " Uncomment the following to have Vim jump to the last position when
     " reopening a file
     " au BufReadPost * if line("'\"") > 1 && line("'\"") <= line("$") | exe "normal! g'\"" | endif
    ```

    找到上述位置, 注释已经写明白了, 如果希望回到上次编辑过的位置, 直接把注释解开就行
    
*   resume a process: 在 linux 中编辑文本的时候, `ctrl + z` 会中断当前进程, linux 会返回一个 job number 表示被中断的进程

    ```shell
    # view all jobs managed by current shell
    $ jobs
    # resume job by job number
    $ fg %[job_number]
    ```

    类似的, 当使用 & 使得进程运行在后台, 其还是被 shell 的 jobs 管理, 此时也可以通过上述方式将进程运行到前台

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

## malloc&calloc

下面的内容来自 chatgpt:

Both `malloc` and `calloc` are functions used to allocate memory dynamically in C and C++. However, there are some differences between them.

`malloc` is a function that allocates a block of memory of a specified size and returns a pointer to the first byte of the block. The contents of the block are not initialized, and the block may contain garbage values.

`calloc`, on the other hand, is a function that allocates a block of memory of a specified size and initializes all the bytes to zero. The return value is also a pointer to the first byte of the block.

So the main difference between `malloc` and `calloc` is that `calloc` initializes the allocated memory to zero, while `malloc` does not. This can be important in some cases, such as when working with arrays or structures that need to be initialized to zero.

Another difference is that `calloc` takes two arguments: the number of elements to allocate and the size of each element, while `malloc` takes only one argument, the total size of the block to allocate.

For example, to allocate space for an array of 10 integers using `calloc`, you would use:

```c
int *ptr = calloc(10, sizeof(int));
```

And to allocate the same space using `malloc`, you would use:

```c
int *ptr = malloc(10 * sizeof(int));
```

Overall, `calloc` is often used when you want to initialize the allocated memory to zero, while `malloc` is used when you just need to allocate memory and don't care about its initial contents.

## macro

参考内容来自: [Macros (The C Preprocessor) (gnu.org)](https://gcc.gnu.org/onlinedocs/cpp/Macros.html#Macros)

### concatenate

将两个 token 合并为一个 token, 比如:

```c
#include <stdio.h>

#define CONCATENATE(a, b) a##b

int main() {
    int xy = 42;
    printf("%d\n", CONCATENATE(x, y));  // This will be equivalent to xy
    return 0;
}

```

在写 craft interpreter 中的 clox parser 的时候用到了这个写法, parser 在解析到 variable 时, 会根据下一个 token 的类型动态生成 get opcode 或者 set opcode

此外 variable 还分为了 global variable 和 local variable 的; opcode 的大小可能是一个字节长的, 还可能是两个字节长的, 综上一共有 8 种组合, 借助 concatenate 可以简化写法

```c
// chunk.h

typedef enum {
    CLOX_OP_GET_GLOBAL,
    CLOX_OP_GET_GLOBAL_16,
    CLOX_OP_SET_GLOBAL,
    CLOX_OP_SET_GLOBAL_16,
    CLOX_OP_GET_LOCAL,
    CLOX_OP_GET_LOCAL_16,
    CLOX_OP_SET_LOCAL,
    CLOX_OP_SET_LOCAL_16,
} OpCode;

// compiler.c

static void variable(Compiler *compiler, bool assign) {
#define PARSE_VARIABLE(operation, idx, scope) do {\
        if ((idx) > UINT8_MAX) emit_bytes(compiler->parser, 3, CLOX_OP_##operation##_##scope##_16, (idx) >> 8, (idx) & 0xff);\
        else emit_bytes(compiler->parser, 2, CLOX_OP_##operation##_##scope, (idx));\
    } while(0);
    
    int local_idx = resolve_local(compiler->resolver, compiler->parser->previous);
    int global_idx = 0;
    if (local_idx == -1) global_idx = make_constant(OBJ_VALUE(new_string(compiler->parser->previous->lexeme, compiler->parser->previous->length)), compiler->parser); 
    if (assign && match(compiler->parser, 1, CLOX_TOKEN_EQUAL)) {
        // set variables
        if (local_idx != -1) {
            // local set
            PARSE_VARIABLE(SET, local_idx, LOCAL);
        }
        else {
            // global set
            PARSE_VARIABLE(SET, global_idx, GLOBAL);
        }
    } else {
        // local get 
        if (local_idx != -1) {
            // local get
            PARSE_VARIABLE(GET, local_idx, LOCAL);
        }
        else {
            // global set
            PARSE_VARIABLE(GET, global_idx, GLOBAL);
        }
    }

#undef PARSE_VARIABLE
}
```

parser 的最终目的是生成字节码, 上面的 8 种字节码可以根据 scope, operation, length 分类, 宏 PARSE_VARIABLE 根据刚刚提到的 3 个参数动态生成 8 种字节码指令

>   如果没有这个宏, 那么 emit_bytes 需要重复超级多次

## makefile

一套带走 makefile -> [Makefile Tutorial By Example](https://makefiletutorial.com/)

### simple example

```shell
$ vim makefile
hello:
	echo "hello, makefile"
	
$ make
echo "hello, makefile"
hello, makefile
```

在使用 makefile 的时候需要使用 **<kbd>Tab</kbd>** 作为缩进, 而不是空格, 否则 make 会报错

### syntax

makefile 的基本语法如下:

```makefile
target: prerequisites
	command
	command
	...
```

target 对应的是 make 需要生成的文件名通常只有一个

prerequisite 为生成对应 target 文件需要的依赖文件名, 多个文件名之间空过空格分割

command 为生成对应 target 文件需要执行的各种命令

在上面的例子中 hello 就是 target, 这里要说明的是, 在 make 的当前目录下不存在名为 hello 的文件时, `make hello` 才可以正常工作, 如果存在那么 make 不会执行任何操作

```shell
$ vim hello
$ make hello
make: 'hello' is up to date.
```

一般而言 target 就是一个需要生成的 file, target 下的各个 command 就是用来生成这个 file 的，所以在说到 makefile 的时候会将 target 和 file 混用, 但也有例外, 在 make file 中会存在一个名为 `clean` 的 target, `clean` 是用来批量删除文件的, 此时的 `clean` 并不是一个 file

之前使用 make 的时候, 也遇到过 make 不携带参数的情况, 此时默认执行当前目录下的第一个 target

### essence of make

前面提到了 make 的作用就是生成 target, 只要当前目录下存在 target, 那么此时 make 不会执行 target 下的 command, 这句话其实是有问题的, 一个完整的 make 命令中还包含了 prerequisite, **只要 prerequisite 中的任何一个文件比 target 更新**, 那么 make 还是会执行后续 command

>   在 linux 中文件的 metadata 中包含了文件的修改时间信息, make 通过读取 target 和 prerequisite metadata 中的时间判断当前是否需要重新生成 target

不过文件的修改时间是可以被篡改的, 修改之后 make 就无法实现正确的编译了

### make clean

上面也说过了一般而言 `make clean` 是为了删除某些 target(file) 而存在的, 不过 clean 本身在 makefile 中并不是一个关键字, `clean` 的删除特性也不过是人们约定好的

习惯上:

*   `clean` 不会是 makefile 中的第一个命令

    >   因为 makefile 的主要目的是为了生成而不是删除, 所以一般第一个命令生成的都是 executable objective file(直接可以运行的文件), 这样人家一个 `make` 就可以直接运行了

*   如果在项目路径中确实存在名字为 `clean` 的文件, 那么此时为了避免歧义 -> 生成名字为 `clean` 的 file 还是执行删除操作, 此时可以添加 `.PHONY` 进行修正

### make all

类似于 `make clean` 的存在, 人们习惯上将 `all` 作为 makefile 中的第一个 target, `make all` 会生成若干个中间文件和一个可执行目标文件, 不过 `all` 也不是 makefile 中的关键字, 因此也会出现如果项目路径下存在名为 `all` 的文件时就无法正常工作的情况, 此时也需要通过 .PHONY 进行修正  

### .PHONY

对一个 target 添加了 .PHONY 可以让 make 避免将对应的 target 认为是一个 file, 这样执行对应的 target 语句的时候 make 都会执行一次

```makefile
some_file:
	touch some_file
	touch clean

.PHONY: clean
clean:
	rm -f some_file
	rm -f clean
```

在上面的例子中 some_file 会在项目的根路径下添加 clean 文件, 如果没有 `.PHONY` 进行标记, 那么此时执行 `make clean` 时,  make 会因为路径中已经存在了名为 `clean` 的文件而不执行删除操作

在一个规范的 makefile 中应该在 `all` 和 `clean` 前都添加 `.PHONY` 进行标识

### variable

在 makefile 中可以定义变量并通过 \${variable} 或 \$(variable) 进行引用, 不过和一般编程语言不同的是, makefile 中具有四种赋值方式:

*   `:=`(simply expanded) 最贴合直觉的赋值方式, 和一般的语言中的赋值具有相同含义, 此时会根据前面已经定义好的变量进行赋值, 而如果变量定义在后面则无法赋值

    ```makefile
    # normal assign
    variable_emp := emp
    # assign a with emp
    a := ${variable_emp}
    ```

    ```makefile
    # abnormal assign
    # a cannot be assigned with tmp
    a := ${variable_emp}
    variable_emp := tmp
    ```

*   `=`(recursive) 递归赋值, 可以认为此时的赋值发生在命令运行期间, 而不是定义期间, 可以认为是一种延迟赋值, 此时后面的值会覆盖前面的值

    ```makefile
    a = old
    b = ${a} word
    a = new
    
    .PHONY: all
    all: 
    	echo ${a} 
    	echo ${b}
    ```

    此时 echo \${b} 得到的是 `new word`

*   `?=` 在前面没用定义过的时候可以进行赋值

    ```makefile
    a ?= old
    a ?= new
    
    .PHONY:all
    all:
    	echo ${a}
    ```

    由于只有在未定义的时候赋值, 因此上述 `make all` 的输出为 old

*   `+=`(append) 这个赋值符号的定义和一般编程语言中的 `+=` 类似, 标识在当前变量后面额外添加

    ```makefile
    a := old
    a += new
    
    .PHONY: all
    all:
    	echo ${a}
    ```

对于字符串类型的变量特别要注意空格的影响, 在变量的定义/赋值时字符串结尾的空格会被拼接, 而字符串开头的空格会被忽略掉

```makefile
a := test space      # a end with 6 space
b := ${a} lala # b begin with a space, followed by a and lala

.PHONY: all
all:
	echo ${b}
```

上述 b 的输出中具有 7 个空格, 但如果更换一下

```makefile
a := test space
b :=      lala ${a} # b begin with a space, followed by a and lala

.PHONY: all
all:
	echo ${b}
```

此时输出的 b 中不会包含任何前置空格

在 makefile 中字符串引号 `''` 和 `""` 没有什么特殊含义, 会被认为是普通的字符, 因此在 makefile 中没必要额外使用引号, 此外尽管 $variable 有的时候也可以引用变量, 但不建议这么写

一个规范的 makefile 中的变量不要擅自添加引号, 引用变量的时候最好添加上 {} 或者 ()

#### automic variable

在 makefile 的规范中定义了很多 automic variable, 但这里只说常用的几个:

*   `$@` command 中的 `$@` 指代的是所有的 target

    ```makefile
    all: foo1.o foo2.o
    
    foo1.o foo2.o:
    	echo $@
    # equals to
    # foo1.o:
    # 	echo foo1.o
    # foo2.o:
    # 	echo foo2.o
    ```

    上述 `$@` 指代了两个 target, 所以带有 `$@` 的 command 运行次数和 target 的个数相关

*   `$^` command 中的 `$^` 指代的是所有的 prerequisite

*   `$?` command 中的 `%?` 指代的是所有比 target 更新的 prerequisite

#### implicit variable

默认 make 可以用来编译 c/c++ 代码, 在 make 内部设计了一些特定变量专门用来编译 .c 或 .cpp 文件

*   `CC`: c 的编译器, 默认是 `cc`, 可以修改为 gcc 或者 clang
*   `CXX`: c++ 的编译器, 默认是 `g++`, 可以修改为 clang++
*   `CFLAGS`: flags for c complier
*   `CXXFLAGS`: flags for c++ complier
*   `CPPFLAGS`: flags for c preprocessor
*   `LDFLAGS`: flags for compilers when they are supposed to invoke the linker

将一个 .c 文件编译为一个 .o 文件可以写成:

```makefile
target.o: target.c
	${CC} -c ${CFLAGS} $^ -o $@
```

比如:

```makefile
CC := gcc

.PHONY: all
all: main.o

main.o: main.c
	${CC} -c ${CFLAGS} $^ -o $@

main.c:
	echo 'int main() { return 0; }' >> main.c
	
.PHONY: clean
clean: 
	rm -rf main.*
```

而将一个 .cpp 文件编译为一个 .o 文件可以写成:

```makefile
target.o : target.cpp
	${CXX} -c ${CXXFLAGS} $^ -o $@
```

因为上述模板实在时太固定了, 因此在生成 .o 文件的时候, make 默认会找到 .c 或者 .cpp 文件根据上面命令编译源码

```makefile
.PHONY: all
all: main.o

main.c: 
	echo 'int main() { return 0; }' >> main.c

.PHONY: clean
clean: 
	rm -rf main.*
```

#### *wildcard

一种 makefile 的匹配模式

```makefile
# use wildcard function to match all .c files under source dir
SRC_FILE := $(wildcard ${SRC_DIR}/**/*.c)	
```

### string subsitution

`$(patsubst, pattern, replacement, text)$`, 字符串替换

```makefile
```



## constructor/deconstructor

c 语言中并没有类, 但在 c 中存在类似 `constructor` 和 `deconstructor` 的说法, 只不过这里围绕的是 main 函数, 其中 constructor 会在 main 之前调用, 可以用来执行一些初始化操作, 而 deconstructor 会在 main 结束后调用用来执行一些资源释放操作

```c
// example.c
#include <stdio.h>

static inline void __attribute__((constructor)) my_constructor(void) {
    // Initialization code
    printf("Constructor called\n");
}

static inline void __attribute__((destructor)) my_destructor(void) {
    // Cleanup code
    printf("Destructor called\n");
}
```

不管是 constructor 还是 deconstructor 都是没有返回值的

## inline

这个关键字是用在方法上的, 比如:

```c
// test.c

inline int add(int a, int b) {
    return a + b;
}

//test.h, 注意在 .h 文件中使用 inline 的时候需要将方法声明为 static
static inline int add(int a, int b) {
    return a + b;
}
```

inline 关键字是用来告知编译器将这里的代码进行展开的 -> 将函数调用展开为代码

不过现代编译器并不会严格意义上执行 inline 的行为, 在不同的优化级别上编译器会做出不同的选择, 有的时候添加了 inline, 编译器也不会将其展开, 有的时候不添加 inline, 编译器也会将其展开, 在代码中添加 inline 其实也只是向编译器添加了一个 flag, 表示我希望这里是展开, 但也只能是希望了 ...

# CMake

构建工具

## miniumum cmake

首先创建项目路径, 在项目跟目录下随便写个 main.cpp

使用 cmake 的关键在于 `CMakeLists.txt`, 最简形式:

```cmake
# 指定最小的支持的 cmake 版本
cmake_minimum_required(VERSION 3.10)
# 指定需要创建的项目名称
project(main)
# 告诉 cmake 使用什么样的文件创建可执行目标文件
add_executable(main main.cpp)
```

对于本例子而言 `CMakeLists.txt` 放在项目的根目录下, 习惯上会在 `[项目根路径]/build` 中进行项目构建, 所以一个使用了 cmake 的项目具有的最简格式为:

```ascii
├── CMakeLists.txt
├── build
└── main.cpp
```

进入 `./build`, 执行 cmake 生成构建目录, 在本例中就是 makefile

```bash
$ mkdir build
$ cd build
# 这里使用 `cmake ../`, 因为要找到根目录下的 CMakeLists.txt, 以此生成 makefile
$ cmake ../
```

>   更加简洁的写法是: mkdir 后直接 `cmake -S . -B build`
>
>   参数 `-S` 后面紧跟的是 source 的路径, 默认就在项目根目录下, 因此直接写 `.` 就行
>
>   参数 `-B` 后面紧跟的是 build 的路径, 默认放在 build 中, 因此直接写 `build` 就行
>
>   cmake-GUI 中配置的 souce 和 build 不过也就是这两个路径吧

随后生成的 makefile 完成项目构建

```bash
$ cd build
$ cmake --build ./
```

>   更加简洁的写法是: 直接在根目录下 `cmake --build build`
>
>   不带参数的 cmake 命令用来生成了 makefile, 而带有 `--build` 标志的 cmake 可以完成项目构建

## add library

在实际的项目中, main 中进行的方法调用, 其实现大都写在了其他的文件中, 为了减少 main 和各个方法之间的耦合, 在 cmake 中可以将其调用的方法抽取为一个 library, 分开构建直到链接的时候才耦合

### simple library

假设当前项目中同时存在 `main.cpp` 和 `func.cpp`, `main.cpp` 调用了 `func.cpp` 中的函数, 那么此时就可以进行 library 的抽取

```cpp
// func.hpp
namespace func {
    int simple_func();
}

// func.cpp
#include "func.hpp"

namespace func {
    int simple_func() {
        return 10;
    }
}

// main.cpp
#include <iostream>
#include "func.hpp"

int main(int argc, char** argv) {
    std::cout << "rst is:" << func::simple_func() << std::endl;
    return 0;
}
```

此时修改 `CMakeLists.txt`, 将 func 进行抽取

```cmake
cmake_minimum_required(VERSION 3.10)

project(main)
# 添加了库 libfunc, STATIC 将其标记为静态库, 这个库需要 func.cpp 编译得到
add_library(libfunc STATIC func.cpp)

add_executable(main main.cpp)
# 为可执行目标文件添加链接 libfuc
target_link_libraries(main libfunc)
```

### add subdirectory

在实际的项目中是不会将所有的 library 放入一个目录的, 全部堆积在项目的根目录下也不方便管理, 因此更多的写法是将 library 放在一个单独的目录下, 此时项目具有如下的目录结构

```ascii
├── CMakeLists.txt
├── build
├── func
│   ├── CMakeLists.txt
│   ├── func.cpp
│   └── include
│       └── func
│           └── func.hpp
└── main.cpp

```

注意到 `func.cpp` 和 `func.hpp` 已经放入了 `./func` 下, 并且 `func.hpp` 更进一步放入了 `include` 中, 此外在 `/func` 中还额外存在了一个 `CMakeLists.txt`

分别解释一下两个 `CMakeLists.txt`:

```cmake
# ./func/CMakeLists.txt
# 添加了一个静态库 func, 通过 func.cpp 编译得到
add_library(libfunc STATIC func.cpp)
# 添加了一个 include path, 为当前目录下 ./include
target_include_directories(libfunc PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
```

>   注意到上面的 `target_include_directories` 中包含了一个变量: CMAKE_CURRENT_SOURCE_DIR, 这个变量对应的是当前 `CMakeLists.txt` 所在的文件路径

```cmake
# ./CMakeLists.txt
cmake_minimum_required(VERSION 3.10)

project(main)
# 为当前的项目添加了一个编译路径 ./func
add_subdirectory(func)

add_executable(main main.cpp)
# 在 linking 阶段为当前项目 link libfunc
target_link_libraries(main libfunc)
```

### library for private

出于模块化设计的考虑, 在 main 中其实并不关心 libfunc 的实现, 一般而言 libfunc 是需要依赖一些系统库函数的, 比如现在 func.c 中需要从文件中读取到一个数字作为输入

```c
// func/include/func/func.h
#include <csapp.h>

#define MAXBUF 8192

int read_rst();

// func/func.c
#include "func/func.h"

int read_rst() {
    rio_t rio;
    int fd = Open("./rst", O_RDONLY, S_IRUSR);
    Rio_readinitb(&rio, fd);
    char buff[MAXBUF];
    Rio_readlineb(&rio, buff, MAXBUF);
    int rst =  atoi(buff);
    Close(fd);
    return rst;
}
```

这里使用了 csapp 提供的 wrapper function 进行文件读取

```cmake
add_library(libfunc STATIC func.c)

target_include_directories(libfunc PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
# 添加了一个 private 的 library, 此时 libcsapp 仅仅会被 libfunc linking
target_link_libraries(libfunc PRIVATE csapp)
```

剩下的写法完全不需要修改, 重新 cmake 生成构建文件并完成构建

>   注意这里需要在 `/build` 下创建 `rst`, 并在其中写入一个数字作为结果, 不然程序会报错

上面的例子中使用了 .c 文件作为例子, 而不是 .cpp 文件, 这里主要是遇到了一个问题, csapp.c 是使用 c 编写的, 提供的都是 syscall 的 wrapper, 当我尝试使用 main.cpp 进行 linking 的时候报错了, 尚未解决这个问题, 索性后面都使用 .c 了

# tmux

terminal multiplexer 直译过来就是终端复接器, 将 session 和 shell 相互独立开来

一般而言通过 ssh 的方式连接到远程服务器, 其实就是通过一个 shell 程序登录到远程并开启一个 session, 此时 session 和 shell 程序之间是绑定的, 当网络出现波动, ssh 断开连接, session 也结束了, 有了 tmux, 可以将两者分离

```shell
# 创建一个 session
$ tmux new -s <session-name>
# 查看当前已经存在的 session
$ tmux ls
# 连接到一个已经存在的 session 中
$ tmux attach -t <session-name>
```

一般而言建议给 session 添加一个 name, 方便进行多个用户之间的管理

tmux 操作的核心是: <kbd>Ctrl + b</kbd>, 在此基础添加其他键实现不同的功能:

*   `+ d`: detach from current session, 分离出去但 session 依旧保留

*   `+ %`: Split the window into two panes horizontally, 水平分屏, 简单来说就是分成左右两个屏

*   `+ "`: Split the window into two panes vertically, 垂直分屏, 简单来说就是分成上下两个屏

*   `+ [方向键]`: Move between panes, 通过方向键在不同的屏之间移动

*   `ctrl + b + [方向键](同时)`: 修改当前 pane 的大小

*   `+ x`: Close pane, 关闭当前的分屏, 如果只有一个屏的话, 会直接删除当前 session

*   `+ c`: Create a new window, 这里的 window 类似于虚拟桌面, 一个 session 可以有很多桌面, tmux 创建的 session 默认只有一个 window 0

*   `+ n(p)`: Move to the next(previous)window, 移动到前一个或者后一个桌面上

*   `+ [数字]`: Move to a specific window by number, 移动到指定编号的桌面上

*   `+ &`: Delete current window, 删除当前 window

*   `+ :`: 进入 tmux 的命令行模式, 在各种奇怪的环境中, 按键绑定可能存在问题, 因此有的时候需要通过命令进行 tmux 配置

    *   `:resize-pane -D`: resize current pane down 1 unit
    *   `:resize-pane -U`: resize current pane up 1 unit
    *   `:resize-pane -L`: resize current pane left 1 unit
    *   `:resize-pane -R`: resize current pane right 1 unit

    >   额外添加一个数字作为参数, 可以修改 resize 的 unit 尺寸

tmux 甚至可以用来共享 session, 两个 ssh 连上同一个 session, 可以实现屏幕共享

# pipe

linux 的管道机制, 主要是用来服务进程之间通信的, 可以将一个程序的输出作为另一个程序的输入, 具有如下格式:

```shell
Command A | Command B
```

将 Command A 的输入作为 Command B 的输入, 比如:

```shell
$ ls | grep main
```

除了服务于 shell 命令之外, 用户程序还可以直接调用 syscall -> pipe() 显式的实现父进程和子进程之间的通信

```c
#include <unistd.h>
// returns 0 on success, returns -1 on error
int pipe(int pipefd[2]));
```

pipe 的参数需要一个 `int` 类型的数组, 从变量的命名上可以看出, 调用 pipe 本质上就是设置两个 file descriptor, 其中 `pipefd[0]` 作为 pipe 的输出端, **只能** read, 而 `pipefd[1]` 作为 pipe 的输入端, **只能** write

一对 pipefd 表示一个字节流的方向, 调用 pipe 之后, 再调用 fork, 那么父子进程就可以通过 pipefd 通信了, 不过这里要注意: 父子进程**只能使用两个 fd 中的一个**, 即如果父进程使用 `pipefd[1]` 向子进程发送数据, 就不能再使用 `pipefd[0]` 从子进程中获取数据了

所以如果希望实现父子进程之间的双向通信, 则需要两对 pipefd 数组

借助 pipe() 可以实现并行求解素数, 其原理一张图就能表示:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/07/06/20:09:25:parallel_prime_sieve.gif)

伪代码:

```shell
p = get a number from left neighbor
print p
loop:
    n = get a number from left neighbor
    if (p does not divide n)
        send n to right neighbor
```

每个素数筛进程都会筛掉某个素数的若干倍数, 比如第一个素数筛进程可以筛掉所有 2 的倍数, 而第二个进程可以筛掉所有 3 的倍数

要注意的是这个并行计算的效率并不高, 因为每个进程的负载是不同的, 越靠前的进程需要筛掉的数字个数越多

# IO 重定向

简单的把标准输出重定向到某个文件中, 比如:

```shell
$ objdump -d main.o > dumpfile
```

使用 `>` 和 `>>` 都可以进行重定向, 区别在于 `>` 会以 `w` 的方式修改文件, 而 `>>` 会以 `a` 的方式修改文件

>   所以根本不需要担心文件不存在的问题

linux 中默认 stdout 和 stderr 都会输出到屏幕上, 在进行 log 记录的时候可能需要进行区分, 如果此时源代码已经不能修改了, 那么可以通过  `1>`(`1>>`)将 stdout 重定向, 通过 `2>`(`2>>`)将 stderr 重定向, 默认情况下 `>` 等价于 `1>` 即仅重定向 stdout

## log all message

默认重定向会将 stdout 和 stderr 的输出导入到不同的文件中, 如果希望按照时间顺序将所有的输出导入到相同的 log 文件, 此时需要一种特殊写法:

```shell
# 符号 2>&1 表示将 stderr 重定向到 fd 为 1(stdout)的 file
$ ./main > ./log.txt 2>&1
```

>   不要想着写成: `1> log.txt 2> log.txt` 这种写法会导致输出混乱, log 中的 stdout 和 stderr 打印不是严格按照时间顺序的

## /dev/null

unix 的设计者专门设计了这个文件, 这个 null 文件可以看成是垃圾桶, 将输出重定向到 `/dev/null` 相当于丢弃输出, 读 `/dev/null` 会直接返回 `EOF`

*   discard output: `> /dev/null`
*   discard error message: `2> /dev/null`
*   discard input: `< /dev/null`

# /dev/random (/dev/urandom)

单从使用的角度上来看, `/dev/random` 更安全, 但使用的时候可能被阻塞 (在我 1c2g 的服务器中生成 10 个字节的随机数就等了半分钟); 而 `/dev/urandom` 更高效, 因为它不会阻塞, 其并不是完全随机的

linux 下随机数生成器的工作原理和 kernel 中的 entropy pool 有关, 简单来说在 os 运行的过程中, entropy pool 会不断积累自身的 entropy (google 翻译直接将它翻译为"熵", 这里还是直接写英文吧)

当使用 /dev/random 生成随机数时, 随机数仅仅和 entropy pool 相关, 所有的数字都是从这个 pool 中获取的, 因此当 pool 为空的时候进程会被阻塞等待 pool 的填充, 而使用 /dev/urandom 生成随机数时, 会从 pool 中获取数字作为 seed, 并根据 seed 生成伪随机序列

从原理上, 只要 seed 相同, 伪随机序列是可以被预测的, 因此 /dev/urandom 得到的随机数并不是真正意义上的随机数

entropy pool 的填充和很多因素相关, 比如系统中断, 环境噪声因子 (intel CPU 上有一个引脚可以用来获取噪声因子), 系统事件相关, 因此并不能直接调整 entropy pool 的填充速度

linux 随机数生成器生成的数字是二进制的形式, 默认是不可读的, 在 shell 中可以通过 `dd` 或者 `head -c` 读取若干字节, 并通过管道的方式存入文件, 不过因为文件不可读, 查随机数的时候需要将其转为 16 进制格式

# curl

一个简单好用的 http 工具, 简单的几个命令就可以发出特定需要的 http 请求

*   `curl [url]`: 发出一个基本的 HTTP GET request, 这种情况下会将 HTTP response body 导向 stdout
*   `curl -o [file] [url]`: 和上面一样, 只不过这回会将 HTTP response body 保存到 file 中
*   `curl -O [url]`: 本地保存的文件名和 url 相同
*   `curl -L [url]`: 允许 redirect, 即此时返回的 response body 为重定向的后的结果
*   `curl -X POST -d "key1=value1&key2=value2" [url]`: 发出 HTTP POST request, -d 参数中为 resquest body 中的键值对
*   `curl -H "Content-Type: application/json" -H "Authorization: Bearer [token]" [URL]`: 发出一个 HTTP GET request, -H 参数中为 request header 中的键值对
*   `curl -f [url]`: 当 curl 失败后, 将返回 HTTP code (4xx 和 5xx 系列)
*   `curl -s [url]`: silent mode, curl 将不打印下载进度和错误信息
*   `curl -S [url]`: 通常和参数 `-s` 一起使用, 此时 curl 仅仅打印错误信息

一种经典的用法: `curl -fsSL [url]` 此时不会打印下载进度, 但会打印错误信息, 并且允许重定向 (此时原链接和重定向链接的错误信息都会被打印)

代理设置, 出于某些魔法原因在使用 linux 的时候不可避免的需要使用代理, 一般而言建议设置全局的代理, 在对应的 .zshrc(.bashrc) 中添加:

```bash
export ALL_PROXY=socks5://[ip]:[port]
```

如果只是为某次 curl 设置一个代理的话其实很简单:

```shell
$ curl --proxy socks5://[ip]:[port] [url]
```

在使用 curl 下载时最好使用参数 -L, 某些网站下载的时候会进行重定向, 这里最好加上

# wget

类似 curl

*   `-q`: quite mode, 此时 wget 不会打印任何下载相关信息
*   `-O`: 用来指定目标文件

代理设置:

```shell
export http_proxy=socks5://[ip]:[port]
export https_proxy=socks5://[ip]:[port]
```

# dd

data duplicator (也可以是 data dump), 用来复制的

```shell
# simple copy
$ dd if=[input_file] of=[output_file]
# create disk image (create image from disk -> /dev/sda)
$ dd if=/dev/sda of=disk_image.img
# disk wiping (dangerous, think before operation !!! this operation will wipe disk -> /dev/sda)
$ dd if=/dev/zero of=/dev/sda
# random data
$ dd if=/dev/urandom of=random_numbers.bin bs=1 count=10
```

# readelf

ELF(executable linkable file), 这种文件已经是二进制的了, 尽管包含了信息, 但不再是简单的文本信息, 强行打开其实也是可以看的, 不过需要转换到十六进制

readelf 是专门用来解析 ELF 文件信息的工具

>   啥也不知道的时候直接 `readelf --help`

添加各种参数实现不同程度的解析:

*   `-a`: 解析整个文件(一般不用, 太大了)
*   `-h`: 只解析 ELF header
*   `-S`: 解析每个 section 的 header
    *   address 表示当前 section 在 virtual address 中的位置
    *   offset 表示当前 section 在文件中的偏移量 

# IEEE 754

>   有关浮点数的规范

每个浮点数被分为三部分保存, 每个浮点数可以表示为: $(-1)^\textcolor{red}{\text{sign bit}}(1 + \textcolor{green}{\text{fraction}})\times2^{\textcolor{violet}{\text{exponent}} -\text{bias}}$

每个浮点数通过四个参数表示, 其中参数 `bias` 是一个确定值, 在单精度 (float) 的情况下, bias 取 127, 在双精度 (double) 的情况下 bias 取 1023

在这种二进制的表示方式中, fraction 一定是一个小数, 即保证有 $1\leq 1 + \textcolor{green}{\text{fraction}} < 2$

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/12/25/10:50:57:IEEE_754.png)

在单精度浮点数中, exponent 通过 8 bit 表示, 可以表示的范围为 0 ~ 255, 其中 0 和 255 具有特殊含义, 保留下来不做某个浮点数表示, 实际可以表示的指数范围为 -126 ~ 127

而在双精度浮点数中, 同理可得 11 bit 二进制表示范围为 0 ~ 2047, 其中 0 和 2047 保留, 实际可以表示的指数范围为 -1022 ~ 1023

然后就是一些特殊情况:

*   0 值: 不管是单精度函数双精度, 只要所有位都是 0 就表示该值为 0 值
*   Infinity: (biased) exponent 为全 1 (在单精度中为 255, 在双精度中为 2047), 而 fraction 为全 0; 此时根据 sign bit 决定是 positive infinity (sign bit 为 0) 或者是 negative infinity (sign bit 为 1)
*   NaN (not a number): (biased) exponent 为全 1 (在单精度中为 255, 在双精度中为 2047), fraction 可以为除了全 0 之外的任意值, sign bit 的可为 0 也可为 1

>   从表示形式上, Infinity 和 NaN 最大的区别在于, Infinity 的 fraction 部分一定为全 0, 而 NaN 的 fraction 部分一定不能是全 0
>
>   NaN 用来表示非法操作, 最典型的 dividing by zero, 由于 NaN 实际可以表示的范围很大 (单精度有 23 bit, 双精度有 52 bit), 因此有一种被称为 NaN Boxing 的技术, 可以通过浮点数表示任意的类型

考虑一个例子: 0.085

*   首先这是一个整数, 因此 sign bit 一定为 0

*   然后考虑将 0.085 使用科学计数法表示: $0.085 = (-1)^{\textcolor{red}{\text{0}}}(1 + \textcolor{green}{\text{fraction}})\times2^{\textcolor{violet}{\text{power}}}$

*   等价运算为: $\frac{0.085}{2^{\textcolor{violet}{\text{power}}}} = (-1)^{\textcolor{red}{0}}(1 + \textcolor{green}{\text{fraction}})$

*   代入不同的 power 使得 $1 \leq (1 + \textcolor{green}{\text{fraction}}) < 2$, 在本例中, $\textcolor{violet}{\text{power}}$ 取 -4 时 有 $\frac{0.085}{2^{\textcolor{violet}{-4}}} = 1.36$

*   考虑单精度浮点数的情况, 有 $\textcolor{violet}{\text{exponent}} = 123 = \text{b}01111011$

*   最后就是将 0.36 使用二进制表示了, 将小数部分转化为二进制表示:

    -   Multiply the fractional part of the decimal number by 2.
    -   Note the integer part of the result (it will be the next bit in the binary representation).
    -   Update the fractional part to the decimal part of the result.
    -   Repeat the process until the fractional part becomes 0 or until you have obtained the desired precision.
    -   The binary representation is the sequence of integer parts obtained.

    并不是所有的小数都可以通过有限位的二进制表示的, 应该说大部分小数在表示为二进制时, 都存在精度问题, 针对 0.36, 采取上述方式进行二进制转化得到:

    ```ascii
    0.36 x 2 = 0.72
    0.72 x 2 = 1.44
    0.44 x 2 = 0.88
    0.88 x 2 = 1.76
    0.76 x 2 = 1.52
    0.52 x 2 = 1.04
    0.04 x 2 = 0.08     Once this process terminates or starts repeating,  
    0.08 x 2 = 0.16     we read the unit's digits from top to bottom
    0.16 x 2 = 0.32     to reveal the binary form for 0.36: 
    0.32 x 2 = 0.64
    0.64 x 2 = 1.28      0.01011100001010001111010111000...
    0.28 x 2 = 0.56
    0.56 x 2 = 1.12
    0.12 x 2 = 0.24
    0.24 x 2 = 0.48
    0.48 x 2 = 0.96
    0.96 x 2 = 1.92
    0.92 x 2 = 1.84
    0.84 x 2 = 1.68
    0.68 x 2 = 1.36
    0.36 x 2 =  ...  (at this point the list starts repeating)
    ```

    考虑单精度浮点数时, $\textcolor{green}{\text{fraction}}$ 部分使用 23 bit 表示, 因此这里需要对其截断处理, 得到 $\textcolor{green}{\text{0.01011100001010001111011}}$

*   最后拼接一下就是整个单精度浮点数了: $\textcolor{red}{\text{0}}\ \textcolor{violet}{\text{01111011}}\ \textcolor{green}{\text{01011100001010001111011}}$

# 磁盘相关

以使用 SATA 接口的磁盘为例, 其在 linux 中的磁盘名为 /dev/sd[a-p], 每个 SATA 接口对应一个不同的下标

此外磁盘还可以进行分区, 不同分区在 linux 中也映射到了不同的文件名, 比如 /dev/sda1, /dev/sda2

## 分区表

同一个磁盘下, 不同的分区, 需要一种管理方式, 比如记录某分区占用的扇区的起止位置。这种管理方式被称为磁盘分区格式, 目前常见的两种分区格式: MBR (master boot record) 和 GPT (GUID partition table)

### MBR

在 MBR 中第一个扇区用来保存开机管理程序和分区表,  通常该扇区的大小为 512 Bytes, 从名字中也能看出来, master boot record -> 主要 开机 记录, 其实保存的就是开机引导程序, 在第一扇区中通常占用 446 Bytes, 而分区表占用 64 Bytes, 分区表记录了分区的起止位置, 有如下格式:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/01/05/20:53:13:MBR_partition_table.png)

由于分区表的大小限制, 在 MBR 的第一扇区中最多只能记录四个分区, 上述的四个分区可能被表示为: /dev/sda1 /dev/sda2 /dev/sda3 /dev/sda4

>   分区从某种程度上更有利于 space locality, 毕竟其可以将数据限制在某些柱面中而不是整块磁盘

这里被使用的四组分区被分为 primary partition(主分区) 或 extended partition(扩展分区)

在实际中, 一块磁盘显然不仅仅可以分为四个区, 尽管第一个扇区仅能存储 4 个分区, 再随便多用几个扇区自然就可以保存更大的分区表了

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/01/05/21:00:43:MBR_extened_partition_table.png)

上图展示了一个使用 extended partition 实现分区扩展的例子, 这里被 extened partition table 记录的各个分区被称为 logical partition

在 linux 中, logical partition 从 5 开始计数, 还是上面的例子, 其文件名会被识别为: 

-   P1:/dev/sda1
-   P2:/dev/sda2
-   L1:/dev/sda5
-   L2:/dev/sda6
-   L3:/dev/sda7
-   L4:/dev/sda8
-   L5:/dev/sda9

由 linux 本身的限制, extended partition 最多只能有一个, 在完成了磁盘格式化之后, 可以被使用的分区为 primary partition 和 logical partition

在 MBR 中最重要的扇区就是第一扇区, 任何对磁盘上数据的获取, 首先都要从第一扇区的分区表开始

### GPT

随着磁盘容量不断变大, 一个扇区的大小也长到了 4 KB, 分区的设置不再局限于某些柱面 (MBR), 而是由 logical block address 确定, 其中每个 LBA 大小为 512 Bytes

GPT 将磁盘看成是一个 LBA 数组, 前 34 个 LBA 用来管理分区信息, 此外磁盘最后的 33 个 LBA 用来保存分区表的备份, GPT 具有如下格式:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/01/05/21:16:41:gpt_scheme.jpg)

LBA0 的格式和 MBR 的第一扇区类似, 之前的 446 Bytes 保存了开机管理程序, 原本用来保存分区表的 64 Bytes, 现在进行了特殊标识, 表明该磁盘使用了 GPT 分区格式

LBA1 保存了 GPT header, 记录分区表的大小和位置 (包括备份表的位置), 此外还是用了 32 bit 的 CRC 校验位, 用来给 os 进行校验

要注意的是在 GPT 中没有 primary, extended, logical partition 的区分, 可以使用了 GPT 的分区的磁盘, 每个分区都是主分区

## 引导程序

设备通电开机之后, 并不会直接载入系统, 还需要使用引导程序作为驱动, 主流的可以分为 BIOS 和 UEFI







