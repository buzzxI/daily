# 一些概念

主要参考：[Overview | Docker Documentation](https://docs.docker.com/get-started/)

# Centos

官方上说的，建议设置Docker的源，然后使用yum安装

先装依赖：这个依赖是用来添加添加源的

```shell
$ sudo yum install -y yum-utils
```

然后添加源：国内的话，就用镜像吧，不然太慢：

```shell
$ sudo yum-config-manager --add-repo  https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

然后就可以安装了

```shell
$ sudo yum install docker-ce docker-ce-cli containerd.io
```

安装好了之后，会新建一个用户组`docker`，这个组里是没有用户的

如果不把用户添加到`docker`用户组中，那么以后使用`docker`命令时总是需要加上`sudo`

> 抱歉我是root用户

启用`docker`：

```shell
$ systemctl start docker
```

额外提一下，国内的话最好使用docker的镜像加速器，不然每次`pull`太慢

这里使用的是阿里云的镜像加速器

```shell
# 注意这里的文件可能就是新建的，所以不要说没这个文件
$ sudo vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://[这里每个人的id不同].mirror.aliyuncs.com"]
}
```

> 阿里云获取镜像：https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors

# 官网教程

[Build your Java image | Docker Documentation](https://docs.docker.com/language/java/build-images/)

官网上说了，部署springboot项目

不过它建议先进行docker的开始教程[Orientation and setup | Docker Documentation](https://docs.docker.com/get-started/)

> 行

```shell
 $ docker run -d -p 80:80 docker/getting-started
```

> `-d`表示以`detached mode`运行程序，其实就是让程序在后台运行，这里和`screen`很像
>
> `-p 80:80`表示将容器的`80`端口映射到实体机的`80`端口
>
> `docker/getting-started`就是一个官方的新手镜像
>
> 上面的命令可以简写为：
>
> ```shell
>  $ docker run -dp 80:80 docker/getting-started
> ```

默认的`docker`会先在本地查看是否具有对应的镜像，如果没有就从远程仓库`pull`镜像

> 以下内容从官网上扒下来的

## 容器

简单来说，容器是一个沙箱化的进程，这个进程和其他的容器进程之间是相互隔离的

这些隔离的特性是利用了linux中的内核命名空间和cgroups的概念

> 扩展阅读:cry:：[Demystifying Containers - Part I: Kernel Space | by Sascha Grunert | Medium](https://medium.com/@saschagrunert/demystifying-containers-part-i-kernel-space-2c53d6979504)

* 容器是镜像的一个运行实例，通过DockerAPI或者命令，可以创建，运行，终止，移动，删除实例
* 容器可以运行在本地环境，或者虚拟机环境，或者部署在远程（云上）
* 容器彼此之间是相互隔离的，每个容器中运行不同的软件，具有各自的二进制库，以及配置文件

## 镜像

运行时的容器具有相互隔离的文件系统，这些文件系统是由容器镜像提供的

既然镜像中包含了文件系统，那么它也包含了运行程序需要的各种依赖，配置，脚本，二进制源

此外镜像中还具有容器的配置，比如环境变量，运行指令，以及其他的一些元数据



说了一堆，都没啥用，还得把它刚才运行的docker容器停下来：

```shell
$ docker stop [container name]
```

## 使能BuildKit

这个是用来创建镜像的

如果安装的是`Docker Desktop`，就不需要使能`BuildKit`了，因为它默认是使能的，如果是`linux`的话就需要手动使能

最简单的是使能方式：在docker build的时候使能

```shell
$ DOCKER_BUILDKIT=1 docker build
```

缺点就是每次build都需要写这个配置

其实把这个写在配置文件中：

```shell
$ sudo vim /etc/docker/daemon.json
{
  "features":{"buildkit" : true}
}
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

## 从spring官网上下载demo

我们的目的是制作一个docker镜像，首先需要一个spring的项目，他这里就是从spring官网上下了一个项目

```shell
$ cd /root/java_projects
$ git clone https://github.com/spring-projects/spring-petclinic.git
$ cd spring-petclinic
```

先看看这个项目能不能跑：（**慎点**）

```shell
$ ./mvnw spring-boot:run
```

我是在远程的linux上部署的docker，所以它这个项目下到了linux上，缺少好多依赖的jar包，你在linux上打包，鬼知道会下多长时间

## 创建Dockerfile（重点）

说白了这个文件就是用来创建docker镜像的

使用`docker build`生成镜像，docker默认从`Dockerfile`中读取指令，用以生成镜像，如果希望指定具体的文件可以使用：`docker build --file [文件名]`

在`Dockerfile`中`#`表示注释

比如：

```dockerfile
RUN echo hello \
# comment
world
-----
RUN echo hello\
world
# 上面这两个是等效的
```

### parser directives

我也不知道这个应该怎么翻译，总之这部分个人认为是关于docker本身的设置

因为它的写法：

```dockerfile
# key=value
```

和注释一样，所以默认的话必须写在文件的最开头才能生效

他这个语法还挺严格的https://docs.docker.com/engine/reference/builder/#parser-directives

其实他这个key一共就两个取值：`syntax`、`escape`

#### `syntax`

这个只有在使用`BuildKit`的时候才有效（如果前面没有使能的话，这个就不用写了）

官方建议写成：

```dockerfile
# syntax=docker/dockerfile:1
```

### 基础镜像

我们需要告诉docker，我们制作的镜像是以那个镜像为基础的，最基本的，spring项目肯定需要java环境

```dockerfile
# syntax=docker/dockerfile:1
FROM openjdk:11.0.14.1-oraclelinux8
# 指明后续指令的路径
WORKDIR /app
# 将生成的jar包放入镜像中
COPY blogs-0.0.1-SNAPSHOT.jar blogs.jar
# 声明容器将会暴露8090端口，但仅仅是声明，没有进行绑定操作，其实就是注释的作用
EXPOSE 8090
# 创建好镜像后，容器执行这里的指令开始运行
ENTRYPOINT ["java", "-jar", "blogs.jar", "--spring.profiles.active=aliyun"]
```

