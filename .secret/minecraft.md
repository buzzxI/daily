终于开始搞 minecraft 服务器了

参考资料主要来自 [wiki](https://minecraft.fandom.com/zh/wiki/教程/架设服务器)

其实安装运行真的很简单，本身服务器就是一个 jar 包，直接 java -jar 运行就好了

因为 minecraft 已经被微软收购了，所以现在下载 server 的官方地址就一股微软味

地址在 [Download server for Minecraft | Minecraft](https://www.minecraft.net/zh-hans/download/server)

>   zh 的 url 都变成中文了很方便

把他通过 scp 搞到服务器上，因为这个 jar 运行后会有一堆文件，所以最好单独建一个文件保存服务器文件

都准备好了后，直接 java -jar 就行，不过官方的说法是：

```shell
$ java -Xmx1024M -Xms1024M -jar server.jar nogui
```

第一次运行会直接退出，别急，看一下打印输出，是要看看 eula.txt 文件

好吧，其实就是一个用户协议，直接改成 true 表示同意协议

然后重新运行就好了

>   因为带有 jvm 参数的命令太长了，还是搞一个脚本运行吧

其实这里主要的配置文件是 server.properties，后面的配置都是围绕这个

# 简单配置

首先自己的用户得是管理员用户吧，不然我怎么在游戏中使用命令呢

```shell
$ /op [用户名]
```

这个最开始需要在服务器后台添加，因为最开始没有管理员权限，之后的话可以在游戏中使用命令添加其他用户为管理员

