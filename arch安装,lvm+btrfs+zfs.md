# arch安装,lvm+btrfs+zfs

# 简介

最大限度的使用各个技术的优点

* arch

  * 比较简单的,可以自定义的系统
* lvm

  * 动态管理卷和磁盘分区
* btrfs

  * linux内核自带的,功能强大的文件系统,同时兼容性好
* zfs

  * 最强大的文件系统,自带了三个级别的数据去重

# 参考教程

lvm教程+在lvm上面安装arch的教程+在zfs上面安装arch的教程:

<iframe src="https://wiki.archlinux.org/title/LVM_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)" data-src="" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

<iframe src="https://wiki.archlinux.org/title/Install_Arch_Linux_on_LVM" data-src="" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

<iframe src="https://wiki.archlinux.org/title/Install_Arch_Linux_on_ZFS" data-src="" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

<iframe src="https://wiki.archlinux.org/title/GRUB_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E5%AE%89%E8%A3%85_2" data-src="" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

<iframe src="https://player.bilibili.com/player.html?bvid=BV1s5411N7qN&amp;page=1&amp;high_quality=1&amp;as_wide=1&amp;allowfullscreen=true" data-src="" border="0" frameborder="no" framespacing="0" allowfullscreen="true" sandbox="allow-top-navigation-by-user-activation allow-same-origin allow-forms allow-scripts allow-popups" style="height: 360px; width: 640px;"></iframe>

# 准备工作

## 硬盘分区计划

### gpt+legacy+efi

* 分区一

  * 1MB大小未格式化空间,用于安装GRUB
* 分区二(Fat32)

  * efi分区
* 分区三(物理卷 (PV))

  * 变成lvm卷组(VG)

    * (下列都是逻辑卷(LV))
    * arch(BTRFS)  
      ==GRUB不支持zfs的全部特性,所以需要一个单独的分区存放boot,而BTRFS还可以满足快照的要求==

      * @boot
      * @root
      * @snapshots  
        使用snapper进行快照管理,自动快照
    * home(ZFS)

      * home  
        打开压缩与去重
    * (swap)
    * snapshots(lvm的快照)

[![vWBrM4.jpg](https://s1.ax1x.com/2022/08/28/vWBrM4.jpg)](https://imgse.com/i/vWBrM4)

## 连接到网络

### 确保网络接口已启用

```bash
ip link
```

输出如下

```bash
  ...
  2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state DOWN mode DEFAULT qlen 1000
  ...
```

当 `UP`在 `<BROADCAST,MULTICAST,UP,LOWER_UP>`中时，表示已启动，跟后面的 `state DOWN`没有关系

提示：可以用`ip link set *interface* up|down`来启用或关闭网络接口

### 配置网络连接

有线网络

```bash
systemctl start dhcpd
```

### 用 `ping`检查网络连接

```bash
ping -c4 8.8.8.8
```

## 更新系统时间

```bash
timedatectl set-ntp true
```

确保时间正确

```bash
timedatectl status
```

## 给iso换源

禁用reflector,并不好用

```bash
pacman -R reflector
```

将以下内容编辑到`/etc/pacman.d/mirrorlist`的顶端

```bash
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
```

## 给iso添加zfs

### 添加archzfs仓库

将以下内容添加到`/etc/pacman.conf`的末尾

```bash
[archzfs]
# Server = http://archzfs.com/$repo/$arch
Server = http://mirror.sum7.eu/archlinux/archzfs/$repo/$arch
# Server = https://mirror.biocrafting.net/archlinux/archzfs/$repo/$arch
# Server = https://mirror.in.themindsmaze.com/archzfs/$repo/$arch
# Server = https://zxcvfdsa.com/archzfs/$repo/$arch
```

仓库和软件包都已签名，因此您必须将密钥添加到 pacman 的可信密钥列表中

```bash
pacman-key -r DDF7DB817396A49B2A2723F7403BD972F75D9D76
pacman-key --lsign-key DDF7DB817396A49B2A2723F7403BD972F75D9D76
```

更新pacman数据库

```bash
pacman -Sy
```

### 给iso安装zfs模块与zfs工具

```bash
pacman -S archzfs-dkms zfs-utils
```

### 启用zfs模块

```bash
modprobe zfs
```

# 硬盘分区与文件系统创建

## 创建legacy需要的启动分区

使用BIOS引导GPT的分区情况（BIOS/GPT）下，必须使用BIOS 启动分区。GRUB将`core.img`嵌入到这个分区。

 ==**注意：**==

* ==在尝试这种方法之前请记住不是所有的系统都支持这种分区方案，请参阅 [GUID 分区表](https://wiki.archlinux.org/title/Partitioning#GUID_Partition_Table "Partitioning")。==

* ==此额外分区只由 GRUB 在 BIOS/GPT 分区方式中使用。对于 BIOS/MBR 分区方式，GRUB会把`core.img`放到MBR后间隙(post-MBR gap)中去，而在GPT中并不能保证在第一个分区之前有可以这样使用的空间。==
* ==UEFI系统也不需要这额外分区，因为它不需要嵌入启动扇区。UEFI系统需要有EFI 系统分区。==

安装 GRUB 前，在一个没有文件系统的磁盘上，创建一个1兆字节（使用fdisk或gdisk和参数`+1M`）的分区，将分区类型设置为 GUID `21686148-6449-6E6F-744E-656564454649`。

* 对于fdisk，选择分区类型 `BIOS boot`。
* 对于gdisk，选择分区类型代码 `ef02`。
* 对于parted， 在新创建的分区上设置/激活 `bios_grub` 标记。

这个分区可以处于磁盘的前 2TB 空间中的任意位置，但需要在安装 GRUB 之前创建好。分区建立好后，按下面的命令安装启动管理器。

第一个分区之前的空间也可以用作 BIOS 启动分区，但是这会违反 GPT 对齐规范。因为这个分区不会经常访问，所以性能的影响很小，只不过有些分区工具会发出警告。可以在fdisk或gdisk中创建一个从 34 扇区开始，一直到 2047扇区的分区，然后按照上述方式设置类型。为了让其它分区对齐，可以最后再创建此分区。

## 在LVM上安装Arch Linux的简介

你应该在[安装指南](https://wiki.archlinux.org/title/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))中的[分区](https://wiki.archlinux.org/title/Partitioning_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))和[创建文件系统](https://wiki.archlinux.org/title/File_systems_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E5%88%9B%E5%BB%BA%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F "File systems (简体中文)")这一步中创建LVM卷。不要直接格式化一个分区作为你的根文件系统（/），而应将其创建在一个逻辑卷（LV）中。

快速导览：

* 创建物理卷（PV）所在的分区。
* 创建物理卷（PV）。如果你只有一个硬盘，那么你最好只创建一个分区一个物理卷；如果你有多个硬盘，你可以创建多个分区，在每个分区上分别创建一个物理卷。
* 创建卷组（VG），并把所有物理卷加进卷组。
* 在卷组（VG）上创建逻辑卷（LV）。
* 继续[安装指南](https://wiki.archlinux.org/title/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E6%A0%BC%E5%BC%8F%E5%8C%96%E5%88%86%E5%8C%BA "Installation guide (简体中文)")中的格式化分区步骤。
* 当你做到安装指南中的“Initramfs”步骤时，把 `lvm2`加入到 `mkinitcpio.conf`文件中（请参考下文）。

**警告：** 若你使用不支持LVM的引导程序，`/boot`不能置于LVM中。你必须创建一个独立的`/boot`分区并直接格式化它。已知支持LVM的引导程序只有GRUB。

## 创建lvm部分,自己写的教程(gpt+legacy)(无swap)(EFI未启用)

### 安装lvm所需的软件

```bash
pacman -S lvm2
```

### 创建lvm

#### 创建物理卷(PV)

可通过以下命令列出可被用作物理卷的设备：

```bash
lvmdiskscan
```

在列出的设备上创建物理卷：

```bash
pvcreate /dev/sda3
```

#### 创建卷组(VG)

创建完成物理卷（PV）之后，下一步就是在该物理卷创建卷组（VG）了。 首先必须先在其中一个物理卷（PV）创建一个卷组：

```bash
vgcreate Arch /dev/sda3
```

#### 创建逻辑卷（LV）

创建完卷组（VG）之后，就可以开始创建逻辑卷（LV）了。输入下面命令以指定新逻辑卷的名字、大小及其所在的卷组：

```bash
lvcreate -L <卷大小> <卷组名> -n <卷名>
```

该逻辑卷创建完后，你就可以通过`/dev/mapper/Volgroup00-lvolhome`或`/dev/VolGroup00/lvolhome`来访问它。与卷组命名类似，你可以按你的需要将逻辑卷命名。

如果你想让要创建的逻辑卷拥有卷组（VG）的所有未使用空间，请使用以下命令：

```bash
lvcreate -l +100%FREE  <volume_group> -n <logical_volume>
```

==**注意：** 为了使上述命令能正常运行，你可能需要加载*device-mapper*内核模块（请使用命令 **modprobe dm-mod** ）。==

## 建立文件系统与挂载逻辑卷

现在你的逻辑卷应该已经在`/dev/mapper/`和`/dev/<i>YourVolumeGroupName</i>`中了。如果你无法在以上位置找到它，请使用以下命令来加载模块、并扫描与激活卷组：

```bash
modprobe dm-mod
vgscan
vgchange -ay
```

现在你可以在逻辑卷上创建文件系统并像普通分区一样挂载它了（如果你正在安装Arch linux，需要更详细的信息，请参考[挂载分区](https://wiki.archlinux.org/title/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E6%8C%82%E8%BD%BD%E5%88%86%E5%8C%BA "Installation guide (简体中文)")）：

**警告：** 挂载点请选择你所新建的逻辑卷（例如：`/dev/mapper/Volgroup00-lvolhome`），**不要**使用逻辑卷所在的实际分区设备（即不要使用：`/dev/sda2`）。

### btrfs部分

建立btrfs文件系统并挂载

```
mkfs.btrfs /dev/mapper/Arch-arch
mount /dev/mapper/Arch-arch /mnt
```

建立子卷

```bash
cd /mnt
btrfs subvolume create @boot
btrfs subvolume create @
btrfs subvolume create @snapshots
```

重新挂载子卷

```bash
cd /
umonut /mnt
mount /dev/mapper/Arch-arch /mnt -o subvol=@

mkdir /mnt/boot
mkdir /mnt/.snapshots

mount /dev/mapper/Arch-arch /mnt/boot -o subvol=@boot
mount /dev/mapper/Arch-arch /mnt/.snapshots -o subvol=@snapshots
```

### zfs部分

创建zfs池

```bash
mkdir /mnt/home
zpool create -O compression=zstd -O mountpoint=none -R /mnt homepool /dev/mapper/Arch-homepool -f
```

创建zfs数据集

```bash
zfs create -o dedup=on -o mountpoint=/home homepool/home
```

挂载zfs数据集

```bash
zfs mount homepool/home /mnt/home
```

# 安装系统

## 安装基本系统

```bash
pacstrap /mnt base base-devel linux-zen linux-firmware dkms zfs-dkms nano neovim networkmanager btrfs-progs
```

## 配置系统

### 生成fstab

```bash
genfstab -U /mnt > /mnt/etc/fstab
```

==记得`nano /mnt/etc/fstab`,检查lvm,zfs部分是否正确==

### 给新的系统添加archzfs仓库

将以下内容添加到`/etc/pacman.conf`的末尾

```bash
[archzfs]
# Server = http://archzfs.com/$repo/$arch
Server = http://mirror.sum7.eu/archlinux/archzfs/$repo/$arch
# Server = https://mirror.biocrafting.net/archlinux/archzfs/$repo/$arch
# Server = https://mirror.in.themindsmaze.com/archzfs/$repo/$arch
# Server = https://zxcvfdsa.com/archzfs/$repo/$arch
```

仓库和软件包都已签名，因此您必须将密钥添加到 pacman 的可信密钥列表中

```bash
pacman-key -r DDF7DB817396A49B2A2723F7403BD972F75D9D76
pacman-key --lsign-key DDF7DB817396A49B2A2723F7403BD972F75D9D76
```

### chroot

```bash
arch-chroot /mnt
pacman -Syy
```

### 时区

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc --localtime
timedatectl set-ntp 1
timedatectl set-timezone Asia/Shanghai
timedatectl set-local-rtc 1
pacman -S ntp
systemctl enable ntpd
```

### 本地化

移除 `/etc/locale.gen`对应行前面的注释符号（＃）即可，建议选择带 UTF-8 的项

```bash
/etc/locale.gen
------------------------------
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
```

```bash
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

**警告:** 不推荐在此设置任何中文 locale，会导致 TTY 乱码。

### 网络

```bash
echo archlinux > /etc/hostname
```

```bash
/etc/hosts
------------------------------
127.0.0.1	localhost
::1	        localhost
```

```bash
systemctl enable NetworkManager
```

### Initramfs

lvm与zfs的配置

```bash
/etc/mkinitcpio.conf
------------------------
HOOKS=(base systemd ... block sd-lvm2 zfs filesystems)
```

为了使 `btrfs check`能够在挂载系统上使用

```bash
/etc/mkinitcpio.conf
------------------------
MODULES=(btrfs)
BINARIES=(btrfs)
```

```
mkinitcpio -p linux-zen
```

### zfs自动导入挂载服务

```bash
systemctl enable zfs-import-cache zfs-import.target zfs-mount zfs-zed zfs.target
```

### 限制zfs的ARC(这个位置我限制的是2GB,你可以根据自己的内存适当的改大或改小)

```bash
echo "options zfs zfs_arc_max=2147483648" >> /etc/modprobe.d/zfs.conf
```

### 设置root密码

```bash
passwd
```

## 安装引导程序

如果你使用的是 `intel`或 `AMD`的cpu，安装 `microcode`

```bash
pacman -S intel-ucode
```

或者

```bash
pacman -S amd-ucode
```

如果你安装到移动u盘上，两个都要安装

安装 `grub`

```bash
pacman -S grub efibootmgr os-prober ntfs-3g
```

```bash
/etc/default/grub
--------------------
GRUB_PRELOAD_MODULES="... btrfs"
GRUB_DISABLE_OS_PROBER=false
```

```bash
grub-install --target=i386-pc /dev/sdX
grub-mkconfig -o /boot/grub/grub.cfg
```

`os-prober`是为windows和linux双系统准备的

`ntfs-3g`是为访问ntfs文件系统准备的

## 为lvm配置内核参数

编辑 `/etc/default/grub` 并将`root=/dev/mapper/Arch-arch `添加至`GRUB_CMDLINE_LINUX_DEFAULT` 行：

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

然后重新生成 `grub.cfg` 文件：

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

## 导出池,卸载文件系统

退出 `chroot`环境，输入 `exit`或者按 `Ctrl+d`

zfs

```bash
zfs umount -a
zpool export homepool
```

其他的

```bash
umount -R /mnt
poweroff
```

关机，移除安装介质，然后重启
