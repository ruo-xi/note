# Archlinux Basic 

## installation

### make Startup Disk

download the iso file and [GnuPG](https://wiki.archlinux.org/index.php/GnuPG) signature file

[download page](https://www.archlinux.org/download/)

* windows   use **rufus** to make Startup Disk

```
gpg --keyserver-options auto-key-retrieve --verify archlinux-version-x86_64.iso.sig
```



* linux     use **Ventoy**  to make Startup Disk

```
pacman-key -v archlinux-version-x86_64.iso.sig
```

### install the system with startup disk

```
# List the partition tables for the specified devices and then exit.
fdisk -l 

# what you need is a swap partition system partition efi partiton
# sdb is the disk which you will opreate
fdisk /dev/sdb     
# if it's a new disk you need to type g to create a new gpt partion table
# p print the disk info
# n add a new partition 
# q quit
# w save your opreation
# m for more help

# eg: 
# efi partiton:/dev/sda1
# swap partition:/dev/sda2 
# system partition:/dev/sda3 

# make file system
mkswap /dev/sdb2
mkfs.vfs /dev/sdb3

# wired network dont'd need to do anything to connect the internet
# wireless network need wifi_menu to connect
wifi-menu

ping baidu.com

# mount partition
mount /dev/sda3 /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
swapon /dev/sda2
 
# instal base package
pacstrap /mnt base base-devel linux linux-firmware

#config the fstab
genfstab -L /mnt >> /mnt/etc/fstab

# chroot
arch-chroot /mnt

# set locale
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# hwclock --systohc              
timedatectl set-local-rtc true # 是archlinux 使用CST时间 避免因为使用UTC与windows 冲突

# instal package needed eg:network  boot  text editor
sudo pacman -S networkmanager grub efibootmgr os-prober ntfs-3g vim

# config lang
vim /etc/locale.gen
locale-gen
vim /etc/locale.conf  #文件不存在 保存时会自动创建  文首添加 LANG=en_US.UTF-8

# config hostname
vim /etc/hostname  # 文首添加 你的主机名 如 ${hostname}  记得替换
vim /etc/hosts # 文尾添加 
							#127.0.0.1 localhost
              #::1   localhost
              #127.0.1.1 ${hostname}.hostdomain ${hostname} 
passwd  #设置root密码

#Grub
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg

# add user
useradd -m -G wheel username 
passwd username  # set passwd for username
visudo  
# find line with # %wheel ALL=(ALL)ALL 
# remove the # symbol
# if error run ln -s /bin/vim /bin/vi  for visudo need the vi command


reboot

# config network
sudo systemctl enable --now NetworkManager
sudo pacman -S network-manager-applet # 网络设置图标

# install video card driver
sudo pacman -S nvidia  #nvidia only on card

# windows enviroment
sudo pacman -S plasma kde-applications   # kde
sudo pacman -S xfce4 xfce4-goodies       # xface

# display manager
sudo pacman -S sddm
sudo ststemctl enable --now sddm   #之后就会进入图形界面

```