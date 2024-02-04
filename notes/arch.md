# 安装

要注意的是, 在开始安装之前需要先进入 bios 关闭安全启动

基本的思路直接参考: [Installation guide - ArchWiki (archlinux.org)](https://wiki.archlinux.org/title/installation_guide)

首先检查一下是否启用了 UEFI 模式:

```shell
$ ls /sys/firmware/efi/efivars
```

>   只要有这个目录就说说明启用了 UEFI

然后通过 fdisk 完成磁盘分区, 这里需要一个 EFI 分区 (512 MB, EFI 类型分区), root 分区 (剩下的所有空间分配给 root, Linux 类型分区)

>   swap 分区通过挂载 swapfile 实现

然后分别在两个分区中创建文件系统, 其中 EFI 分区使用 FAT32 格式, root 分区使用 Ext4 格式, 随后进行文件系统的挂载, 这里需要先挂载 root 分区 (在有其他分区的情况下)

然后需要需要让 arch 连上网, 有网线的, 直接连上应该就能获取 ip 了, 没有网线就需要通过 iwctl 连上 wifi 了

连上网之后, 需要修改 mirror list, 默认的 repo 太慢了, 并且还没有代理, 这一步很有必要, 更换 mirror list 需要依赖工具 reflector (先使用 pacman 获取一下)

这步结束之后才正式开始 arch 的安装: base, linux, linux-firmware, vim

完成 arch 的下载后, 需要对其进行配置, 使其可以获取到磁盘分区信息 -> 之前的分区都只是在临时的 u 盘中的 arch 而言的, 后面需要也让新下好的 arch 也知道分区信息 -> 生成 fstab 信息

此后通过 chroot 进入新下好的 arch 系统

首先需要修改时区信息, 这里配置为 Asia/Shanghai 即可; 然后设置 locale, 为了避免 tty 的乱码问题, 这里设置为 en_US.UTF-8

尽管临时 arch 的网络已经配置好了, 但新的 arch 还没有进行网络配置 -> 配置 hostname 并安装一个网络管理工具, NetworkManager 或者 systemd-networkd

随后修改 root 密码: 这里是唯一的一次机会, 错过了就重装系统吧...

然后安装一个 bootloader, 因为是 UEFI 启动, 因此使用 Grub 作为 bootloader -> 从这个时候开始完成 EFI 分区的挂载

>   这一步不要忘了安装对应处理器的 microcode

到此为止, 都是一个 root 一梭子到底, 显然不太合适, 重启之前还需要额外添加一个用户

>   最后的话就是桌面环境了, 要么是 KDE, 要么是 Gnome, 随便吧



