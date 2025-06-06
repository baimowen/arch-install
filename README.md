# 手动安装 archlinux

*   [前期准备](##前期准备)
*   [设置字体大小](##设置字体大小)
*   [更新包管理](##更新包管理)
*   [分区管理](##分区管理)
*   

<div STYLE="page-break-after: always;"></div>

## 前期准备

*   一个刷写好archlinux live的U盘
*   一台机器

---

## 设置字体大小

```bash
setfont ter-132n  # 正常 较大
or
setfont ter-122b  # 粗体 较小
```

---

## 网络设置

>   [!note]
>
>   有线连接一般dhcp自动获取ip并连接
>
>   无线连接需使用iwctl手动联网

```bash
iwctl
# 以下为iwctl命令行
device list                             # 列出网卡列表
station <nic_name> scan                 # 扫描无线网络
station <nic_name> get-networks         # 列出已扫描的无线网络
station <nic_name> connect <wlan_name>  # 连接无线网
exit
# 以下为sh环境
ping www.baidu.com  # 检测网络连通性
or
ip a                # 查看网络连接信息
```

---

## 更新包管理

包管理镜像源列表位置在：`/etc/pacman.d/mirrorlist`

```bash
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak  # 备份镜像源列表
reflector --country China \        # 设置镜像源地区
  --age 24 \                       # 最近24h
  --sort rate \                    # 排序：以传输速度排序
  --protocol https \               # 镜像源所用传输协议 https为非明文传输 http为明文传输
  --save /etc/pacman.d/mirrorlist  # 生成的镜像源列表路径
# more /etc/pacman.d/mirrorlist    # 检查镜像源列表
pacman -Syu
```

---

## 分区管理

```bash
lsblk           # 查看分区表
gdisk /dev/sda  # 进入gdisk模式为/dev/sda磁盘分区
gdisk /dev/sdb
```

>   [!note]
>
>   fdisk、gdisk 交互式分区管理工具，fdisk、gdisk 分别管理 mbr、gpt分区
>
>   parted 命令式分区管理工具
>
>   对应关系：gpt -> UEFI  mbr -> BIOS

<div style="float: left; width: 50">
创建分区过程：<br>
gdisk /dev/sda<br>
o → y # 创建分区表<br>
n → 1 → (默认起始) → +512M → EF00 # EFI分区<br>
n → 2 → (默认起始) → +17G → 8E00 # LVM分区<br>
n → 3 → (默认起始) → +1G → 8200 # Swap分区<br>
n → 4 → (默认起始) → (默认结束) → 8300 # Btrfs快照分区<br>
w → y<br>
<br>
fdisk/gdisk常用参数：<br>
Options:<br>
- d&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;delete a partition<br>
- i&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;show detailed information on a partition<br>
- l&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;list known partition types<br>
- n&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;add a new partition<br>
- p&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;print the partition table<br>
- w&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;write table to disk and exit<br>
- ?&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;print this menu<br>
<br>
常用分区对应十六进制码：<br>
8200 Linux swap<br>
8300 Linux filesystem<br>
8302 Linux /home<br>
ef00 EFI<br>
8e00 LVM<br>
<br>
检查分区信息：<br>
gdisk -l /dev/sda<br>
</div>
<div style="float: right; width: 50%;">
<img src="pic/create efi.png" style="zoom: 110%" />
<img src="pic/create filesystem.png" style="zoom: 110%" />
<img src="pic/create swap.png" style="zoom: 110%" />
<img src="pic/print disk partition.png" style="zoom: 110%" />
</div>
<div style="clear: both;"></div>

分区规划（使用GPT分区表）

系统盘 (/dev/sda)

| 分区      | 大小     | 类型        | 文件系统 | 用途                |
| :-------- | :------- | :---------- | :------- | :------------------ |
| /dev/sda1 | 512M     | EFI系统分区 | FAT32    | /boot/efi           |
| /dev/sda2 | 17G      | Linux LVM   | -        | LVM物理卷（系统VG） |
| /dev/sda3 | 1G       | Linux swap  | swap     | 交换空间（512M~1G） |
| /dev/sda4 | 剩余空间 |             | btrfs    | 系统快照存储        |

数据盘 (/dev/sdb)

| 分区      | 大小 | 类型      | 文件系统 | 用途                |
| :-------- | :--- | :-------- | :------- | :------------------ |
| /dev/sdb1 | 全部 | Linux LVM | -        | LVM物理卷（数据VG） |

```bash
# ===================== 系统盘：sda =====================
mkfs.fat -F32 /dev/sda1   # 格式化EFI分区

pvcreate /dev/sda2        # 创建物理卷
vgcreate sysvg /dev/sda2  # 创建卷组（命名为sysvg）

# 创建逻辑卷
lvcreate -L 16G -n root sysvg       # 根分区
lvcreate -L 100%FREE -n var sysvg   # 日志分区

# 格式化逻辑分区
mkfs.btrfs -f -L "ROOT" /dev/sysvg/root  # 格式化根分区
mkfs.btrfs -f -L "VAR" /dev/sysvg/var    # 格式化可变数据分区

mount /dev/sysvg/root /mnt  # 挂载根分区
# 创建btrfs子卷结构
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@var

# 设置默认子卷
btrfs subvolume set-default 256 /mnt  # 256通常是第一个创建的子卷的ID
or
btrfs subvolume list /mnt
btrfs subvolume set-default <子卷ID> /mnt

umount /mnt  # 卸载临时挂载

vgcreate snapvg /dev/sda4                     # 创建卷组
lvcreate -l 100%FREE -n all_snapshots snapvg  # 创建逻辑卷
mkfs.btrfs -f -L "SNAPSHOTS" /dev/sda4        # 格式化逻辑分区
mount /dev/snapvg/all_snapshots /mnt          # 挂载临时分区
btrfs subvolume create /mnt/@all_snapshots    # 创建子卷
umount /mnt                                   # 卸载临时挂载

# ===================== 数据盘：sdb =====================
pvcreate /dev/sdb1                        # 创建物理卷
vgcreate datavg /dev/sdb1                 # 创建卷组（命名为datavg）
lvcreate -L 13G -n home datavg            # 用户数据 /home
lvcreate -L 5G -n etc datavg              # 配置备份 /etc、/var
lvcreate -l 100%FREE -n otherdata datavg  # 其他数据

# 格式化并配置/home
mkfs.btrfs -f -L "HOME" /dev/datavg/home
mount /dev/datavg/home /mnt
btrfs subvolume create /mnt/@home
umount /mnt

# 格式化并配置/etc备份
mkfs.btrfs -f -L "ETC" /dev/datavg/etc
mount /dev/datavg/etc /mnt
btrfs subvolume create /mnt/@configs
btrfs subvolume create /mnt/@ssh_keys
umount /mnt

# 格式化数据存储
mkfs.btrfs -f -L "OTHERDATE" /dev/datavg/otherdata


# 激活所有卷组
vgchange -ay

# 检查逻辑卷状态
lvscan

# 检查LVM状态
vgs && lvs && pvs

# 检查Btrfs子卷
btrfs subvolume list /

# 查看空间使用
btrfs filesystem usage /
```

>   [!tip]
>
>   lvdisplay /dev/<卷组>/<逻辑卷>             # 查看逻辑卷
>
>   lvremove /dev/<卷组>/<逻辑卷>              # 删除逻辑卷
>
>   lvextend -l +100%FREE /dev/<卷组>/<逻辑卷> # 扩容逻辑卷
>
>   vgremove <卷组>                           # 删除卷组（前提删除卷组内所有逻辑卷）
>
>   btrfs subvolume list show                # 列出btrfs所有子卷
>
>
>   btrfs subvolume delete /mnt/<子卷名>      # 删除btrfs子卷（挂载状态）

>   chattr +C \<dir\>  # 禁用指定目录COW