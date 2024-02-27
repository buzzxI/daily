在 windows 中, 使能了 wsl 之后, 就可以使用 docker desktop 了, 安装之后, 默认创建了两个 distro, docker-desktop 和 docker-desktop-data, 分别用来:

*   run docker engine (dockerd)
*   store containers and images

这两个 distro 默认保存在了 c 盘下的某个 docker 目录下面, 在最新版的 docker desktop 中, 可以对该路径进行重新配置 

# 一些概念

主要参考：[Overview | Docker Documentation](https://docs.docker.com/get-started/)

## container

简单来说，container 是一个沙箱化的进程，docker 借助 kernel namespace 和 cgroups 实现了不同 container 之间的隔离

> 扩展阅读:cry:：[Demystifying Containers - Part I: Kernel Space | by Sascha Grunert | Medium](https://medium.com/@saschagrunert/demystifying-containers-part-i-kernel-space-2c53d6979504)

container 是 image 的一个运行实例，通过 DockerAPI 或者命令，可以创建，运行，终止，移动，删除实例

## image

image 可以为不同的 container 提供相互独立的 file system, 对于运行在 container 中的应用程序而言, 其运行仅仅依赖 container 自带的 filesystem 本身, 因此 image 中也包括了应用程序需要的各种依赖和配置

## 简单操作

docker 的构建是基于 Dockerfile 的, 根据 Dockerfile 生成 image, 通过 image 得到运行的 container

container 需要运行一个项目, 这里选择 docker 官网给出的示例, 官方使用的是一个 node 项目

>   部署 docker 完全不需要考虑 node 项目本身, 只要知道这个能跑就行了

```shell
$ git clone https://github.com/docker/getting-started.git
```

clone 得到的项目可以直接放在 nodejs 环境中运行, 而为了放在 docker 环境中运行首先需要创建 image, 在 app 目录下创建 Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
   
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
EXPOSE 3000
```

>   Dockerfile 中的:
>
>   *   `FROM node:18-alpine` 表示从 `node:18-alpine` 版本的 image 开始构建当前项目的 image
>   *   `WORKDIR /app` 表示 container 中项目的工作路径为 `/app`
>   *   `COPY . .` 表示将当前路径下的所有文件复制到 `/app` 下
>   *   `RUN yarn install --production` 表示运行 `yarn install --production` 添加项目的各种依赖
>   *   `CMD ["node", "src/index.js"]` 表示当 container 开始运行之后需要执行的指令; 其和 `RUN` 的最大区别在于 `RUN` 运行在构建 image 阶段, 因此各种依赖在 image 构建完成之后就已经写入 image 了, 生成 container 的时候所有的依赖都已经准备好了, 而 `CMD` 表示在 container 开始运行的时候需要执行的指令
>   *   `EXPOSE 3000` 表示 container 在运行的时候需要监听 3000 端口, 要注意的是 `EXPOSE` 并不会直接占用端口, 它只是表示了当前 container 的应用程序在运行的时候需要占用端口  3000, 其在 Dockerfile 中的作用类似于指示文档, 用来告知测试人员在测试的时候不要忘了映射 container 的 3000 端口

### 创建 image

在当前目录下运行:

```shell
$ docker build -t getting-started .
```

>   `docker build` 表示创建 image, `-t` 后接着创建的 image name, `.` 表示使用当前目录下的 Dockerfile 创建 image

### 运行 container

在当前目录下运行:

```shell
$ docker run -dp 3000:3000 getting-started
```

>   *   `-d` 标志标识让 container 运行在 `detached mode`(background)
>   *   `-p <host_port>:<container_port>` 标识将主机的 host_port 端口映射到 container 的 container_port 上(之前在 Dockerfile 中写道了当前项目需要占用端口 3000)

因为之前已经创建好了名为 `getting-started` 的 image, 否则 docker 会从 dockerhub 上 pull 对应名称的 image

### container 操作

```shell
# 查看当前运行的 container
$ docker ps
# 停止当前正在运行的 container
$ docker stop <container_id>
# 删除 container
$ docker rm <container_id>
# 直接终止并删除 container
$ docker rm -f <container_id>
```

### image 操作

```shell
# 查看当前主机中存在的 image
$ docker image ps # 或者直接 docker images
# 删除某个 image
$ docker rmi <image_name>:<tag_name> # 或者直接 docker rmi <image_id>
```

# 历史遗留问题

## 使能 BuildKit

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

## 从 spring 官网上下载 demo

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

## 创建 Dockerfile（重点）

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

