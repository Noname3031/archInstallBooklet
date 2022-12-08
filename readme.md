# 安装Arch
使用btrfs和加密的LVM
在阅读本教程前,请确认电脑架构并且酌情使用相关配置.
这是根据网上四篇教程总结出的,相关链接:
[其一](https://www.viseator.com/2017/05/17/arch_install/) [其二](https://www.howtoing.com/how-to-install-arch-linux-with-full-disk-encryption/) [其三](https://blog.yangmame.org/在U盘安装加密的Linux系统.html) [其四](https://snowfrs.com/2019/08/10/intall-archlinux-with-btrfs.html)

注意,全程假设目标硬盘是`/dev/sda`

## 前期准备
请到[ArchLinux](https://www.archlinux.org/download)官网下载光盘镜像,并写入到U盘/光盘上,并从UEFI启动.
联网:`dhcpcd`或着`iwctl`

后者请按照流程输入WiFi配置信息

时间更新:`timedatectl set-ntp true`

更新包管理器的配置:`vim /etc/pacman.conf`

取消注释Misc options里的paralleldownloads = 5来启用多线程下载,

测试镜像服务器,并更新列表:`reflector --save /etc/pacman.d/mirrorlist --protocol https --sort rate -a 6 -c china`

扫描并更新源:`pacman -Syy`

## 准备磁盘
查看磁盘:`lsblk`
硬盘一般来说是sd*,或者nvme*,前者sata或usb接口,后者nvme接口.

**注意,磁盘将被擦除,请在此之前做好数据备份.**
擦除磁盘:`shred --verbose --random-source=/dev/urandom --iterations=1 /dev/sda`

这里iterations=1指的是擦除1遍.时间可能会很长.

## 分区
概览:

|分区名称|逻辑卷名称|文件系统|安装时的挂载点| 建议大小|
|---------|---------|---------|---------|---------|
|/dev/sda1|/boot|vfat(FAT32)|/mnt/boot|256MB或以上|
|/dev/sda2|system|物理卷|不适用|剩余空间的大小,如果不需要额外的分区|
| |system-root|btrfs|/mnt|另外两个分区余下的大小|
| |system-swap|swap|无|RAM大小或更大|

若需要挂起到硬盘则swap大小至少大于或等于ram大小,以防休眠到硬盘时出错.
如不需要,设置为ram的1/4即可.

|分区名称|例子(SSD,可用476.94GiB)|
|---------|---------|
|/dev/sda1|512MB(已用96.16MiB)|
|/dev/sda2|476.44GiB|
|system-root|444.4GiB(已用15.7GiB)|
|system-swap|32GiB|

开始分区:`cfdisk /dev/sda`

然后选择gpt分区,创建上表
后期可通过mkswap [文件路径]和swapon [文件路径]指令创建和激活swap文件.
分区后:(虚拟机)
运行lsblk后即可看到下图(部分省略)

|NAME|SIZE|TYPE|
|----|----|----|
|sda|476.9G|disk|
|┣sda1|512M|part|
|┗sda2|476.4G|part|



## 配置分区

新建加密分区:`cryptsetup -c aes-xts-plain64 -s 512 -h sha512 -i 8192 --use-urandom luksFormat --type=luks2 /dev/sda2`

打开lvm卷:`cryptsetup open --type luks /dev/sda2 cryptlvm`

创建物理卷:`pvcreate /dev/mapper/cryptlvm`

创建System组:`vgcreate system /dev/mapper/cryptlvm`

创建根分区:`lvcreate -L [分区大小] system -n root`

创建swap分区:`lvcreate -L [分区大小] system -n swap`

## 格式化/挂载
|子卷挂载列表,酌情创建:|                       |
| --------------------- | --------------------- |
| 子卷名[顺序:自上而下] | 位置                  |
| @                     | /mnt                  |
| @home                 | /mnt/home             |
| @logs                 | /mnt/var/log          |
| @tmp                  | /mnt/tmp              |
| @docker               | /mnt/var/lib/docker   |
| @pkgs                 | /mnt/var/cache/pacman |
| @build                | /mnt/var/lib/build    |
| @snapshots            | /mnt/.snapshots       |

|硬盘优化选项(多选)||
| ---- | ---- |
| 必选 | noatime,nodiratime              |
| ssd  | compress=zstd,ssd,discard=async |
| hhd  | compress-force=zstd,autodefrag  |

格式化boot分区:`mkfs -t vfat -F 32 /dev/sda1`

格式化root分区:`mkfs -t btrfs /dev/mapper/system-root`

格式化swap分区:`mkswap /dev/mapper/system-swap`

开启swap分区:`swapon /dev/mapper/system-swap`

挂载btrfs根分区并创建子分区:`mount -o [优化选项] /dev/mapper/system-root /mnt`

创建子分区:`btrfs subvolume create /mnt/[子卷名]`

卸载:`umount /mnt`

挂载btrfs根分区:`mount -o [优化选项],subvol=@ /dev/mapper/system-root /mnt`

创建子卷目录:`mkdir -p /mnt/{btrfs-root,boot,home,var/{log,lib/{docker,build},cache/pacman},tmp}`

挂载子卷:`mount -o [优化选项],subvol=[子卷名] /dev/mapper/system-root [位置]`

挂载root:`mount -o [优化选项],subvol=/ /dev/mapper/system-root /mnt/btrfs-root`

挂载boot分区:`mount /dev/sda1 /mnt/boot`

按需创建并挂载后:lsblk(部分省略)

| NAME         | SIZE   | TYPE  | MOUNTOPTIONS    |
| ------------ | ------ | ----- | --------------- |
| sda          | 476.9G | disk  |                 |
| ┣sda1        | 512M   | part  | /mnt/boot       |
| ┗sda2        | 476.4G | part  |                 |
| ┗cryptlvm    | 476.4G | crypt |                 |
| ┣system-root | 444.4G | lvm   | /mnt/btrfs-root |
| ┃            |        |       | /mnt/home       |
| ┃            |        |       | /mnt            |
| ┗system-swap | 32G    | lvm   | [SWAP]          |

## 安装
安装基本系统:`pacstrap -i /mnt base base-devel  linux linux-firmware vim lvm2 snapper dhcpcd grub grub-btrfs efibootmgr wpa_supplicant`

可选在同时安装kde桌面:
在上述命令后加上:`plasma kde-applications`

可选在同时安装clamav防毒软件本体和防毒软件前端:
在上述命令后加上:`clamav clamtk`

如果安装了kde桌面请务必执行:(注意大小写)`systemctl enable NetworkManager sddm`

创建fstab文件,用于开机时挂载分区:`genfstab -U -p /mnt >> /mnt/etc/fstab`

## 配置
改变根目录:`arch-chroot /mnt`

配置地区设置:`vim /etc/locale.gen`
取消注释en_US.UTF-8和zh_CN.UTF-8 退出

应用地区设置:`locale-gen`

设置语言:`echo LANG=en_US.UTF-8 > /etc/locale.conf`

设置时区:`ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime`

设置root密码:`passwd root`

设置主机名:`echo [想设置的主机名] > /etc/hostname`

添加用户:`useradd -m -g users -G wheel,games,power,optical,storage,scanner,lp,audio,video -s /bin/bash [用户名称]`

更改新用户密码:`passwd [用户名称]`

让wheel组用户可以使用sudo:`EDITOR=vim visudo`
取消注释 %whell all=(all) all

记下加密物理卷的UUID:`blkid`
是带有'TYPE="crypto_LUKS"'的分区,请不要和其它分区的UUID混淆.

配置grub:`vim /etc/default/grub`
并在GRUB_CMDLINE_LINUX=""的双引号间加入:`cryptdevice=UUID=[是刚刚记下的UUID]:cryptlvm root=/dev/system/root resume=/dev/system/swap`

配置内核:`vim /etc/mkinitcpio.conf`
在MODULES=(...)里加入**btrfs**,
在HOOKS=(...)加入 **keyboard,keymap,encrypt,lvm2,resume**和替换**fsck**为**btrfs**(注意顺序)
例子:
`HOOKS=(base udev autodetect keyboard keymap modconf kms consolefont block encrypt lvm2 resume filesystems btrfs)`

如果是笔记本等设备 在/etc/systemd/logind.conf里添加"HandleLidSwitch=hibernate".

制作内核:`mkinitcpio -p linux`

安装引导:`grub-install --efi-directory=/boot --boot-directory=/boot --recheck --removable`

导出启动配置:`grub-mkconfig -o /boot/grub/grub.cfg`

退出chroot:`exit`

重启:`reboot

重启后 设置硬件时钟:`hwclock --systohc --utc`

重启后 设置使用网络时间:`timedatectl set-ntp true`

### 至此 你的电脑应该能正常使用了,欢迎使用Arch.
重启后,如果在在上面步骤安装了clamav,请务必执行:
```
systemctl enable clamav-daemon 
systemctl start clamav-daemon
```
如果安装了kde桌面请务必执行:(注意大小写)`pacman -S archlinux-appstream-data packagekit-qt5 flatpak fwupd`

## 附录
### 系统相关
▷显卡驱动
intel:`sudo pacman -S xf86-video-intel`

nvidia:`sudo pacman -S xf86-video-nouveau`

ATI:`sudo pacman -S xf86-video-amdgpu`

▷处理器微码(Microcode) 
AMD:`pacman -S amd-ucode`

Intel:`pacman -S intel-ucode`

▷快照
根据自己的subvolume实际情况创建snapshot策略

创建配置文件:`snapper -c [名字] create-config [子卷的目录]`

编辑配置文件:`vim /etc/snapper/configs/[已经创建的名字]`

列出配置文件:`snapper list-configs`

撤回:`snapper -c [名字] undochange [起始快照编号]..[目标快照编号]`

启用自动创建/清理功能:
```
systemctl enable --now snapper-timeline.timer
systemctl enable --now snapper-cleanup.timer
```

### 软件相关
▷AUR(用户软件库)
```
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```
▷fcitx输入法和字体
`pacman -S noto-fonts-cjk fcitx fcitx-im kcm-fcitx`

kcm-fcitx是图形化配置程序.
在/etc/profile文件,在文件开头加入三行:
```
export XMODIFIERS="@im=fcitx"
export GTK_IM_MODULE="fcitx"
export QT_IM_MODULE="fcitx"
```
▷vscode:`yay -S code`

▷v2ray(梯子) :`yay -S v2raya`

▷Yakuake(下拉式终端):`pacman -S yakuake`

▷Zsh(Shell):`yay -S zsh`

`chsh -s /bin/zsh`或`vim /etc/`passwd编辑默认shell.
可选oh-my-zsh-git:`yay -S oh-my-zsh-git`

▷VirtualBox 虚拟机:`pacman -S virtualbox virtualbox-ext-vnc virtualbox-guest-iso virtualbox-host-modules-arch linux-headers`

VirtualBox 扩展安装后:`sudo usermod -a -G vboxusers [用户名]`
