---
title: "Linux 文件系统"
summary: "讲解 VFS 抽象与 inode、ext4、xfs、btrfs、f2fs 等主流文件系统的取舍、日志与 CoW 机制，以及 mount、fstab、lsblk 等挂载与维护命令"
date: 2026-09-26
categories: [Infra, Linux]
tags: [linux]
---

## 概述

- 参考文档：[Arch Wiki: File systems](https://wiki.archlinux.org/title/File_systems)
- 文件系统是组织存储设备上数据的一套数据结构与算法，Linux 通过 VFS 给上层一套统一接口
- 层次结构：

  ```mermaid
  flowchart LR
    A["应用程序"] --> B["VFS"]
    B --> C["ext4 / xfs / btrfs"]
    B --> D["FUSE(用户态)"]
    B --> E["NFS / 9p(网络)"]
    C & D & E --> F["页缓存与块设备"]
  ```

- "一切皆文件"：普通文件、目录、设备节点、管道、socket、`procfs`里的内核状态，都走`open`/`read`/`write`这一套

## VFS 与 inode

- VFS 的四个核心对象：
  - `superblock`：一个已挂载文件系统的实例
  - `inode`：文件的元数据(权限、大小、时间戳、数据块指针)，**inode 里没有文件名**
  - `dentry`：目录项，把文件名映射到 inode，串成目录树
  - `file`：进程打开文件后的实例，含读写偏移
- 硬链接是同一 inode 的多个 dentry，软链接是存目标路径的特殊文件，基础操作见[文件系统](/2024/03/12/linux-file/)
- 小文件海量场景会"有空间没 inode"：`df -i`看 inode 余量，`df -h`看空间余量，两个都要看

## 主流文件系统

- `ext4`：
  - 日志型，发行版最稳妥的默认，`e4defrag`可做在线碎片整理
  - 日志默认只保证元数据一致(`data=ordered`)，断电后结构完整但文件内容可能缺尾
- `xfs`：
  - RHEL 系默认，擅长大文件与高并发，分配组带来并行扩展性
  - **只支持扩容(`xfs_growfs`)不支持收缩**，规划容量时想清楚
- `btrfs`：
  - CoW(写时复制)型：快照、压缩(`compress=zstd`)、子卷、数据校验、多设备一体化
  - CoW 带来写放大，数据库等重随机写负载注意`nodatacow`逃生门
  - openSUSE 与 CachyOS 的默认选择
- `f2fs`：为闪存量身设计的日志结构文件系统，手机与嵌入式常见
- `zfs`：功能最全的 CoW 系，CDDL 许可与 GPL 不兼容，只能以外部模块加载
- 互操作格式：`ntfs3`(内核原生可写 NTFS)、`exfat`与`vfat`(U 盘与 EFI 分区)、`iso9660`(光盘镜像)
- 伪文件系统：`proc`、`sysfs`、`tmpfs`(内存盘，`/tmp`常见)、`devtmpfs`，只存在于内存不落盘
- `overlayfs`：把只读层与可写层叠加，容器镜像与不可变发行版的基石

## 挂载

- 手动挂载与卸载：

  ```console
  $ mount -t btrfs -o compress=zstd /dev/nvme0n1p2 /mnt/data
  $ umount /mnt/data
  ```

  - 设备用`UUID=`或`LABEL=`指定，**比`/dev/sda1`可靠**，盘序会变而 UUID 不会
  - `mount -o remount,rw /`运行中改挂载选项
  - `umount`报`target is busy`时用`lsof +D 挂载点`找占用者，详见[Linux 进程管理命令](/2026/09/28/linux-process-cli/)
- 开机挂载写`/etc/fstab`：

  ```fstab
  # <设备>            <挂载点>   <类型>   <选项>                    <dump> <pass>
  UUID=XXXX-XXXX      /boot/efi  vfat    defaults,umask=0077       0      2
  UUID=uuid-of-root   /          btrfs   subvol=/@,compress=zstd   0      1
  /swapfile           none       swap    sw                        0      0
  tmpfs               /tmp       tmpfs   defaults,nosuid,nodev     0      0
  ```

  - `pass`列：`1`给需要`fsck`的根分区，`2`给其它要检查的，`0`跳过
  - **写完先`findmnt --verify`校验，fstab 写错会导致开机进紧急模式**
- 常用挂载选项：`noatime`不更新访问时间(性能常用)、`ro`只读、`user`允许普通用户挂载、`umask`控制权限掩码

## 相关命令

- 查看现状：

  ```console
  $ lsblk -f            # 设备树 + 文件系统与 UUID
  $ findmnt             # 当前挂载表，比裸 cat /proc/mounts 友好
  $ df -T               # 占用与类型
  $ du -sh 目录          # 目录占用
  ```

- 创建与检查：
  - `mkfs.ext4 /dev/sdX2`、`mkfs.btrfs -L data /dev/sdX2`建文件系统，分区用`fdisk`或`parted`
  - `fsck.ext4 -f`修复(需先卸载)、`xfs_repair`、`btrfs check`
- btrfs 日常：

  ```console
  $ btrfs subvolume create /mnt/data/@snap      # 建子卷
  $ btrfs subvolume snapshot /mnt/data/@snap /mnt/data/@snap-1017   # 快照是 O(1) 的
  $ btrfs subvolume list /mnt/data
  $ btrfs filesystem usage /mnt/data            # 比 df 准确
  ```

## 相关阅读

- [文件系统](/2024/03/12/linux-file/)：文件权限、链接与基础命令
- [操作系统的文件系统](/2025/06/10/os-filesystem/)：课程层面的设计与理论
- [Linux 发行版与包管理器](/2026/09/08/linux-distro-package-manager/)：各家默认文件系统的选择
