# arch安装笔记
## 简介
### 文件系统
gpt分区表
lvm+btrfs+zfs
### 启动方式
bios+efi
## 注意事项
本教程同时适用于用archlivecd或者用一个安装好的arch系统来安装arch
## 开始
### 分区计划(以一个256GB的硬盘为例)
-   先用 `lsblk`查看硬盘设备
-   再用`cfdisk`分区
-   sda1
    -   1MB
    -   标签BIOS boot
    -   未格式化
    -   给gpt+bios的方案安装grub用
-   sda2
    -   500MB
    -   标签EFI System
    -   fat32
    -   efi分区
-   sda3(物理卷)
    -   标签Linux LVM
    -   Arch卷组
        -   arch逻辑卷
            -   100GB
            -   btrfs
            -   @boot,@,@snapshots
        -   homepool逻辑卷
            -   100GB
            -   zfs
            -   home
        -   swap逻辑卷
            -   4GB
            -   swap
            -   swap
        -   .snapshots逻辑卷
            -   剩余全部空间
            -   未格式化
            -   lvm快照卷
### 分区
```bash
cfdisk /dev/sda
```
### 创建lvm
#### 创建物理卷(PV)
```bash
pvcreate /dev/sda3
```
#### 创建卷组(VG)
```bash
vgcreate Arch /dev/sda3
```
#### 创建逻辑卷(LV)
```bash
lvcreate -L 100G Arch -n arch
lvcreate -L 100G Arch -n homepool
lvcreate -L 4G Arch -n swap
lvcreate -l +100%FREE Arch -n .snapshots
```
### 安装`mkfs.vfat`需要的包(用一个安装好的arch系统来安装arch)
```bash
pacman -S dosfstools
```
### 格式化文件系统
```bash
mkfs.btrfs /dev/mapper/Arch-arch
mkfs.vfat /dev/sda2
mkswap /dev/mapper/Arch-swap
```
### 启用swap
```bash
swapon /dev/mapper/Arch-swap
```
### 挂载btrfs分区,建立子卷
```bash
mount /dev/mapper/Arch-arch /mnt
cd /mnt
btrfs subvolume create @boot
btrfs subvolume create @
btrfs subvolume create @snapshots
cd /
umount /mnt
```
### 挂载文件系统
```bash
mount /dev/mapper/Arch-arch /mnt -o subvol=@

mkdir /mnt/boot
mkdir /mnt/boot/efi
mkdir /mnt/.snapshots

mount /dev/mapper/Arch-arch /mnt/boot -o subvol=@boot
mount /dev/sda2 /mnt/boot/efi
mount /dev/mapper/Arch-arch /mnt/.snapshots -o subvol=@snapshots
```
### 建立并挂载zfs文件系统
```bash
zpool create -O compression=zstd -O mountpoint=none homepool /dev/mapper/Arch-homepool
zfs create -o dedup=on -o mountpoint=/mnt/home homepool/home
```
### 安装`pacstrap`(用一个安装好的arch系统来安装arch)
```bash
pacman -S arch-install-scripts
```
### 安装系统
```bash
pacstrap /mnt base base-devel linux-zen linux-zen-headers linux-firmware nano neovim networkmanager btrfs-progs lvm2 grub efibootmgr archzfs-dkms zfs-utils
```
### 生成`fstab`
```bash
genfstab -U /mnt > /mnt/etc/fstab
```
这个fstab用的是uuid,我不放心,更改了一下
```bash
/etc/fstab
------------------------------
# /dev/mapper/Arch-arch
UUID=65bcfa74-24a7-467f-bdec-e579d8686c51       /               btrfs           rw,relatime,space_cache=v2,subvolid=257,subvol=/@       0 0

# /dev/mapper/Arch-arch
UUID=65bcfa74-24a7-467f-bdec-e579d8686c51       /boot           btrfs           rw,relatime,space_cache=v2,subvolid=256,subvol=/@boot   0 0

# /dev/sdb2
UUID=543A-1903          /boot/efi       vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro   0 2

# homepool/home
# homepool/home        /home           zfs             rw,xattr,noacl  0 0

# /dev/mapper/Arch-arch
UUID=65bcfa74-24a7-467f-bdec-e579d8686c51       /.snapshots     btrfs           rw,relatime,space_cache=v2,subvolid=258,subvol=/@snapshots      0 0

# /dev/mapper/Arch-swap
UUID=c9ce44f4-8008-4edb-997e-266015dc494a       none            swap            defaults        0 0
```
### chroot
```bash
arch-chroot /mnt
```
### 时区设置
```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc --localtime
```
==如果你是用一个安装好的arch系统来安装arch,可以一起把下列命令运行==

```bash
timedatectl set-ntp 1
timedatectl set-timezone Asia/Shanghai
timedatectl set-local-rtc 1
pacman -S ntp
systemctl enable ntpd
```

### 本地化设置

移除  `/etc/locale.gen` 对应行前面的注释符号（＃）即可，建议选择带 UTF-8 的项
```bash
/etc/locale.gen
------------------------------
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
```
然后创建locale.conf文件，并编辑设定LANG变量，比如
```bash
/etc/locale.conf
------------------------------
LANG=en_US.UTF-8
```
```bash
locale-gen
```
### 网络配置
```bash
echo Archlinux > /etc/hostname 
systemctl enable NetworkManager
```
### `Initramfs`配置
lvm与zfs配置
```bash
/etc/mkinitcpio.conf
------------------------
HOOKS=(base systemd ... block lvm2 zfs filesystems ... )
```
为了使 `btrfs check`能够在挂载系统上使用
```bash
/etc/mkinitcpio.conf
------------------------
MODULES=(btrfs)
BINARIES=(btrfs)
```
重新创建一个 Initramfs
```
mkinitcpio -p linux-zen
```
### 限制zfs的ARC(这个位置的数字的单位是字节)
```bash
echo "options zfs zfs_arc_max=4294967296" >> /etc/modprobe.d/zfs.conf
```
### 设置root密码
```bash
passwd
```
### 安装微码更新
如果你使用的是 `intel`或 `AMD`的cpu，安装 `microcode`
```bash
pacman -S intel-ucode amd-ucode
```
### 安装grub
为lvm与btrfs配置文件
```bash
/etc/default/grub
--------------------
GRUB_PRELOAD_MODULES="... btrfs"
GRUB_DISABLE_OS_PROBER=false
GRUB_CMDLINE_LINUX_DEFAULT="... root=/dev/mapper/Arch-arch"
```
安装grub
```bash
grub-install --target=i386-pc /dev/sdb
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Arch --removable
```
然后生成  `grub.cfg`  文件
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
### 退出 `chroot`环境，输入 `exit`或者按 `Ctrl+d`
### 重新设置zfs数据集挂载点
```bash
zfs set mountpoint=/home homepool/home
```
### chroot
```bash
arch-chroot /mnt
```
### 开启zfs的自动导入与挂载服务
```bash
systemctl enable zfs-import-cache zfs-import.target zfs-mount zfs-zed zfs.target
```
### 退出 `chroot`环境，输入 `exit`或者按 `Ctrl+d`
### 卸载文件系统
#### zfs
```bash
zfs umount -a
zpool export homepool
```
#### 其他的
```bash
umount -R /mnt
```
### 关机，移除安装介质，然后重启