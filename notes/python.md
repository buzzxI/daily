# 环境

因为 linux 自带了 python 环境, 直接 vscode + wsl 就好了

先查看一下版本:
```shell
$ python3 -- version
Python 3.6.9
```

反正我在 ubuntu 18.04 中默认携带的就是这个版本

在不使用梯子的情况下，还是需要换源的, 在当前用户的根路径下修改(或者创建) .pip/pip.conf 为

```conf
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host=mirrors.aliyun.com
```

上面是修改为了阿里的镜像, 实际地址可能随着时间变化, 每次到[阿里巴巴开源镜像站-OPSX镜像站](https://developer.aliyun.com/mirror/)看一下镜像是否变化

# jupyter notebook

因为本身不使用 python 开发, 所以环境越简单越好, 就使用自带的 pip 进行包管理就行

为了避免环境变成依托, 这里使用的是虚拟环境，在 python 3.3 后可以通过 python 自带的 venv 创建虚拟环境，具体的可以参考官方文档 [venv — Creation of virtual environments](https://docs.python.org/3/library/venv.html)

然而 wsl 中的 python3 是阉割了的，需要先更新好 venv

```shell
# apt-get install venv
```

然后使用 venv 创建并激活虚拟环境

```shell
$ python3 -m venv [virtual environment path]
# 查看当前使用的 python
$ which python
```

剩下的直接去 [jupyter](https://docs.jupyter.org/en/latest/) 官网就行了

>   这里使用的是 jupyter notebook, 而不是 jupyter lab, 因为后面根本不会在浏览器上开发, 直接借助 vscode 实现代码高亮和自动补全

```shell
$ pip install notebook
```

>   期间如果报错的话可能需要升级一下 pip
>
>   ```shell
>   $ python -m pip install --upgrade pip
>   ```

如果需要使用 jupyter notebook 画图的话，还需要装一下 matplotlib

```shell
$ pip install matplotlib
```

