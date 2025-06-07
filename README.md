# 手动安装 archlinux

*   [前期准备](##前期准备)
*   [设置字体大小](##设置字体大小)
*   [更新包管理](##更新包管理)
*   [分区管理](##分区管理)
    *   [分区规划](###分区规划（使用GPT分区表）)
        *   [系统盘](####系统盘（/dev/sda）)
        *   [数据盘](####数据盘（/dev/sdb）)

    *   [对系统盘进行分区](###对系统盘进行分区)
    *   [对数据盘进行分区](###对数据盘进行分区)
    *   [挂载文件系统](###挂载文件系统)
    *   [一些维护命令](###一些维护命令)
*   [安装操作系统](##安装操作系统)
*   [生成分区表](##生成分区表)
*   [设置时区、编码](##设置时区、编码)
*   [设置主机名](##设置主机名)
*   [网络配置](##网络配置)
*   [添加用户](##添加用户)
*   [安装引导](##安装引导)
*   [验证](##验证)
*   [完成安装](##完成安装)

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
reflector --country China --age 24 --sort rate --protocol https --save /etc/pacman.d/mirrorlist  
# more /etc/pacman.d/mirrorlist    # 检查镜像源列表
pacman -Syu --noconfirm
```

*   `--country`：设置镜像源地区
*   `--age`：最近{num}h
*   `--sort`：排序：以传输速度排序
*   `--protocol`：镜像源所用传输协议 https为非明文传输 http为明文传输
*   `--save`：生成的镜像源列表路径

---

## 分区管理

```bash
lsblk -o NAME,UUID,MOUNTPOINT           # 查看分区表
lsblk -f /dev/sda{num}
gdisk /dev/sda                          # 进入gdisk模式为/dev/sda磁盘分区
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

### 分区规划（使用GPT分区表）

#### 系统盘 (/dev/sda)

| 分区      | 大小     | 类型        | 文件系统 | 用途                |
| :-------- | :------- | :---------- | :------- | :------------------ |
| /dev/sda1 | 512M     | EFI系统分区 | FAT32    | /boot/efi           |
| /dev/sda2 | 17G      | Linux LVM   | -        | LVM物理卷（系统VG） |
| /dev/sda3 | 1G       | Linux swap  | swap     | 交换空间（512M~1G） |
| /dev/sda4 | 剩余空间 |             | btrfs    | 系统快照存储        |

#### 数据盘 (/dev/sdb)

| 分区      | 大小 | 类型      | 文件系统 | 用途                |
| :-------- | :--- | :-------- | :------- | :------------------ |
| /dev/sdb1 | 全部 | Linux LVM | -        | LVM物理卷（数据VG） |

### 对系统盘进行分区

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

mount /dev/sysvg/root /mnt            # 临时挂载分区
btrfs subvolume create /mnt/@         # 创建btrfs子卷结构
btrfs subvolume create /mnt/@var
btrfs subvolume set-default 256 /mnt  # 设置默认子卷，256通常是第一个创建的子卷的ID
or
btrfs subvolume list /mnt
btrfs subvolume set-default <子卷ID> /mnt
umount /mnt                           # 卸载临时挂载

mkswap -f /dev/sda3  # 格式化swap分区
swapon /dev/sda3     # 启用交换分区
swapon --show        # 查看交换分区大小
or
free -h

vgcreate snapvg /dev/sda4                     # 创建卷组
lvcreate -l 100%FREE -n all_snapshots snapvg  # 创建逻辑卷
mkfs.btrfs -f -L "SNAPSHOTS" /dev/sda4        # 格式化逻辑分区
mount /dev/snapvg/all_snapshots /mnt          # 挂载临时分区
btrfs subvolume create /mnt/@all_snapshots    # 创建子卷
umount /mnt                                   # 卸载临时挂载
```

### 对数据盘进行分区

```bash
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
```

>   [!tip]
>
>   对于卷等操作：
>
>   ```bash
>   lvdisplay /dev/<卷组>/<逻辑卷>              # 查看逻辑卷
>   lvremove /dev/<卷组>/<逻辑卷>               # 删除逻辑卷
>   lvextend -l +100%FREE /dev/<卷组>/<逻辑卷>  # 扩容逻辑卷
>   vgremove <卷组>                            # 删除卷组（前提删除卷组内所有逻辑卷）
>   btrfs subvolume list show                 # 列出btrfs所有子卷
>   btrfs subvolume delete /mnt/<子卷名>       # 删除btrfs子卷（挂载状态）
>   ```

### 挂载文件系统

```bash
# ===================== 挂载文件系统 =====================
vgchange -ay       # 激活所有卷组
lvscan             # 检查逻辑卷状态
vgs && lvs && pvs  # 检查LVM状态

mount UUID=a795255d-8ee7-4c15-9dd6-831364ff77ee /mnt -o subvol=@  # 首先挂载根分区
mkdir -p /mnt/{boot/efi,var,etc,home,snapshots,otherdata}         # 创建挂载点目录结构
# 挂载分区
mount /dev/sda1 /mnt/boot/efi
mount UUID=14190dae-1e89-4636-8d1a-87e9a46e4b0d /mnt/var -o subvol=@var
mount UUID=27333bb2-7961-4406-881a-316d2dfe1718 /mnt/snapshots -o subvol=@all_snapshots
mount UUID=c5418a7f-2f2b-492c-a8c3-464d60b6881b /mnt/etc -o subvol=@configs
mount UUID=9788b05b-6d48-4158-b527-3e1ce1f61b62 /mnt/home -o subvol=@home
mount UUID=02c8fb43-6a24-474d-8ecf-2997ff7b9d98 /mnt/otherdata

systemctl daemon-reload

# 检查所有挂载点
mount | grep btrfs

# btrfs子卷检查
btrfs subvolume list /mnt/<子卷名>
# eg: btrfs subvolume list /mnt/home
```

### 一些维护命令

```bash
# ===================== 维护命令 =====================

sudo btrfs subvolume list /                # 检查Btrfs子卷
sudo btrfs filesystem usage /              # 查看空间使用
sudo btrfs filesystem defrag -r /mnt/data  # 碎片整理

sudo rsync -av --delete /etc/ /mnt/etc/                                  # 定期备份/etc
sudo btrfs subvolume snapshot / /.snapshots/$(date +%Y%m%d)              # 创建系统快照
sudo btrfs subvolume snapshot /home /mnt/snapshots/home_$(date +%Y%m%d)  # 创建数据快照

sudo chattr +c /     # 禁用CoW对系统目录
sudo chattr +C /var  # 禁用CoW对日志目录
```

>   我的分区情况如下：
>
>   ```
>   root@archiso ~ # lsblk -f
>   NAME                     FSTYPE      FSVER            LABEL       UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
>   loop0                    squashfs    4.0                                                                       0   100% /run/archiso/airootfs
>   sda                                                                                                                     
>   ├─sda1                   vfat        FAT32                        3F1D-C638                                14.5G     0% /mnt/boot/efi
>   ├─sda2                   LVM2_member LVM2 001                     J3Ep2y-0qEr-x7Mj-kwdH-vxtz-ftYb-kxQXkf                
>   │ ├─sysvg-root           btrfs                        ROOT        a795255d-8ee7-4c15-9dd6-831364ff77ee     14.5G     0% /mnt
>   │ └─sysvg-var            btrfs                        VAR         14190dae-1e89-4636-8d1a-87e9a46e4b0d    904.6M     1% /mnt/var
>   ├─sda3                   swap        1                            355c95c7-dfc8-45de-9446-03598c7a677c                  [SWAP]
>   └─sda4                   LVM2_member LVM2 001                     29l4C6-DBtN-DSP3-cyTH-cDm3-AIsB-KtWtDo                
>     └─snapvg-all_snapshots btrfs                        SNAPSHOTS   27333bb2-7961-4406-881a-316d2dfe1718      1.3G     0% /mnt/snapshots
>   sdb                                                                                                                     
>   └─sdb1                   LVM2_member LVM2 001                     LVkKOb-cEnr-ejsa-HJ6E-mDns-o2RE-2pGudm                
>     ├─datavg-home          btrfs                        HOME        9788b05b-6d48-4158-b527-3e1ce1f61b62     12.5G     0% /mnt/home
>     ├─datavg-etc           btrfs                        ETC         c5418a7f-2f2b-492c-a8c3-464d60b6881b      4.5G     0% /mnt/etc
>     └─datavg-otherdata     btrfs                        OTHERDATE   02c8fb43-6a24-474d-8ecf-2997ff7b9d98      1.8G     0% /mnt/otherdata
>   sr0                      iso9660     Joliet Extension ARCH_202506 2025-06-01-09-10-39-00                       0   100% /run/archiso/bootmnt
>   root@archiso ~ # mount | grep btrfs
>   /dev/mapper/sysvg-root on /mnt type btrfs (rw,relatime,space_cache=v2,subvolid=256,subvol=/@)
>   /dev/mapper/sysvg-var on /mnt/var type btrfs (rw,relatime,space_cache=v2,subvolid=256,subvol=/@var)
>   /dev/mapper/snapvg-all_snapshots on /mnt/snapshots type btrfs (rw,relatime,space_cache=v2,subvolid=256,subvol=/@all_snapshots)
>   /dev/mapper/datavg-etc on /mnt/etc type btrfs (rw,relatime,space_cache=v2,subvolid=256,subvol=/@configs)
>   /dev/mapper/datavg-home on /mnt/home type btrfs (rw,relatime,space_cache=v2,subvolid=256,subvol=/@home)
>   /dev/mapper/datavg-otherdata on /mnt/otherdata type btrfs (rw,relatime,space_cache=v2,subvolid=5,subvol=/)
>   ```

---

## 安装操作系统

```bash
pacstrap /mnt linux linux-firmware linux-headers base base-devel vim bash-completion
arch-chroot /mnt
pacman -S --noconfirm btrfs-progs lvm2 sudo nano dhcpcd openssh
```

-   `btrfs-progs`：Btrfs 文件系统工具
-   `lvm2`：LVM 管理工具
-   `nano`：文本编辑器
-   `sudo`：权限管理
-   `dhcpcd`：网络连接

>   [!note]
>
>   `arch-chroot /mnt` 将当前的根目录切换到 `/mnt`，即进入挂载的目标文件系统。
>
>   `pacstrap` 这是一个 Arch Linux 安装工具，用于将指定的软件包安装到目标挂载点（这里是 `/mnt`）。`pacstrap` 是 `pacman` 的一个封装工具，专门用于安装初始的系统软件包。
>
>   区别：`arch-chroot /mnt` 可以执行更为复杂的配置，而 `pacstrap` 则仅向目标系统中安装软件包

>   [!warning]
>
>   全新安装时应首先使用 `pacstrap` 安装linux内核和工具包，否则 `chroot` 直接进入会报错，未安装系统



## 生成分区表



## 设置时区、编码



## 设置主机名



## 网络配置



## 添加用户



## 安装引导



## 验证



## 完成安装

