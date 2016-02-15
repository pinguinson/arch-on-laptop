# Arch Linux on laptop
Installation guide for Arch Linux on a modern laptop (EFI). Basic installation part should work with almost any laptop (you may encounter problems with wifi though). Advanced part is mostly optimized for ASUS UX32L.

## Basic installation

#### Connect to the Internet and set correct time

First of all we need to set up connection to the Internet and set the right time:
```
# wifi-menu
# timedatectl set-ntp true
```

#### Partitioning and formatting disk
This example shows how to create two partitions using 'parted': one 512Mb for EFI and one other using 100% of remaining space:
```
# parted /dev/sdx
(parted) mkpart ESP fat32 1MiB 513MiB
(parted) set 1 boot on
(parted) mkpart primary ext4 513MiB 100%
```
Once the partitions have been created, each must be formatted with an appropriate file system:
```
# mkfs.ext4 /dev/sdxY
# mkfs.fat -F32 /dev/sdxY
```

Now mount partitions:
```
# mount /dev/sdxY /mnt
# mkdir /mnt/boot
# mount /dev/sdxY /mnt/boot
```

#### Install the base packages
Note: you may want to set `SigLevel = Never` in `/etc/pacman.conf` to avoid key issues while installing base packages. Clearly it's not the best solution but I haven't found a better one yet:
```
# pacstrap -i /mnt base base-devel
```

#### Generate fstab
Generate an fstab file. The `-U` option indicates UUIDs, labels can be used instead through the `-L` option:
```
# genfstab -U /mnt >> /mnt/etc/fstab
```

#### Change root
Chroot to the new system in `/mnt`:
```
# arch-chroot /mnt /bin/bash
```

#### Setting locales
Uncomment needed localisations (`en_US.UTF-8 UTF-8`) in `/etc/locale.gen`, generate the new locales:
```
# locale-gen
```
Create new `/etc/locale.conf` file:
```
# echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

#### Time
Create the symbolic link `/etc/localtime`:
```
# ln -s /usr/share/zoneinfo/Europe/Moscow /etc/localtime
```
It is recommended to adjust the time skew, and set the time standard to UTC:
```
# hwclock --systohc --utc
```

#### Initramfs
Generate the initramfs image:
```
# mkinitcpio -p linux
```

#### Install a boot loader
First, install *intel-ucode* package:
```
# pacman -S intel-ucode
```
Install systemd-boot to the EFI system partition:
```
# bootctl install
```
When successful, create a boot entry in `/boot/loader/entries/arch.conf`:
```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=PARTUUID=XXXX rw
```
Where XXXX is PARTUUID of your ext4 partition. You can get one with this command:
```
# ls -l /dev/disk/by-partuuid/ | grep sdxY | cut -d ' ' -f9
```

#### Hostname
Set the hostname to your liking:
```
# echo "deepthought" > /etc/hostname
```

#### Wireless
Install *iw*, *wpa_supplicant*, and (for *wifi-menu*) *dialog*:
```
# pacman -S iw wpa_supplicant dialog
```

#### Root
Set root password with:
```
# passwd
```

#### User
Create user profile:
```
# useradd -m -g users -G wheel -s /bin/bash myusername
# passwd myusername
```

#### Sudo without password
So you don't have to enter password every fucking time:
```
# echo "%wheel ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/10-grant-nopasswd-sudo
```
Here the basic installation comes to an end.

## Advanced installation

#### Configuring wifi

You don't want to use wifi-menu every time, right? Let's automate this process. First, install *wpa_actiond* package:
```
# pacman -S wpa_actiond
```

Then start and enable the service:
```
# systemctl start netctl-auto@interface.service
# systemctl enable netctl-auto@interface.service
```

#### Install yaourt

You will need this to install packages from AUR. First, install git:
```
# pacman -S git
```

Then install *package-query* and *yaourt* packages:
```
$ git clone https://aur.archlinux.org/package-query.git
$ cd package-query
$ makepkg -si
$ cd ..
$ git clone https://aur.archlinux.org/yaourt.git
$ cd yaourt
$ makepkg -si
$ cd ..
$ rm -rf package-query/ yaourt/
```

#### Graphics drivers

Download nvidia drivers:
```
# pacman -S nvidia
```
Install bumblebee:
```
# pacman -S bumblebee
```
In order to use Bumblebee, it is necessary to add your regular user to the bumblebee group:
```
# gpasswd -a username bumblebee
```
Enable *bumblebeed* service and reboot:
```
# systemctl enable bumblebeed.service
# reboot
```

#### Xorg

First, install *xorg-server* and *xorg-xinit*:
```
# pacman -S xorg-server xorg-xinit
```

#### Dotfiles
Install dotfiles from `pinguinson/dotfiles-mba`, then install few packages:
```
$ yaourt -S bspwm sxhkd compton-git lemonbar-xft-git ttf-ubuntu-font-family xorg-xrdb xorg-xsetroot rxvt-unicode-patched zsh prezto-git
```

Then switch to zsh:
```
$ chsh -s /bin/zsh
```

#### Fonts
Install infinality. First, add two repositories to `/etc/pacman.conf`:
```
[infinality-bundle]
Server = http://bohoomil.com/repo/$arch
SigLevel = Never

[infinality-bundle-fonts]
Server = http://bohoomil.com/repo/fonts
SigLevel = Never
```

Then install meta-packages:
```
# pacman -S ibfonts-meta-base infinality-bundle
```

Install few fonts:
```
$ yaourt -S ttf-monaco ttf-ubuntu-font-family
```

#### Appearance
Install few packages:
```
$ yaourt -S lxappearance gtk-theme-arc-git vertex-icons-git xcursor-human
```
Then set themes in LXAppearance app.

#### Touchpad

Install Synaptics driver:
```
# pacman -S xf86-input-synaptics
```

Create a file with synaptics config:
```
# sudo touch /etc/X11/xorg.conf.d/synaptics.conf
```

This basic configuration is quite enough for start:
```
Section "InputClass"
        Identifier "touchpad catchall"
        Driver "synaptics"
        MatchIsTouchpad "on"
        MatchDevicePath "/dev/input/event*"
        Option "TapButton1" "1"
        Option "TapButton2" "2"
        Option "EmulateTwoFingerMinZ" "35"
        Option "EmulateTwoFingerMinW" "8"
        Option "TapButton3" "3"
        Option "VertTwoFingerScroll" "1"
        Option "HorizTwoFingerScroll" "1"
        Option "VertEdgeScroll" "1"
        Option "HorizEdgeScroll" "1"
        Option "VertScrollDelta" "100"
        Option "HorizScrollDelta" "100"
        Option "CoastingSpeed" "8"
        Option "CornerCoasting" "1"
        Option "CircularScrolling" "1"
        Option "CircScrollTrigger" "7"
EndSection
```

But if you want to have more options, you can find the full description in manual:
```
# man 4 synaptics
```
