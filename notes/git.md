想学一下 git，主要是因为现在有两台设备，需要记录笔记，本身使用坚果云已经挺好的了，但是希望将笔记同步到 github 上方便保存和展示

>   毕竟坚果云在浏览的时候很多都不支持，比如目录

主要参考的是官方的书 git pro

# 起步

安装就不说了，win 下的教程在[配置 win](./配置 win.md)中有

在 linux 因为使用 rpm 或者 dpkg 很好装，centos 下就是 dnf，ubuntu 下就是 apt

首先一定需要配置就是用户信息，这个我感觉就是用来连接远程仓库用的

使用命令：

```shell
$ git config --global user.name [这里填写用户名]
$ git config --global user.email [这里填写邮箱]
```

这里可能比较疑惑的点在于，--global 的含义，下面说一下 git 的配置层级结构，git 读取配置分为三个层次：

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/git_hierarchy.png)

内部的配置会覆盖外部的配置，比如如果读取 user.name 未从 local 获取到，就读 global，最后读 system

针对三个层级，对应的三个配置文件：

*   system 级别配置文件：/etc/gitconfig，参数添加 --system 进行该层次的配置
*   global 级别配置文件：~/.gitconfig，参数添加 --global 进行该层次的配置
*   local 级别配置文件：.git/config，参数添加 --local 进行该层级的配置

>   配置文件可能不存在，这个时候就手动创建吧，git config 会识别的

所以上面的命令相当于在 ~/.gitconfig 中写入了：

```config
[user]
    name = [对应的用户名]
    email = [对应的邮箱]
```

如果现在需要配置全局的文本编辑器，那么应该写成：

```shell
$ git config --system core.editor vim
```

查看配置：

```shell
$ git config --list [--show-origin]
```

>   注意后面的参数 --show-origin 添加了这个参数在展示 git 的配置时还会展示配置文件的地址，打印当前展示配置的层级

其他的部分，查看 git config 的帮助文档：

```shell
$ git help config
# 其实和 man git config 是一样的
```

## 常用配置

这些常用配置其实都很简单，这不过时间长了命令可能会忘

**所有的配置，需要明确 global 和 local**，不管是查看，还是配置，都需要明确作用域，因为作为用户，可能不是很明确 git 的缺省作用域，这种情况下，避免出错的最好方法就是明确每条命令的作用域

>   如果是单个用户使用的话，其实全都 global 就行，但如果在多用户场景中配置是很复杂的，此时明确 global 和 local 就很重要了

### 查看配置

```shell
$ git config --[local/global] --list
```

不建议使用 

```shell
$ git config --list
```

因为语义模糊，它会把 global 配置和 local 配置混在一起，不能具体的配置

如果只需要看看某个配置

```shell
$ git config --[local/global] [配置名]
```

比如：

```shell
$ git config --local user.name
```

### 代理

主要配置的 http 和 https 代理

```shell
$ git config --global http.proxy=socks5://127.0.0.1:[对应端口]
$ git config --global https.proxy=socks5://127.0.0.1:[对应端口]
```

### 用户名和邮箱

大多数情况下 global 配置就足够了，但如果考虑多用户的场景，就需要明确每个 repo 到底属于哪个用户

```shell
$ git config --local user.name "[用户名]"
$ git config --local user.email "[邮箱]"
```

### 取消设置

```shell
$ git config --[local/global] [配置名]
```

比如有的时候这个代理并不一定比不代理更快，所以如果需要取消代理

```shell
$ git config --global --unset http.proxy
$ git config --global --unset https.proxy
```

# 简单操作

## 获取 git 仓库

这个不多数了，要么 git init 初始化一个，要么 git clone 从远程获取一个

## git 的声明周期

在仓库中的文件分为两种：已跟踪和未跟踪；简单来说，那些已跟踪的文件是指被纳入版本控制的文件，可以通过 git log 查看提交的版本，从而恢复已跟踪的文件到不同的版本；而那些未跟踪的文件就是新键的文件

已跟踪的文件具有三种状态：未修改过、已修改、进入暂存区

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/git_lifecycle.png)

新键文件时，文件处于 untracked 状态，通过 git add 该文件将处于已跟踪状态，且被放入了暂存区；

>   此时查看记录 git status 会发现有一条文件新增的记录(暂存)

此时通过 git commit 将提交新增文件的记录，此时新键的文件将处于 unmodified 状态

之后如果对该文件进行修改，再次通过 git status 将看到该文件处于 modified 状态，通过 git add 可以将其修改的变更保存到暂存区

综上，可以将 git add 理解为**将内容添加到暂存区中，供下一次提交使用**，这同时也意味着，我们进行版本控制，**所谓的版本，就是那些提交的暂存区变更**

所以，如果在 git add 后再次修改文件，此后进行 commit 后，仅仅会提交之前的修改(因为本次修改并没有通过 git add 保存在暂存区)

## 忽略文件

如果有些文件并不希望一直跟踪，可以在仓库中新建 .gitignore 文件，在 .gitignore 中添加不希望 git 跟踪的文件，避免在提交版本时将这些文件一起提交

.gitignore 的格式如下：

*   \# 表示注释

*   ! 表示取反，就是显式的写出跟踪那些文件

    >   比如 !*.java 表示跟踪所有的 .java 后缀结尾的文件

*   文件夹使用 / 结尾

    >   比如 test/ 表示忽略 test 文件夹下的所有文件

更多的内容可以看 man gitignore

有的时候某些文件已经被 git 管理了, 在更新 .gitignore 之后发现其还是被提交了; 这主要是因为 .gitignore 主要是针对于 untracked file, 而已经被提交的文件并不属于这个范围, 因此这里首先需要清理一下当前 git 管理的文件记录

```shell
# clear cache for tracked file
$ git rm --cahed [file_name]
# clear cache for all file
$ git rm -r --cached .

$ git add .
$ git commit -m ".gitignore now work"
```

## git diff

用来查看文件区别的命令

*   默认的，查看的是出于暂存区的文件和当前仓库中文件的区别

*   git diff --stage 用来对比暂存区的文件和上一次提交的文件的区别

## git commit

没什么好说的，用的很多了，现在要注意的是，如果在 commit 的时候添加参数 -a 那么 git 会将所有已经跟踪的文件暂存后进行提交(避免了 git add)

然后要注意，上面最关键的是已经跟踪的文件可以这样进行提交，那些新建的文件，因为没有被跟踪，还是需要 git add 后才能提交

## git rm

用来删除文件的，如果是单纯的 rm 的话，还需要将本次删除文件的这个行为添加到暂存区提交

要注意的是如果当前需要删除的文件处于 staged 或者 modified 状态，需要使用 git rm -f 删除

如果仅仅是希望当前文件不被 git 跟踪，而依旧保留在当前文档的目录中，需要使用 git rm --cached

## git mv

类似 git rm，使用这个命令移动的时候省的自己提交变更了

## git log

git log --oneline --decorate --all --graph

>   查看各个分支的 git log 情况

# 分支

**git 保存的不是文件的变化，而是某时刻的快照**

## git 中的快照

举例说明 git 的工作流程：假如在某个仓库中存在三个文件，均处于 modified 状态(或者是 untracked 状态，总之就是还未处于暂存区)

现在进行 git add，这个操作会计算每个文件的校验和(这里可以理解为 SHA-1 哈希算法)，git 随后创建 blob object 保存文件快照，这个文件快照的内容就包括了校验和信息

>   注意上面是一个文件对应了一个 blob object
>
>   一个 blob object 对应了一个文件的内容 

然后进行 git commit，这个操作会让 git 统计每个子目录(包括根目录)的 blob object 信息，并将这些信息保存在树对象(tree object)中，然后 git 会创建一个提交对象(commit object)，commit object 包括了 metadata 和一个指向了 tree object 对象的指针

>   metadata 中包含的信息包括了之前提到的 user.name，user.email 等信息

现在 git 中存在 5 个对象，三个 blob object，对应了仓库中三个文件的快照信息；一个 tree object，记录了当前仓库的目录结构和指向 blob 对象的指针；一个 commit object，包含了 metadata 和一个指向了 tree object 的指针

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/git_commit_tree.png)

可以看到，一次 commit 就对应了一个 commit object

当进行下一次 commit 时，新的 commit object 还会保存一个指向了上一次 commit object 的指针

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/git_commits.png)

## git 中的分支

所谓的 git 分支(比如 main 分支)，其实就是一个指针；现在考虑只存在一个 main 分支的情况，那么**每次 commit，main 指针(分支)都会指向最新的 commit object**

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/git_branch.png)



每当新建分支的时候，其实就是新建了一个指针

正常情况下，每次 commit 都会让分支的指针(比如上面的 main 指针)指向最新的 commit object，但如果是多个分支的情况下，总不能让当前 commit object 上的所有指针每次都移动吧(这样分支就没意义了)

所以存在当前分支的概念，仅当前分支上的指针会随着 commit 而指向到最新的 commit object；为了表示当前分支，git 使用了 HEAD 指针，指向了某个分支指针，被指向的那个分支指针被称为当前分支

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/git_current_branch.png)

此后再进行 commit 提交的时候仅当前分支的指针会指向最新的 commit object

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/git_switch_branch_and_commit.png)

之前也说过，git 保存的是文件某时刻的快照，如果此时切换回 master 分支，因为这个分支本身指向了一个 commit object，对应了之前的一个快照，那么此时工作目录下的文件会变回 testing 分支 commit 之前(准确来说是 testing 分支进行文件修改之前)的状态

现在如果在 master 分支上修改文件并 commit，则此时项目的提交历史将出现分叉

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/git_divergent_history.png)

每次切换分支之前，最好注意一下当前仓库中的情况，比如在暂存区中是否未提交的修改；保证每次切换之前都是 "nothing to commit, working tree clean" 的状态，不然切换过去，再切换回来，谁知道暂存区会变成什么样

## 分支的合并

一般而言都不会直接在 master 分支上直接进行修改，而是本地切换一个分支，测试没问题 commit，然后切换回 master 分支合并修改

### merge

fast-forward：如果在 test 分支上的 commit object 是 master 分支上 commit object 的后继(就是说顺着 master 分支可以直接走到 test 分支)，那么此时进行分支合并，因为没有冲突，此时的 merge 是 fast-forward，此时不会新建 commit object，仅仅是将 master 指针指向了最后的 commit object

recursive：如果被合并的分支不是当前分支的后继(比如上面的图中 master 和 test 的样子)，此时 git 会查找当前分支(比如 master)和被合并分支(比如 test)的公共祖先(上图中为 f30ab)，git 将公共祖先和 test 分支的末端(87ab2)和 master 分支的末端(c2b9e)进行一次三方合并，此时 git 会创建一个新的 commit object，即在 git log 中会新增一个合并提交

这种分支非后继分支的合并的情况可能出现冲突，此时 merge 有如下情况(我这里是新建了 a，然后两个分支分别对 a 进行了修改)：

```shell
Auto-merging a
CONFLICT (content): Merge conflict in a
Automatic merge failed; fix conflicts and then commit the result.
```

此时造成冲突的文件会被修改：

>   我在 master 分支插入了 "master insert"；而在 test分支插入了 "test insert"
>
>    ==== 隔离两个分支的内容；HEAD 表示当前分支(就是 master 分支)

```shell
<<<<<<< HEAD
master insert
=======
test insert
>>>>>>> test
```

手动消除分支就需要从二者中选择一个保留

### rebase

还是分支合并，rebase 会让在其中一个分支上的修改全部转移到另一个分支上

比如现在有这样的 git log

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/git_before_rebase.png)

如果希望将 experiment 分支 rebase 到 master 分支上

```shell
$ git checkout expreiment
$ git rebase master
```

在 rebase 后变为了：
![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/git_after_rebase.png)

随后切换回 master 分支执行 git merge，此时会执行 fast-forward 的 merge



## 分支的命令

```shell
# 创建分支，名为 test
$ git branch test
# 切换分支到 test
$ git checkout test
# 删除 test 分支
$ git branch -d test
# 直接创建分支 test 并完成切换
$ git checkout -b test
# 查看已有的分支(分支名前面加了 * 的为当前分支；这个命令不包括远程分支)
$ git branch
# 查看远程分支
$ git branch -r
```

## 远程分支

### 简单的分支管理

不用远程仓库怎么能行

远程服务器本身也存在自己的分支，自然也就包括自己的当前分支，HEAD 指针，通过 git clone，在本地自然也就保留了这些信息；而在本地开发也拥有本地的当前分支和HEAD指针，这看起来有点重复，还是举例子吧

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/git_clone_repository.png)

git clone 后本地会新增一个名为 origin/master 的分支，此外本地也具有自己的 master 分支

通过 git fetch 重新从远程仓库抓取更新 origin/master 分支，在 git fetch 后紧跟一个 git merge 可以将本地分支和远程分支合并

```shell
# 将名为 origin 的远程仓库同步到本地
$ git fetch origin
# 将远程分支 orign/master 并入本地的 master 分支
$ git merge
```

>   如果只有一个远程分支的话，还是 git pull 吧

使用 git pull 可以将本地分支推送到远程仓库

```shell
# 一般的写法为 $ git pull [远程仓库名，一般为 origin] [分支]
# 将本地的 main 分支推送到远程名为 origin 仓库的 main 分支
# 注意这个远程仓库名是本地为了区分不同远程仓库的名字，并不是 github 上的仓库名
$ git pull origin main main
# 当然可以简写
$ git pull
```

### 推送本地分支到远程

本地的 git 仓库和 github 上的远程仓库之间的关联和解绑其实是有点麻烦的, 所以之前一直都是 github 上创建一个仓库, 然后 clone 到本地, 在本地进行修改

这里主要考虑将本地创建的仓库推到远程, 首先将本地仓库和远程仓库关联:

```shell
$ git remote add [远程主机名] [仓库地址]
```

这里注意:

*   一般习惯上远程主机名都写成了 origin, 这个就是个人喜好的问题
*   仓库地址可以是一个 HTTP 开头的地址(此时可能需要输入 github 的用户名和密码)也可以是一个 git 开头的地址(通过 SSH key 进行关联), 个人建议为 github 账户配置好 ssh publicy key, 通过 ssh 关联 github 账户, 更安全一点

然后将本地分支推到远程, 如果远程不存在对应的远程分支, 那么会创建一个新的远程分支

```shell
$ git push -u [远程主机名] [本地分支名]:[远程分支名]
```

这里注意:

*   可以不写远程分支名, 此时创建的远程分支和本地相同
*   可以不写本地分支名, 此时相当于将远程分支删除
*   默认的使用 `git push` 其实就是把本地的当前分支更新到远程的对应分支

>   ```shell
>   # 一次性可以推送所有分支到远程 (不建议)
>   $ git push -u [远程主机名] --all
>   ```
>
>   将多个本地分支推送到 github 后, 会自动创建 PR, 十分醒目, 我这里需要推送的的时候, 会为每个分支创建一个 PR, 然后关闭

### 获取远程分支到本地

有的时候需要从远程仓库获取本地不存在的分支

*   如果需要将远程分支的修改同步到当前分支:

    ```shell
    $ git pull [远程主机名] [远程分支名]:[本地分支名]
    ```

    >   所以 `push` 和 `pull` 本身是有很多参数的, 对于分支名, 如果是 `push` 表示将本地分支推到远程, 因此参数顺序是: `[本地分支]:[远程分支]`, 而如果是 `pull` 表示将远程分支拉取到本地, 因此参数顺序是: `[远程分支]:[本地分支]`

*   如果只是希望将远程分支同步到本地的新分支上:

    ```shell
    $ git checkout -b [本地分支名] [远程主机名]/[远程分支名]
    ```

### 其他配置

在进行各种命令的时候, 不免需要让本地仓库和远程分支断开关系:

```shell
$ git remote remove [远程主机名]
```

最后为了查看本地仓库和远程仓库之间是否建立了正确的关系, 需要查看本地仓库和远程仓库的关联信息

```shell
# 查看本地仓库和远程仓库之间的关联信息
$ git remote -v

# 查看本地分支和远程分支的关联信息
$ git branch -vv
```

# 代理

如果 git clone 失败的话，可以尝试通过为 git 配置本机代理：

```shell
$ git config --global http.proxy socks5://${ip}:${port}
$ git config --global https.proxy socks5://${ip}:${port}
```

一般的话还是不要持久化了，用完了就取消掉代理：

```shell
$ git config --global --unset http.proxy
$ git config --global --unset https.proxy
```

# github_cli

查[手册](https://github.com/cli/cli)吧, 挺全面的, 目前看来最大的作用是将代码以 gist 的形式推到 github 上
