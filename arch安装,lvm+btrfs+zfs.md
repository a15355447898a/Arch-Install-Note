# arch安装,lvm+btrfs+zfs(尝试过的版本)

- 在VMware Workstation Pro中用archiso建立了一个新的虚拟机,测试arch安装,lvm+btrfs+zfs

  - 连接网络
    ```bash
    systemctl start dhcpcd.service
    ```
  - 更新系统时间
    ```bash
    timedatectl set-ntp true
    ```
  - 换源
    - 先把`reflector​`删了,这个不好用
      ```bash
      pacman -R reflector
      ```
    - 将以下内容编辑到 `/etc/pacman.d/mirrorlist` 的顶端​
      ```bash
      Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
      ```
  - 安装zfs前面需要先导入archzfs仓库密钥,但是密钥的网站需要代理, VMware Workstation Pro走宿主机代理我整不起来,用[[proxychains]]连接宿主机[[v2raya]]的端口分享时显示连接超时
  - 在livecd中安装zfs发现空间不够,不能够安装
    - 思路改变
      - ==先把home放入btrfs的@home中,安装完成后再更改为zfs==
  - 分区计划
    - 先用` lsblk`查看硬盘设备
    - 再用`cfdisk`分区
    - sda1
      - 1MB
      - 未格式化
      - 给gpt+bios的方案安装grub用
    - sda2
      - 500MB
      - fat32
      - efi分区
    - sda3(物理卷)
      - Arch卷组
        - arch逻辑卷
          - 10GB
          - btrfs
          - @boot,@,@home,@snapshots
        - homepool逻辑卷
          - 5GB
          - 未格式化
          - 之后格式化为zfs放/home
        - .snapshots逻辑卷
          - 剩余全部空间
          - 未格式化
          - lvm快照卷
  - 分区
    ```bash
    cfdisk /dev
    ```
  - 创建lvm
    - 创建物理卷(PV)
      ```bash
      pvcreate /dev/sda3
      ```
    - 创建卷组(VG)
      ```bash
      vgcreate Arch /dev/sda3
      ```
    - 创建逻辑卷(LV)
      ```bash
      lvcreate -L 10G Arch -n arch
      lvcreate -L 5G Arch -n homepool
      lvcreate -L +100%FREE Arch -n .snapshots
      ```
  - 格式化文件系统
    ```bash
    mkfs.btrfs /dev/mapper/Arch-arch
    mkfs.vfat /dev/sda2
    ```
  - 挂载btrfs分区,建立子卷
    ```bash
    mount /dev/mapper/Arch-arch /mnt
    cd /mnt
    btrfs subvolume create @boot
    btrfs subvolume create @
    btrfs subvolume create @home
    btrfs subvolume create @snapshots
    cd /
    umount /mnt
    ```
  - 挂载文件系统
    ```bash
    mount /dev/mapper/Arch-arch /mnt -o subvol=@

    mkdir /mnt/boot
    mkdir /mnt/boot
    mkdir /mnt/home
    mkdir /mnt/.snapshots

    mount /dev/mapper/Arch-arch /mnt/boot -o subvol=@boot
    mount /dev/sda2 /boot/efi
    mount /dev/mapper/Arch-arch /mnt/home -o subvol=@home
    mount /dev/mapper/Arch-arch /mnt/.snapshots -o subvol=@snapshots
    ```
  - 安装系统
    ```bash
    pacstrap /mnt base base-devel linux-zen linux-firmware nano neovim networkmanager btrfs-progs lvm2 grub efibootmgr
    ```
  - 生成fstab
    ```bash
    genfstab -U /mnt > /mnt/etc/fstab
    ```

    这个fstab用的是uuid,我不放心,更改了一下
    ```
    /etc/fstab
    ------------------------------
    /dev/mapper/Arch-arch   /               btrfs           rw,relatime,space_cache=v2,subvolid=257,subvol=/@       0 0
    /dev/mapper/Arch-arch   /boot           btrfs           rw,relatime,space_cache=v2,subvolid=256,subvol=/@boot   0 0
    /dev/sda2               /boot/efi       vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro   0 2
    /dev/mapper/Arch-arch   /home           btrfs           rw,relatime,space_cache=v2,subvolid=258,subvol=/@home   0 0
    /dev/mapper/Arch-arch   /.snapshots     btrfs           rw,relatime,space_cache=v2,subvolid=259,subvol=/@snapshots     0 0
    ```
  - 时区设置
    ```bash
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    hwclock --systohc --localtime
    ```
  - 本地化设置
    ```bash
    locale-gen
    ```

    移除  `/etc/locale.gen` 对应行前面的注释符号（＃）即可，建议选择带 UTF-8 的项​
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
  - 网络配置
    ```bash
    echo archlinux > /etc/hostname
    systemctl enable NetworkManager
    ```
  - Initramfs配置
    lvm配置
    ```bash
    /etc/mkinitcpio.conf
    ------------------------
    HOOKS=(base systemd ... block lvm2 filesystems ... )
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
  - 设置root密码
    ```bash
    passwd
    ```
  - 安装微码更新
    如果你使用的是 `intel`或 `AMD`的cpu，安装 `microcode`
    ```bash
    pacman -S intel-ucode amd-ucode
    ```
  - 安装grub
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
    grub-install --target=i386-pc /dev/sda
    grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Arch --removable
    ```

    然后生成  `grub.cfg`  文件
    ```bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
  - 卸载文件系统
    退出 `chroot`环境，输入 `exit`或者按 `Ctrl+d`
    ```bash
    umount -R /mnt
    ```
  - 关机,移除安装介质,重启
- lvm+btrfs方案的arch安装完成

  - 同步时间
    ```bash
    timedatectl set-ntp 1
    timedatectl set-timezone Asia/Shanghai
    timedatectl set-local-rtc 1
    pacman -S ntp
    systemctl enable ntpd --now
    ```
  - 安装archzfs仓库的密钥
    ```bash
    curl -L https://archzfs.com/archzfs.gpg |  pacman-key -a -
    pacman-key --lsign-key $(curl -L https://ay1.us/https://git.io/JsfVS)
    curl -L https://ay1.us/https://git.io/Jsfw2 > /etc/pacman.d/mirrorlist-archzfs
    ```
  - 添加archzfs仓库
    将以下内容添加到`/etc/pacman.conf`的末尾
    ```bash
    [archzfs]
    # Server = http://archzfs.com/$repo/$arch
    Server = https://ay1.us/http://mirror.sum7.eu/archlinux/archzfs/$repo/$arch
    # Server = https://mirror.biocrafting.net/archlinux/archzfs/$repo/$arch
    # Server = https://mirror.in.themindsmaze.com/archzfs/$repo/$arch
    # Server = https://zxcvfdsa.com/archzfs/$repo/$arch
    ```
  - 安装zfs,激活zfs模块
    - 给iso安装zfs模块与zfs工具还有linux-zen-headers
      ```bash
      pacman -S linux-zen-headers archzfs-dkms zfs-utils 
      ```
    - 启用zfs模块
      ```bash
      /sbin/modprobe zfs
      ```
  - 创建zfs池,创建数据集
    创建zfs池(有压缩的特性)
    ```bash
    zpool create -O compression=zstd -O mountpoint=none homepool /dev/mapper/Arch-homepool -f
    ```

    创建并挂载zfs数据集(有去重的特性)
    ```bash
    zfs create -o dedup=on -o mountpoint=/home homepool/home
    ```
  - 把@home子卷删除,同时在fstab中删除它的挂载
    @home子卷删除
    ```bash
    btrfs subvolume delete @home
    ```

    在fstab中删除它的挂载
    ```bash
    /etc/fstab
    ------------------------------
    /dev/mapper/Arch-arch   /               btrfs           rw,relatime,space_cache=v2,subvolid=257,subvol=/@       0 0
    /dev/mapper/Arch-arch   /boot           btrfs           rw,relatime,space_cache=v2,subvolid=256,subvol=/@boot   0 0
    /dev/sda2               /boot/efi       vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro   0 2
    /dev/mapper/Arch-arch   /.snapshots     btrfs           rw,relatime,space_cache=v2,subvolid=259,subvol=/@snapshots      0 0
    ```
  - 设置zfs的Initramfs
    修改Initramfs
    ```bash
    /etc/mkinitcpio.conf
    ------------------------
    HOOKS=(base systemd ... block lvm2 zfs filesystems ... )
    ```

    刷新
    ```bash
    mkinitcpio -p linux-zen
    ```
  - 开启zfs的自动导入与挂载服务
    ```bash
    systemctl enable zfs-import-cache zfs-import.target zfs-mount zfs-zed zfs.target
    ```
  - 限制zfs的ARC
    ```bash
    echo "options zfs zfs_arc_max=2147483648" >> /etc/modprobe.d/zfs.conf
    ```
