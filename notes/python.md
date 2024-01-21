# conda

管理 python 环境的工具, 因为不涉及科学计算或者大型工程, 这里使用的是 [Miniconda](https://docs.conda.io/en/latest/miniconda.html#)

安装很简单, 直接把网页上的链接 curl 下来就行, 运行 .sh 即可完成安装, 最后不要忘了 source .zshrc(.bashrc) 刷新一下当前的环境

conda 安装好后默认携带了一个版本的 python 环境

```shell
# 查看当前已经安装好的环境
$ conda info --envs
# 添加一个新的版本的 python 环境
$ conda create -n [env-name] python=[python version]
# 激活某个环境(默认激活了 base 环境)
$ conda activate [env-name]
# 失效当前环境
$ conda deactivate
```

# jupyter notebook & matplotlib

python 画图必备了属于是, 有了 miniconda 之后所有的依赖都通过 conda 安装即可:

```shell
$ conda install notebook
$ conda install matplotlib
```

装好了之后在 vscode 中创建 .ipynb 就可以画图了, 但有的时候可能需要图标携带中文, 比如修改图标的标题, 横纵坐标

默认 matplotlib 并[不支持中文字体](https://matplotlib.org/stable/users/explain/fonts.html), 所以需要额外配置字体

首先看一下当前环境下是否已经提供了中文支持:

```shell
$ fc-list :lang=zh family
```

如果一个字体都没有的话, 那么还需要为当前系统配置中文字体, 这里建议使用 [Noto Fonts - Google Fonts](https://fonts.google.com/noto/fonts?noto.continent=Asia&noto.region=CN&noto.script=Hans), 习惯上把所有的字体文件放在 `/usr/share/fonts` 下, 并且可以根据当前字体 `opentype/truetype` 和字体名称进一步细化路径

>   不需要运行 .ttf 或者 .otf 的文件, 直接存下来就行, 重新运行 `fc-list` 应该就可以看到新配置的字体了

不过可能出现 matplotlib 管理的字体名称和系统默认的名称不符合的情况, 此时可以查看一下 matplotlib 的字体列表:

>   一定要在 vscode 的 jupyter 环境下运行, 在 python shell 中运行的结果可能不太一样

```python
import matplotlib as mpl
mpl.font_manager.FontManager().ttflist
```

>   存在可能安装完 matplotlib, 更新版本后还是出现找不到 font_manager 的情况, 此时直接 import 显式引入:
>
>   ```python
>   import matplotlib.font_manager
>   ```

结果可能包含了很多种字体, 这里需要关注的是每种字体的 name 字段, 确认中文字体已经被包含在内了

>   当然还可以通过 for 循环筛选所有的 name
>
>   ```python
>   import matplotlib as mpl
>   
>   for font in mpl.font_manager.FontManager().ttflist:
>       print(font.name)
>   ```

因为有了 miniconda 进行依赖管理, 所有的依赖默认均放在了 `~/miniconda3/lib/python[version]/site-packages` 下, 这里需要配置的是 matplotlib 中的 matplotlibrc 文件, 位置在 `~/miniconda3/lib/python[version]/site-packages/matplotlib/mpl-data` 下

默认 matplotlibrc 文件都是被注释的, 需要关注的地方是:

```shell
#font.family:  sans-serif
#font.style:   normal
#font.variant: normal
#font.weight:  normal
#font.stretch: normal
#font.size:    10.0

#font.serif:      DejaVu Serif, Bitstream Vera Serif, Computer Modern Roman, New Century Schoolbook, Century Schoolbook L, Utopia, ITC Bookman, Bookman, Nimbus Roman No9 L, Times New Roman, Times, Palatino, Charter, serif
#font.sans-serif: DejaVu Sans, Bitstream Vera Sans, Computer Modern Sans Serif, Lucida Grande, Verdana, Geneva, Lucid, Arial, Helvetica, Avant Garde, sans-serif
#font.cursive:    Apple Chancery, Textile, Zapf Chancery, Sand, Script MT, Felipa, Comic Neue, Comic Sans MS, cursive
#font.fantasy:    Chicago, Charcoal, Impact, Western, Humor Sans, xkcd, fantasy
#font.monospace:  DejaVu Sans Mono, Bitstream Vera Sans Mono, Computer Modern Typewriter, Andale Mono, Nimbus Mono L, Courier New, Courier, Fixed, Terminal, monospace
```

可以看到默认选择的字体是 sans-serif, 其对应的字体为: DejaVu Sans, Bitstream Vera Sans, Computer Modern Sans Serif, Lucida Grande, Verdana, Geneva, Lucid, Arial, Helvetica, Avant Garde, sans-serif, 上面这几个就没有一个支持中文的

将其修改为:

```shell
font.family:  sans-serif
#font.style:   normal
#font.variant: normal
#font.weight:  normal
#font.stretch: normal
#font.size:    10.0

#font.serif: Noto Serif SC, DejaVu Serif
font.sans-serif: Noto Sans SC, DejaVu Sans, sans-serif
#font.cursive:    Apple Chancery, Textile, Zapf Chancery, Sand, Script MT, Felipa, Comic Neue, Comic Sans MS, cursive
#font.fantasy:    Chicago, Charcoal, Impact, Western, Humor Sans, xkcd, fantasy
#font.monospace:  DejaVu Sans Mono, Bitstream Vera Sans Mono, Computer Modern Typewriter, Andale Mono, Nimbus Mono L, Courier New, Courier, Fixed, Terminal, monospace
```

>   在修改之前最好将原来的配置进行备份

随后在 vscode 中重启 jupyter kernel, 就可以显式中文了, 如果显示失败的话还可以再通过 font_manager 看一下 matplotlib 是否已经将字体读取

```python
import matplotlib as mpl
mpl.font_manager.get_font_names()
```

>   如果并没有显示字体, 说明 matplotlibrc 的配置并没有成功

# Requests

一个用来发起 HTTP request 的库

>   [官方文档](https://docs.python-requests.org/en/latest/index.html)的标题中几个大字: HTTP for humans, 乐

使用 pip 进行包管理, 首先安装这个库

```shell
$ pip3 install requests
```

发起请求的方式真的很简单:

```python
import requests

url = 'http://localhost:8000'  # Replace with your local server URL
data = {'key': 'value'}  # Replace with your request data (optional)

response = requests.get(url, params=data)  # Replace with the appropriate HTTP method and request data

# Process the response
print('Response:', response.status_code)
print('Response Content:', response.content)
```

此外可以设置代理:

```python
import requests

url = 'http://example.com'  # Replace with the URL you want to send the request to
proxy_url = 'http://proxy.example.com:8080'  # Replace with the proxy URL and port

proxies = {
    'http': proxy_url,
    'https': proxy_url
}

response = requests.get(url, proxies=proxies)

# Process the response
print('Response:', response.status_code)
print('Response Content:', response.content)
```

# Generator object

先看看 chatgpt 的解释吧:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/06/03/22:19:47:chatgpt_generator_object_python.png)

generator object 使用 `yield` 关键字返回, 当调用 next 的时候还会重新进入继续执行, chatgpt 给出的示例表明 generator object 可以作为一个序列发生器

此外借助 send 还可以让 generator object 和外部程序通信

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/23/06/03/22:26:35:chatgpt_generator_object_send.png)

调用 send 后 generator object 重新启动, 且其会将 send 的参数作为 yeild 的返回值传入 generator object

结合上面的的例子:

*   调用定义方法 generator() 可以返回一个 generator object, 此时出于未运行状态
*   调用一次 `next()` 或者 `send(None)` 可以让 generator object 开始运行直到 yield 返回
*   再次调用 `next()` 可以让 generator object 继续执行一次, 而 `send()` 还可以将其参数作为 yeild 的返回值传入
*   借助 yeild 和 send 机制实现 generator object 和外部应用程序之间的通信

当一个 generator object 执行结束后会抛出异常: `StopIteration`, 因此正确的使用姿势是:

```python
def generator():
    yield 'done'
   
gen = generator()

while True:
    try:
        print(gen.send(None))
    except StopIteration:
        print(f'generator has terminated')
```

jyy 说这个 generator object 运行的方式类似于 c 程序: 每次调用 yield 后 generator object 都会将控制传递给应用程序, 并将 yield 的值作为参数, 这一点类似 c 程序调用 syscall, 而当外部应用程度调用 send 后, 控制又重新传入了 generator object, 并且 send 的参数作为 yield 的返回值的方式传入, 类似 c 程序从 syscall 中获取到了返回值

# 正则表达式

在 python 中使用正则表达式需要引入 re 这个库

```python
import re
```

## 匹配替换

api: `re.sub(pattern, repl, string)`, 其中 pattern 为某个字符串形式的正则表达式, string 为需要替换的文本, repl 表示了需要替换的方式, 这里的 repl 可以是一个 lambda 表达式还可以是一个方法

>   因为不会写 python 中的 lambda 表达式, 因此这里定义的是方法

repl 的方法具有如下写法:

```python
# demo.py

def process_match(match):
    # 从 group(0) 中取出的是匹配 pattern 的字符串
    matched_string = match.group(0)
    # several process
    processed_string = f(matched_string)
    # 返回的是处理之后的字符串
    return processed_string
```

从 group(0) 中返回的是符合 pattern 模式的字符串

>   如果 pattern 本身具有分组的话, 可以通过 group(i) 获取对应的分组
>
>   比如: abc(\d+), 可以调用 group(1) 获取 abc 后面的数字

需求: 对符合某个模式的字符串进行替换, 比如之前修改 typora 的 css 文件的时候修改了颜色, 需要修改 css 文件中所有满足了以 `--mono-[数字]`形式的数字, 需要让颜色变淡, 因为模板中的颜色顺序为: 50, 100, 200 ...(除了最低位的 50, 剩下的都是 100 的整数), 通过让数字变大实现更淡的颜色, 此时的 process_match 可以写成:

```python
# demo.py

import re

def process_match(match):
    number = int(match.group(1))
    if number == 50:
        number = 100
    else:
        number = number + 100
    return f'--mono-{number}'

with open(input_file, 'r') as f:
    text = f.read()

new_text = re.sub(pattern, process_match, text)

with open(output_file, 'w') as f:
    f.write(new_text)
```

## 匹配全文

无论什么时候只要是需要匹配每个位置都需要使用 `re.findAll()`

## 简单匹配

一般进行匹配的时候, 使用 re.complie 编译得到一个 pattern, 随后调用 pattern.search 或 pattern.match 进行匹配

```python
import re

text = "abc123"

# Capturing group
pattern_capturing = re.compile(r"([a-z]+)(\d+)")
match_capturing = pattern_capturing.match(text)
if match_capturing:
    print("Capturing group results:")
    print("Group 1:", match_capturing.group(1))
    print("Group 2:", match_capturing.group(2))
```

上述调用了 .match 从头开始匹配; 在上面的例子中调用 match 方法和调用 search 的结果是相同的; 二者的主要区别在于:

-   **`search`:**
    -   Searches for a pattern anywhere in the string.
    -   Returns the first match found.
    -   Matches can occur at any position in the string.
-   **`match`:**
    -   Checks for a pattern only at the beginning of the string.
    -   Returns a match only if the pattern is found at the beginning of the string.
    -   Useful when you want to check if a string starts with a particular pattern.





