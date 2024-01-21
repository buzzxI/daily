# 大纲

为了方便探索互联网，第一步需要从老电脑拷来一份 ShadowsocksR :laughing:

新电脑的环境配置，主要包括：

*   7-zip(直接下，别装 C 盘)
*   snipaste(直接下，因为比较小，也可以在微软商店里面下)
*   [AutoHotKey](#AutoHotKey)：需要配置开机加载脚本
*   Draw.io(直接下，别装 C 盘)
*   [Everything + Wox](#Everything + Wox)：实现全局的文件搜索
*   Foxit PDF Reader(不是 Editor，这个 Reader 是免费的，别装 C 盘)
*   Git：直接下，这里注意可以不勾选安装集成的 vim 和 openSSH 毕竟我们自己后面也会手动装，一般情况下个人也不会使用 git bash，而直接使用 windows terminal，所以需要配置环境变量
*   [Typora](#Typora)：需要配置字体、PicGo
*   [PicGo](#PicGo)：配置 github 图床的位置
*   [sublime](#sublime)：需要配置 package control
*   坚果云(直接下，别装 C 盘)
*   PotPlayer(直接下，别装 C 盘)
*   [Tim + 微信](#Tim + 微信)
*   [vim](#Vim)：这个下的是 gVim，和 git 类似的，安装后按需配置环境变量
*   jdk：下载了 8、11、17 三个版本，配置 JAVA_HOME 和 PATH
*   [maven](#Maven) 和 tomcat：配置好 MAVEN_HOME 和 Path
*   JetBrains 全家桶：主要是 ToolBox，配置 ToolBox 的下载路径，然后再下 IDEA 和 DataGrip
*   [Mysql 8.x](#Mysql 8.x)
*   [Windows Terminal + PowerShell](#Windows Terminal + PowerShell) 美化终端
*   [OpenSSH](#OpenSSH)：主要是配置连接多个终端的 SSH 统一配置文件 config
*   [github](#github)：主要是用来进行远程 github 账户绑定
*   [node.js](#node.js)：主要是写 vue

# 字体

这个字体真的很关键，换了之后确实会好看一点，我这里使用的是：[Caskaydia Cove Nerd Font Complete](https://www.nerdfonts.com/font-downloads)

下好了之后是一个 .zip 格式的压缩文件，里面字体是 .ttf 格式的，双击即可安装

>   我这里对字体进行了额外的备份(统一放在了 [E:/font] 中)

不过其实完全不需要这么麻烦，因为微软默认对于字体的管理还是比较完善的：

用户自行安装的字体放在了：C:/Users/[用户名]/AppData/Local/Microsoft/Windows/Fonts

而这个电脑上所有安装的字体放在了：C:/Windows/fonts

如果需要删除特定的字体，**一定要去** C:/Windows/fonts，不然会出现奇怪的错误

在字体部分，如果写文档的话，个人感觉 Caskaydia Cove Nerd Font 这个字体挺好看的；写代码的话还是 Consolas

>   尽管 Caskaydia Cove Nerd Font 本身是用在 windows terminal 中的，但它这个英文和数字的显示确实好看啊

# AutoHotKey

到官网下好之后需要编写脚本文件，安装时要勾选右键可以新建 .ahk 文件的选项

>   不勾也行，大不了就改后缀名

因为需要开机启动，所以脚本文件的位置选择在启动项中：

```shell
$ C:\Users\[这个位置是用户名]\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
```

因为装 win11 肯定是中文的系统，所以其实路径名应该是下面这样的

![Auto_Hot_Key_Script](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/Auto_Hot_Key_Script.png)

然后根据个人的习惯编写 Script.ahk 文件

```txt
^+e::Run C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe
^+t::Run D:\Typora\Typora.exe
^+c::Run calc.exe
^+s::Run D:\Sublime Text\sublime_text.exe
^+Up::SoundSet +5
^+Down::SoundSet -5
```

具体的写法参考[Quick Reference | AutoHotkey](https://www.autohotkey.com/docs/AutoHotkey.htm)

>   简单说一下：^ 代表了 <kbd>ctrl</kbd>；+ 代表了 <kbd>shift</kbd>；Up 为方向的 <kbd>↑</kbd> 键；Down 为方向的 <kbd>↓</kbd> 键；(剩下的都不说了，就是普通的字母而已)
>
>   :: 后面紧跟脚本，Run 表示运行程序；SoundSet 表示设置音量 $\pm$ 分别表示了增加声量和减少声量
>
>   要注意的是使用这种方法进行音量加减时，不会弹出 windows 的音量提示框，不过如果主动打开声量面板还是可以看到声音变化的

# Everything + Wox

Everything 没什么好说的官网直接下就好

Wox 的新版都是 Pre-release 版本，最新的 stable 版本太老了

注意最新版的 full-installer，默认勾选上了安装 Everything 和 python 环境，我们不需要它安装其他环境，注意把那两个选项取消

安装后，设置快捷键，主题(默认的主题不好看，建议设置成透明磨砂背景的那个 BlurBlack)，[字体](#字体)

# PicGo

注意 PicGo 默认勾选上了快捷上传 <kbd>ctrl + shift + p</kbd>，注意取消掉

*   获取 github token
*   配置 PicGo 中 github 图床的设置(需要配置 CDN 加速)

## 获取Token

这个需要在 GitHub的 [setting] 中选择 [Developer settings]，找到 [Personal access tokens] 选项，现生成一个 token

生成的过程时，需要勾选 [repo] 选项，表示这个 token 可以用来操作当前账户下的仓库

因为 token 只会出现一次，所以需要妥善保存这个 token

## 配置 PicGo

这里主要参考 [配置手册 | PicGo](https://picgo.github.io/PicGo-Doc/zh/guide/config.html#github图床)

唯一需要注意的是在自定义域名选项添加`https://cdn.jsdelivr.net/gh/[用户名]/[仓库名]/`表示使用了 CDN 加速，比如我的配置为：

```shell
https://cdn.jsdelivr.net/gh/buzzxI/img/@latest
```

其中最后的 @latest 加的很玄学，不加的话可能图片的时效性会变差...

>   其他的配置随便点点就行了比较简单

# Typora

收费之后再安装 dev 版本的好像就不能选择安装路径了，所以默认装到了 C 盘

和其一起安装的还有 Pandoc，这个主要是用来导出其他格式的文件的时候用到的

关于 Typora 简单的配置直接在偏好设置里面修改就好了，稍微需要注意的是：

*   字体配置
*   结合 PicGo 实现图片上传至 GitHub

官方给的配置已经挺完善了，如果找不到配置，就到 [Typora Support](https://support.typora.io/) 去查

## 字体配置

需要注意的是，我们修改字体不要在原来的主题文件中进行修改，按照 Typora 官方的说法，在升级的时候会重置，所以我们最好单独写配置

不过官方非常体贴 Typora 加载主题的顺序是：

1.   Typora 基础样式
2.   选定的主题样式(比如默认的 github.css 中的样式)
3.   base.user.css 文件，对就叫这个名字，我们在主题文件夹中直接创建这个文件就行
4.   [currnet_theme].user.css，这个文件的名字和当前加载的主题相关，其余的和上面的第 3 点一样

注意到后面的第 3 、4 点都涉及到了自建文件，因为读取 css 文件的样式会覆盖到前面的配置，所以如果我们对样式不满意，就通过在后面的文件中定义，覆盖掉前面的配置，当然包括了字体配置

>   base.user.css 中的配置会覆盖掉所有的主题中的配置，而 [current_theme].user.css 中的配置仅会覆盖某种主题中的配置

这里以 github 主题为基础，修改字体配置，因为我们[前面](#字体)已经自行安装过字体了，所以这里使用的也就是 CaskaydiaCove NF

>   不是我多嘴，反正这个字体看起来比更纱黑体好看

```css
/* 文本的字体为 CaskaydiaCove NF */
body {
	font-family: CaskaydiaCove NF;
}
/* 修改 Typora 在源码格式下的字体 */
#typora-source {
	font-family: Consolas;
	font-size: inherit;
    --monospace: Consolas; /* 修改代码块部分的字体 */
}
/* 启用字体连字 */
#write {
	font-variant-ligatures: normal;
}
```

==下面部分的内容是是从网上看到的，修改其他部分的字体，个人感觉没这个必要==

然后修改 conf 配置文件，进入 [偏好设置] 中的 [打开高级设置]

这会打开一个文件夹，里面就是两个 conf.json 的配置文件，需要修改的是 "defaultFontFamily"中的配置：

```json
"defaultFontFamily": {
    "standard": "CaskaydiaCove NF", //String - Defaults to "Times New Roman".
    // 字符末端有额外的装饰
    "serif": "CaskaydiaCove NF", // String - Defaults to "Times New Roman".
    // 字符末端没有额外的装饰
    "sansSerif": "CaskaydiaCove NF", // String - Defaults to "Arial".
    // 字符具有相同的宽度
    "monospace": "CaskaydiaCove NF" // String - Defaults to "Courier New".
}
```

>   默认的配置为 null，这里修改为特定的字体

## 联动 PicGo

主要是在 [偏好设置] 中的 [图像] 中进行配置，因为是图形化的界面，随便点点就行

## 代理问题

因为使用 github 作为图床，尽管使用了 cdn 加速了，但实际访问的时候，如果选择使用系统代理，那么还是不能获取图片

>   他这个链接如果是系统代理的情况下，就变为了 rawgithubxxx 的 url，所以如果访问不了 github 那么也是访问不了图片的
>
>   cdn 的作用不是代理，而是加速访问，但如果本来就是断的，那加速也是断的

参考官方提供的代理方式：[Launch Arguments (Windows / Linux) - Typora Support](https://support.typora.io/Launch-Arguments/)

简单来说，可以在每次启动的时候添加启动参数，--proxy-server:address:port

或者把这个参数添加到配置文件中，具体的，添加到高级设置中的 conf.json 文件中，找到带有 flags 的选项

```json
"flags": [["proxy-server", "address:port"]]
```

官方建议把协议加上，比如：

```json
"flags": [["proxy-server", "socks5://127.0.0.1:1080"]]
```

>   实际的端口参考自己开放的端口

# Windows Terminal + PowerShell

*   到微软的商店中下载 Terminal 和 PowerShell
*   [配置字体](#字体)
*   安装 oh-my-posh 和 终端图标(美化后的终端图标)
*   简单配置 terminal 包括：默认启动项，外观字体，毛玻璃特效... (剩下的配置想配哪个就配)
*   修改 PowerShell 的 profile

## 微软商店下载

没啥说的，略过

## oh-my-posh 和终端图标

官网在 [Home | Oh My Posh](https://ohmyposh.dev/)，按照官方建议的方式安装，在 Powershell 中：

```shell
$ winget install JanDeDobbeleer.OhMyPosh -s winget
```

安装后主题的位置在：`C:\Users\[用户名]\AppData\Local\Programs\oh-my-posh\themes`

安装终端图标，在 PowerShell 中：

```shell
$ Install-Module -Name Terminal-Icons -Repository PSGallery
```

>   需要注意的是，完成了这一步之后，仅仅是完成了 oh-my-posh 和终端图标的安装，并未完成配置，所以  PowerShell 中并没有发生什么变化

## 简单配置 terminal

将这个随便点点就行了，进入 terminal 中的 [设置]，可以在原来的 PowerShell 选项卡中进行修改，也可以新定义一个配置，反正我就只是修改了默认的启动项(修改为 PowerShell)、字体(修改为之前安装的字体)、添加了毛玻璃的效果

## 配置 PowerShell 的 profile

首先找到 PowerShell 启动时加载的配置文件：

```shell
$ echo $profile
```

默认的位置为：`C:\Users\[用户名]\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`

>   要注意的是如果是刚刚安装的 PowerShell 那么默认是不存在这个路径的，需要手动在 \Documents 目录下新建 \PowerShell\Microsoft.PowerShell_profile.ps1

然后在这个配置文件中写入：

```txt
# 这一行是配置启动 PowerShell 时加载 oh-my-posh 中的某个主题
oh-my-posh --init --shell pwsh --config ~\AppData\Local\Programs\oh-my-posh\themes\[oh-my-posh 的 themes 目录下的某个主题的配置文件] | Invoke-Expression
# 这一行是配置启动 PowerShell 时加载终端图标
Import-Module -Name Terminal-Icons
```

保存好配置文件后，刷新当前 PowerShell 的配置：

```shell
$ . $profile
```

# OpenSSH

配置 win11 上的 OpenSSH，具体的步骤参考了官方 [安装 OpenSSH | Microsoft Docs](https://docs.microsoft.com/zh-cn/windows-server/administration/openssh/openssh_install_firstuse)

我这里默认安装了 OpenSSH 的客户端，而没有安装 Server，所以根据上面的教程安装了 Server

>   要注意的是在使用下面的 PowerShell 的方式进行安装时，报了没有注册表的错误

正常情况下，在目录 `C:\Users\[用户名]\.ssh\`下保存了密钥，但是默认的情况下并没有 .ssh 目录

这里手动创建 .ssh 目录

因为需要管理多组 ssh 密钥，所以这里使用 config 配置文件的形式，管理连接到不同服务器的密钥

```shell
$ vim config
```

注意这里的配置文件名为 config，没有后缀，具体的配置可以参考下面对于 [github ssh 密钥](#配置 ssh)的配置

## 配置 win 的 OpenSSH server

现在需要从一个 win 下连接另一个 win，首先需要在 win 上装好 OpenSSH server 具体可以参考 [安装 OpenSSH | Microsoft Docs](https://docs.microsoft.com/zh-cn/windows-server/administration/openssh/openssh_install_firstuse)

根据上面的教程启动 win 下的 SSH server 服务

第一次从另一台设备连接时需要注意使用：[用户名]@[ip/域名] 的方式登录

```shell
$ ssh buzz@[ip地址]
```

>   这里 buzz 就是开启了 ssh server 端的用户名

注意这种登录方式需要输入密码，并不是很安全，更好的方法是使用 ssh 私钥进行连接

# github

这里需要配置的是本地仓库和 github 免密码实现 pull 和 push

首先设置本地 git 的用户名和邮箱：

```shell
$ git config --global user.name "[用户名]"
$ git config --global user.email "[邮箱]"
```

## 配置 ssh

在 `C:\Users\[用户名]\.ssh`下新建 `\github` 目录，用来保存 github 的密钥

创建密钥：

```shell
$ ssh-keygen -t [协议名] -f [生成的密钥名(包含了路径)] -C "[邮箱]"
```

>   因为 `-C` 表示的是备注信息，所以并不确定这个字段是不是必要的

然后会得到两个文件 `github_key` (ssh 私钥) 和 `github_key.pub`(ssh 公钥)，我们需要做的就是把公钥配置到 github 的个人账户即可

因为管理了多个 ssh 链接，所以需要对 ssh 进行配置，在 .ssh/config 中添加如下配置

```shell
# 配置用于登录 github 的私钥
# 第一个选项 Host 正常是可以随便取的，但是这里必须为github.com
Host=github.com
# 第二个选项为远程服务器的地址，这里使用的是域名
HostName=github.com
PreferredAuthentications=publickey
# 最后一个选项表示了私钥的地址，注意这里的必须是绝对地址
IdentityFile=C:\Users\[用户名]\.ssh\[ssh key 的路径]
User=git
```

测试的时候使用：

```shell
$ ssh -T git@github.com
```

>   熟悉 ssh 命令的话，因为在 config 文件中已经配置了 User 属性，因此测试的时候的 git@ 是多余的，测试的时候完全可以写成：
>
>   ```shell
>   $ ssh -T github.com
>   ```

## 配置多个 github 账户

需求总是不断在变，现在的需求是一台电脑上关联多个 github 账户，并且都是用 git 进行管理

其实关键就是修改 ssh 的配置，还是一样的，进入 .ssh, 创建一个新的 ssh-key

```shell
$ ssh-keygen -t [协议名] -f [生成的密钥名] -C "[邮箱]"
```

并将 ssh-key 存放到对应的目录下，然后修改 config 文件，关键点在于 Host 的不同，剩下的基本一致

对于第一个账户其 Host 默认配置为了 github.com, 第二个账户为了区分，这里设置为 second.github.com, 测试的时候使用：

```shell
# 测试第一个账户
$ ssh -T git@github.com

# 测试第二个账户
$ ssh -T git@second.github.com
```

>   注意到也仅仅是 Host 的不同

同样的，在 clone 的时候也需要注意，普通的 git clone 直接复制过来就行了，但是对于第二个账户，因为其 Host 的问题，其 clone 语句也需要修改

这里假设第一个账户为 first，第二个账户为 second，他们俩具有一个同名的 repo foo

如果需要 clone 第一个账户的仓库，直接复制 github 上提供的命令即可：

```shell
$ git clone git@github.com:first/foo.git
```

但如果是第二个账户的话**需要手动修改命令**

```shell
$ git clone git@second.github.com:second/foo.git
```

>   当然如果是通过 HTTPS 的方式 clone 就没这些讲究了

### 修改 user/email

也许是因为 git 在每次提交的时候会将用户名和邮箱同步 commit，此后如果 push 到 github 上，那么最终 commit 的用户可能和最开始预想的不太一样

比如因为之前给第一个用户 first 配置过 global 情况下的 user 和 email，在第二个用户的 repo 下进行 push 发现 github 页面中显示 commit 的居然是 first

这里为了避免这种"你中有我"的问题，需要在第二个用户的 repo 下重新设置 user 和 email

```shell
$ git config --local user.name "[第二个用户的用户名]"
$ git config --local user.email "[第二个用户的邮箱]"
```

# Vim

注意环境变量的配置，此外为了防止文字乱码，配置了 _vimrc 文件，这个文件在 win 中存放在了 vim 的安装目录下，添加如下配置

```shell
# utf-8 编码防止乱码
set encoding=utf-8
set fileencodings=utf-8,ucs-bom,gb18030,gb2312,cp936
set termencoding=utf-8
# tab 为 4 个空格
set tabstop=4
# 显示行号
# set number
```

# Tim + 微信

直接下，别装 C 盘，同时注意登录进去，然后修改文件下载的位置选项，然后取消接受文件自动下载，配置下载文件的地址

这里 Tim 有一个小问题，无法通过设置修改默认存放文件的位置(每次修改，就重启 Tim 客户端，然后就又还原配置了)

贴吧老哥提出了，手写配置的方式：

*   首先迁移原来的配置，把他从 ~/Document 复制到希望的位置上(如果还想要原来的配置就迁移，不想要的话直接跳过这一步就行)

*   然后进入 C:/User/public/Document(这个路径可能是中文的，反正自己简单翻译一下就行)，检查当前路径下是否存在 Tencent/QQ/UserDataInfo.ini 文件(如果没有就创建)

*   修改里面的配置：

    ```ini
    [UserDataImportSet]
    NeedImport=0
    OldVersion=
    OldVerDataPathType=
    OldVerDataPath=
    # 这里需不需要改存疑，如果不放心的话就改成 Tim 的安装路径
    OldQQInstallPath=C:\Program Files (x86)\Tencent\QQ
    [UserDataSet]
    UserDataSavePathType=2
    UserDataSavePath=[这里填写新的文件目录]
    NewVersion=
    ```

# Mysql 8.x

主要参考 [MySQL :: MySQL 8.0 Reference Manual :: 2.3.4 Installing MySQL on Microsoft Windows Using a noinstall ZIP Archive](https://dev.mysql.com/doc/refman/8.0/en/windows-install-archive.html)

从官网上下 .zip 格式的安装包，解压，并定义好环境变量 MYSQL_HOME 和 PATH

## my.ini

在 MYSQL_HOME 目录下创建配置文件 my.ini

>   这个文件并不是必须的，在启动 mysql 服务的时候默认会加载这个文件，每次启动 mysql 的时候也可以通过在命令行添加指定参数实现同样的功能

```ini
[mysqld]
# 设置3306端口
port=3306
# 设置mysql的安装目录
basedir="D:/mysql/mysql-8.0.29-winx64"
# 设置mysql数据库的数据的存放目录
datadir="D:/mysql/mysql-8.0.29-winx64/data"
# 允许最大连接数
max_connections=200
# 允许连接失败的次数。这是为了防止有人从该主机试图攻击数据库系统
max_connect_errors=10
# 服务端使用的字符集默认为UTF8(注意这里需要写成UTF8MB4)
character-set-server=UTF8MB4
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB
# 默认使用“caching_sha2_password”插件认证，取代了 mysql_native_password 更加安全，一些老的项目会报错
#default_authentication_plugin=caching_sha2_password
# 在进行 mysql server 初始化的时候，报出了'default_authentication_plugin' is deprecated and will be removed in a future release. Please use authentication_policy instead. 所以下面使用 authentication_policy 代替 default_authentication_plugin
authentication_policy=caching_sha2_password,,
[mysql]
# 设置mysql客户端默认字符集
default-character-set=UTF8MB4
[client]
# 设置mysql客户端连接服务端时默认使用的端口
port=3306
default-character-set=UTF8MB4
```

>   关于多重身份认证参考 [MySQL :: MySQL 8.0 Reference Manual :: 6.2.18 Multifactor Authentication](https://dev.mysql.com/doc/refman/8.0/en/multifactor-authentication.html#multifactor-authentication-policy)

保存后进行 mysql server 的初始化

```shell
$ mysqld --initiailze --user=root --console
```

这会得到一个随机的密码

## 修改密码

```shell
$ mysql -uroot -p
$ [这里写刚才得到的随机密码]
```

第一次连接到 mysql server 时，需要修改 root 用户的密码

```shell
$ alter user 'root'@'localhost' identified with caching_sha2_password by '[新密码]';
$ flush privileges;
```

# Maven

首先是配置 MAVEN_HOME 和 Path

## Maven.conf

配置文件的位置在 MAVEN_HOME/conf/settings.xml，配置**本地仓库地址**和**阿里云镜像**

```xml
<!--p-->
<localRepository>D:/maven_repo</localRepository>
<mirrors>
    <!--配置阿里云 maven 镜像-->
    <mirror>
      <id>aliyunmaven</id>
      <mirrorOf>*</mirrorOf>
      <name>阿里云公共仓库</name>
      <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
</mirrors>
```

# sublime

主要是安装插件，为了方便管理需要安装 package control

直接 <kbd>Ctrl + Shift + p</kbd> 搜索 package control 进行安装即可

*   side bar enhancement：可以将文件在浏览器中打开的插件，写 HTML 的时候用的，当然下载好了还不能直接使用需要简单配置一下，其实就是键位绑定配置

    ```json
    [
        { 
            "keys": ["f12"], 
            "command": "side_bar_files_open_with",
            "args": 
            {
                "paths": [],
                "application": "C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe",
                "extensions":".*",
            }
        }
    ]
    ```


# node.js

[Node.js (nodejs.org)](https://nodejs.org/en/) 官网下，因为希望版本控制更加全面，这里使用 .zip 文件安装的方式

按照其他人的说法需要：

*   解压，并新建两个文件夹 node_global 和 node_cache
*   配好环境变量
*   配置 node 的全局路径和 node 的 cache 路径(就是把上面两个文件的路径配好)
*   换源

注意配置环境变量的时候需要配置两个，一个是 NODE_HOME 指向的是 node.js 的解压的路径，还有一个是 NODE_PATH 指向的是 NODE_HOME/node_global

第一个路径保证了我们可以全局使用 node，npm 的命令

第二个路径保证了我们可以全局使用通过 npm 安装的比如 webpack vue-cli 等等

配置全局路径：

```shell
npm config set prefix "[文件夹 node_global 的路径]"
```

配置 node cache 路径

```shell
npm config set cache "[文件夹 node_cache 的路径]"
```

配置源，具体的地址现查[阿里巴巴开源镜像站-OPSX镜像站-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/mirror/)

```shell
npm config set registry http://registry.npmmirror.com
```

## 其他问题

在配置好全局变量后，发现 npm -v 出现了问题

```shell
npm WARN config global `--global`, `--local` are deprecated. Use `--location=global` instead.
```

主要修改 NODE_HOME 下的两个文件 npm 和 npm.cmd，使用文本编辑器的方式打开，然后对于所有文件中 `prefix -g`的地方都改为`prefix --location=global`

看来以后全局安装就不能一个简单的 -g 就解决了，需要 `--location=global`

## 没地方写的前端基础

这里主要是对于 npm 的命令的介绍，不全面，希望足够支撑使用 vue

### npm install

简单来说就是安装依赖的

*   npm install：不带有任何参数的 npm install 会根据 package.json 下载依赖

    >   据说项目在备份管理的时候不带有 node_modules，即依赖的信息，所以需要重新下
    >
    >   可能是为了节省空间吧

*   npm install [package]：在当前项目中安装对应的 package 依赖，运行结束后依赖放在了：[project-name]/node_modules/ 下

    此时安装后不会将对应的依赖写入 package.json 的 dependencies 或 devDependencies 中，因此 npm install 不会安装对应的依赖

*   npm install [package] --save：和上面基本一样，只不过会将依赖信息添加到 package.json 的 dependencies 节点中

*   npm install [package] --save-dev：和上面基本一样，只不过会将依赖信息添加到 package.json 的 devDependencies 节点中

devDependencies 节点中的依赖是开发时需要的，而在实际部署后不需要；dependencies 节点中的依赖就真的是全局的依赖了

### ES6 模块化规范

使用 export 和 import 关键字导入、导出模块

导出模块(创建模块 userApi.js)：

```js
export default {
	getList() {
		console.log("获取数据");
	},
	save() {
		console.log("保存数据")
	}
}
```

导入模块：

```js
import user from "./userApi.js"
user.getList();
user.save();
```

# SwitchyOmega

一个浏览器的代理的插件，因为默认 ssr 和 v2ray 我都是开的全局代理，所以如果不配置 Edge 的代理的话，就真的默认走全局了

==**一定不要让 ssr 和 v2ray 改变系统代理**==，不然一些 UWP 应用，微软商店，Office 全家桶的联网情况可能出现异常，而且随意的修改系统代理，本身对于系统的网络设置也是一种破坏

>   在 ssr 中设置系统代理规则为保持当前状态不修改

## Auto Switch

实现自动切换的一种情景模式，可以按照官方的教程设定，网上其他的配置也都挺详细的

==**在使用 Auto Switch 之前先配置一种代理模式，即监听本地的 1080 端口(ssr 默认端口)**==

默认情景规则：即不满足规则列表时，使用的代理，这里务必选择直连

规则列表规则：这里选择之前配置好的一种代理模式

规则列表网址：这里选择规则列表格式为 AutoProxy，配合上用的比较多的 [GFWList](https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt) 规则

>   上面的链接就是一个 txt 文件不需要读懂直接配置给 SwitchyOmega 就行，他会安排后面的所有

配置好规则列表网址后，更新情景模式即可

然而真正让我觉得 Auto Switch 有用的地方，并不是所谓的 GFWList，毕竟这个 list 也仅仅是一个 set，也不能包含所有被封禁的网站(当然已经包括绝大部分了)

在这个模式下，还可以在 GFWList 的基础上自定义自己的规则，比如当访问一个网页出现加载错误(可能没有在 GFWList 中但也被封禁了)，此时 SwitchyOmega 很容易自定义当前站点的代理规则，并将规则写入自动切换的情景模式中，下一次再加载就不会出现错误了

>   具体的就是点击插件，选择添加规则即可

## 虚场景

本来我以为搞一个[自动切换](#Auto Switch)已经足够了，直到我买了两个机场，主副同时使用，防止断线

而这段时间，我已经有了一个很完备的 GFWList + 自定义规则的自动切换模式，如果此时引入副线，可以选择再建一个自动切换规则，并重新配置自定义的股则，==**太麻烦了**==

直到看到了虚场景，这个模式主要是用来解决多个代理服务器，如何使用同一个 "GFWList + 自定义切换规则的自动切换模式"的问题

首先要明确虚场景，并不是一种场景，他需要依托其他已经存在的场景存在，相当于其他场景的套壳，特别的，在虚场景的配置中，"虚场景模式"就是被套壳对象，

因为我们要设置不同代理服务器使用同一个自动切换模式，所以就是让多个代理服务器套上同一个壳，然后配置"自动切换模式"的"列表规则"不再是某个具体的代理模式，而是这个壳

因为自定义的自动切换模式已经使用了很长时间了，所以规则列表中具有很多自定义项，一项一项切换太慢了，SwitchyOmage 的虚场景配中的第二项"迁移到虚场景模式" 中的 "取代目标情景模式" 可以实现一键切换

生硬的说还是不够形象，还是举一个例子吧

背景：有一个场景模式为代理模式，假如名为"ssr"，监听本地的 ssr client 的默认端口(1080)，同时还有一个已经使用了很长时间的自动切换模式 ,假如名为 "auto_switch"

现在因为又买了一个机场，且走的是 v2ray 的规则，所以使用的是 v2rayN client(默认端口 10808) 进行代理，在 SwitchyOmege 中已经为其配置好了一个代理模式 "v2ray" 监听了本地的 10808 端口

我不想新增一个自动切换模式用来自动切换到 "v2ray"，因为现成的规则("auto_switch")已经存在，并且很完善了

此时，新建一个虚场景，起名为 virtual，初始化配置"虚场景模式"的"目标"为"ssr"，保存配置后，点击取代目标情景模式

然后，切换回到 "auto_switch" 中发现，原来自定义规则中配置的情景模式已经从 "ssr" 变为了 "virtual"

此时如果虚场景 "virtual" 的"目标"为 "ssr"，那么实现的就是基于本地 1080 端口的自动切换，而如果"目标"为 "v2ray"，那么实现的就是基于本地 10808 端口的自动切换

所以虚场景模式实现了解耦，**代理服务器的选择**和**规则列表配置**之间的解耦

如果想要不同的规则列表，配置不同的"自动切换模式"即可，如果想选择不同的代理服务器，直接在虚场景之间切换即可

而原来的单一的 Auto Switch 自动切换场景是和代理模式强相关的

对于一般的引用场景，比如多个机场下，"科学上网"，配置解耦，是高效的，反正就两个规则，要么代理，要么直连，代理都一样，就是一个冗余

但对于特殊的应用，不同的代理就是不一样，同一个自动切换模式下，需要多种代理规则，指向不同的代理服务器，即代理服务器和规则列表具有很强的相关性，此时虚场景模式就没那么合适了

>   [浏览器代理插件：SwitchyOmega](https://blog.mimvp.com/article/29144.html#menu-proxy-virtual) 这个博客写的很详细，挺好的

# 来到了实验室

因为实验室的显示器数量确实有限，所以现在桌面上键鼠和两台电脑确实有点占地方

现在一个键盘能操作的就不使用鼠标了

## Edge

这里需要配置 vimium

*   vimium 的帮助手册：<kbd>shift + /</kbd>

*   快捷关闭网页：<kbd>x</kbd>
*   恢复刚才退出的页面:<kbd>X</kbd>
*   上下移动网页的手法和 vim 一样，<kbd>j</kbd> 向下 <kbd>k</kbd> 向上
*   左右切换网页：<kbd>J</kbd> 向左；<kbd>K</kbd> 向右
*   检索当前页面的链接，并在新标签页中打开：<kbd>F</kbd>
*   全局检索，并在新标签页打开：<kbd>O</kbd>

vimium 默认配置的全局检索的搜索引擎是百度，太垃圾了，自己换成了 bing 和 google

## IDEA

*   快捷切换 tab：<kbd>alt + ←(→)</kbd>
*   使用 switch 切换 tab：<kbd>ctrl + tab + ↑(↓)</kbd>

# WSL

## 代理

众所周知，本机可以通过 localhost 访问 wsl 下的服务，但 wsl 却不能直接通过 localhost 访问本机

官方提供了一种方法：[使用 WSL 访问网络应用程序](https://learn.microsoft.com/zh-cn/windows/wsl/networking)

通过访问 /etc/resolv.conf 可以访问本机在 wsl 网络下的 ip 地址，当然，在本机中通过 ipconfig 也可以查看这个地址

