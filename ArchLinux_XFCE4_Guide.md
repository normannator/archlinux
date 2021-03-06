﻿

# Arch Linux XFCE4 Installation GUIDE

In this Installation Guide, I will explain my default XFCE4 Setup for ArchLinux with LUKS Drive-Encryption and Kali-Linux Similar Look and Feel of XFCE4. (EFI Style)

## Run the Arch Linux Disc
Burn and Run the ArchLinux ISO until the command prompt appears.

## Set up Base
```bash
loadkeys <language>  # e.g. de or de-latin1
```

### Internet Connection
```bash
ip a # check internet connection (alternative: ip link)
```

#### On Wifi
```bash
wpa_passphrase  SSID  Passwort  > /etc/wpa_supplicant/wpa_supplicant.conf
wpa_supplicant -i wlp0s1 -D wext -c /etc/wpa_supplicant/wpa_supplicant.conf -B
dhcpcd wlp0s1
```

#### On LAN
```bash
dhcpcd <ethernet-adapter>
```

### Update Database
```bash
pacman -Syyy
```

### (extra) If you want a gui install / config use archfi
If you are using this, you can skip until the following steps to *"Set up XFCE"*
```bash
pacman -S wget pacman-contrib
wget archfi.sf.net/archfi
sh archfi
```

### PreConfig
```bash
export EDITOR=nano <or vim>
```

### Configure Partitions
I use *gdisk* for GPT Tables
```bash
lsblk  # Print Devices
gdisk /dev/<os-device>

n, 128, -200M, Enter, ef00  # Create EFI Partition
n, Enter, Enter, Enter, 8e00  # Create Linux LVM Partition
w

# If you are using a separate home drive, use this
gdisk /dev/<home-device>
n, Enter, Enter, Enter, Enter  # Create Linux Filesystem Partition for the entire drive
```
#### Encrypt Root Partition
```bash
cryptsetup luksFormat /dev/sda1  # Root Partition Device
Confirm and Enter Passphrase.

cryptsetup open /dev/sda1 cryptlvm  # Open Created LVM Device
Enter Passphrase

pvcreate /dev/mapper/cryptlvm  # create Physical Volume

vgcreate vg1 /dev/mapper/cryptlvm  # create Volume Group
```

#### Create Logical Volumes
```bash
lvcreate -L <size-of-ram>G vg1 -n swap
lvcreate -l 100%FREE vg1 -n root
```
Check your work with *lsblk*

#### Format Logical Volumes and Partitions
```bash
mkfs.fat -F32 /dev/sda128
mkfs.ext4 /dev/vg1/root
mkswap /dev/vg1/swap

mkfs.ext4 /dev/<home-partition>
```

##### Mount Partitions
```bash
mount /dev/vg1/root /mnt
mkdir /mnt/home
mount /dev/<home-partition> /mnt/home
mkdir /mnt/boot
mount /dev/sda128 /mnt/boot
swapon /dev/vg1/swap
```

### Create Local Mirror List
```bash 
reflector --verbose --country 'Germany' -l 200 -p https --sort rate --save /etc/pacman.d/mirrorlist

# Alternative
nano /etc/pacman.d/mirrorlist
```

### Install Base Packages
```bash
pacstrap /mnt base base-devel linux<-lts> linux-firmware nano <vim> dhcpcd lvm2 reflector   # <> ist optional

### you can add these packages, to the command above if you like:
# File Systems
dosfstools  # DOS filesystem utilities
btrfs-progs  # Btrfs filesystem utilities
xfsprogs  # XFS filesystem utilities
f2fs-tools  # Tools for Flash-Friendly File System (F2FS)
jfsutils  #  JFS filesystem utilities
reiserfsprogs  # Reiserfs utilities
dmraid  # Device mapper RAID interface

```
[f2fs-tools](https://archlinux.org/packages/extra/x86_64/f2fs-tools/ "View package details for f2fs-tools")

### Config Arch Linux

#### Pre Config
```bash
# Set Computer Hostname
echo myhost > /etc/hostname 
 
# Set Keyboard Layout
echo KEYMAP=de-latin1 > /etc/vconsole.conf  

# Set Font (optional)
echo FONT=lat9w-16 >> /etc/vconsole.conf  

# Set Locale
echo LANG=de_DE.UTF-8 > /etc/locale.conf  # Set Language
nano /etc/locale.gen
-> uncomment this:
#de_DE.UTF-8 UTF-8
#de_DE ISO-8859-1
#de_DE@euro ISO-8859-15
#en_US.UTF-8
locale-gen

# Set Time
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime  # Set Timezone
hwclock --sysohc --utc # Sync Hardware-Clock

# Set Root Password
arch-chroot /mnt /root
passwd root
exit

# Gererate fstab
genfstab -U /mnt >> /mnt/etc/fstab  # (alt. change -U to -L to use Label instead of UUID
cat /mnt/etc/fstab  # check gen

# Edit Fstab (optional)
nano /etc/fstab

# Edit Crypttab (optional) -> Not used here
nano /mnt/etc/crypttab  # Config encrypted block devices
```

#### !!!Change to Root!!!
```bash
arch-chroot /mnt
```

#### Edit Hosts file
```bash
cat /etc/hosts  # set Hosts file
-->
#<ip-address>	<hostname.domain.org>	<hostname>
127.0.0.1		localhost.localdomain	localhost
::1				localhost.localdomain	localhost
127.0.0.1		<hostname>.localdomain	<hostname>
```

#### Edit Mirror List
```bash 
reflector --verbose --country 'Germany' -l 200 -p https --sort rate --save /etc/pacman.d/mirrorlist

# Alternative
nano /etc/pacman.d/mirrorlist
```

#### Edit mkinitcpio.conf and generate
```bash
nano /etc/mkinitcpio.conf
# Add:
# ... autodetect keyboard keymap modconf block encrypt lvm2 filesystems ...

# Generate mkinitcpio
mkinitcpio -p linux
```

#### Bootloader
(I used grub, you can use *syslinux* instead, if you want)
```bash
pacman -S grub efibootmgr 
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

blkid  # BlockID
-> Copy or note down UUID of root partition (encrypted)
nano /etc/default/grub
--> Edit: 
...
GRUB_CMDLINE_LINUX="cryptdevice=UUID="<UUID-copied>:cryptlvm root=/dev/vg1/root"

grub-mkconfig -o /boot/grub/grub.cfg  # Generate Grub config file
```

## Set up Arch Linux Root

### Updates
```bash
# Install Contributed scripts and tools for pacman systems
pacman -S pacman-contrib

# Install YAY - simple AUR Package Installation
git clone https://aur.archlinux.org/yay.git
cd yay/
makepkg -si PKGBUILD
# Alternativ Package Installer: trizen, aurman, yaourt (End of life)

# Update System
pacman -Syu

# Clean orphan
pacman -Rns $(pacman -Qqtd)

# Clean Cache
pacman -Sc

# Edit pacman.conf
nano /etc/pacman.conf
# Uncomment Color
# Add ILoveCandy
# uncomment multilib
pacman -Sy

### Optional - Segnature Security ###
pacman -S archlinux-keyring  # Update keyring
pacman-key --refresh-keys  # Refresh pacman keys
# evtl. edit pacman.conf -> SigLevel
gpg --recv-keys  # Add GPG key (optional-additional)
```

### Install

#### Console
```bash
### Generic
pacman -S usbutils dialog powertop
# optional #
# lsof  # Lists open files for running Unix processes
# dmidecode  # Hardware Infos
# nmon  # Systen Monitor
# mc  # Dual Pane File Explorer
# neofetch  # system information tool
# fwupd  # Firmware upgrade
# gpm  # Console Mouse Support
# liveroot  # (AUR) root overlay fs

### Compression Tools
pacman -S zip unzip unrar p7zip lzop

### Network Tools
pacman -S rsync traceroute bind-tools
# optional #
# dnsutils  # DNS tools (nslookup)
# nmap  # Network Scanner
# netdiscover  # (AUR) Network Scanner with MAC vendors
# speedtest-cli  # SpeedTest
# arp-scan  # ARP SCANNER
# wavemon  # WIFI monitor
# net-tools  # (deprecated) old ifconfig
# dsniff  # tools for network auditing and penetration
# mitmproxy  # SSL-capable MITM HTTP proxy
# sslstrip  # tool to hijack HTTPS in MITM attack

### Web Browser
# optional: elinks, links, lynx

### Recovery Tools
# optional # 
# ddrescue  # HD recovery tool
# dd_rescue  # HD recovery tool
# partclone  # Copy used block on partition
```

#### System
```bash
### Services
pacman -S networkmanager openssh xdg-user-dirs intel/amd-ucode (bluez bluez-utils) pkgstats 
# optional #
# cronie  # Cron tasks server
# numlockon  # Numlock on on tty
# net-snmp  # SNMP Server
# samba  # Server SMB
# syslog-ng
# ntp
# haveged

### Filesystem
pacman -S os-prober dosfstools ntfs-3g gvfs 
# optional #
# snapper  # snapshot manager (ext4, lvm, btrfs)
# btrfs-progs  # BTRFS ffile utils
# exfat-utils  # EXFAT file support
# gptfdisk
# autofs 
# fuse2 
# fuse3
# fuseiso 
# sshfs  # ssh client
# cifs-utils  # SMB mound command
# smbclient  # SMB full client
# nfs-utils
# open-iscsi
# glusterfs
# hfsprogs (AUR)
# mtpfs
# unionfs-fuse
# nilfs-utils
# s3fs-fuse

### Sound
pacman -S alsa-plugin alsa-utils pulseaudio pulseaudio-alsa (pulseaudio-bluetooth)
# optional: pulseaudio-equalizer

### Printer
sudo pacman -S cups ghostscript cups-pdf hplip
# optional #
# gutenprint  # Top quality printer drivers for POSIX systems
# foomatic-db foomatic-db-*  # Foomatic - The collected knowledge about printers, drivers, and driver options in XML files, used by foomatic-db-engine to generate PPD files.
```

#### XOrg
```bash
### Print GPU Info
lspci -k | grep -A 2 -E "(VGA|3D)"

### Install
pacman -S --needed xorg

### Fonts - Default
pacman -S ttf-bitstream-vera ttf-dejavu ttf-liberation adobe-source-sans-pro-fonts
# optional #
# font-bh-ttf
# gsfonts 
# sdl_ttf
# xorg-fonts-type1 

### Fonts - Misc
pacman -S ttf-anonymous-pro ttf-droid ttf-gentium  ttf-ubuntu-font-family ttf-ms-fonts ttf-roboto ttf-roboto-mono ttf-font-awesome
# for my purpose
yay -S ttf-hackgen
pacman -S ttf-fira-code
# For More Fonts: pacman -Ss ttf

### input drivers
pacman -S xf86-input-synaptics (optional: Laptop Touchpad-Driver)
# optional #
# xf86-input-libinput  #  Generic input driver for the X.Org server based on libinput
# xf86-input-elographics
# xf86-input-evdev
# xf86-input-vmmouse  # (VMWare)
# xf86-input-void
# xf86-input-wacom
# other: Saitek-*, Madcatz-*

### video drivers

# Proprietary
sudo pacman -S nvidia
# optional #
# virtualbox-guest-utils  # (For Virtualbox)
# nvidia-390xx  # (AUR) End of life
# nvidia-dkms  # (For custom kernel)
# nvidia-390xx-dkms  # (AUR) (for custom kernel) -> for my older laptop
# optional: nvidia-settings

# Opensource
# xf86-video-*
# e.g. nouveau

# ! Only use one Proprietary or Opensource Video Driver !
```
also check out this [link](https://wiki.archlinux.org/index.php/NVIDIA)


#### Desktop Environment
Choose one of the following:
1. [Plasma5](https://wiki.archlinux.de/title/Plasma) (*Components: partitionmanager gnome-keyring xsettingsd kdeconnect sshfs systemdgenie*)  # QT
2. XFCE4  # GTK
3. Gnome  # GTK
4. Cinnamon
5. LXDE
6. LXDE-GTK3
7. LXQt
8. Mate
9. Enlightenment
10. Openbox
11. Deepin

In my case I choosed: **XFCE4**
```bash
sudo pacman -S --needed xfce4 xfce4-goodies lightdm lightdm-gtk-greeter lightdm-gkt-greeter-settings (gvfs-afc) udisks2 network-manager-applet

sudo systemctl enable lightdm
```

*TODO - Package installation guide for the Desktop Environments above* (mit archdi - hint)

#### Display Manager
I choosed lightdm-gtk-greeter
Alternatives:
* gdm
* sddm
* lxdm

#### Applications

##### Office
```bash
### Suits
pacman -S libreoffice-still  # stable version (newer features in "fresh")
# alt. (aur): freeoffice/openoffice

### office language aids
pacman -S hunspell-en hunspell-de mythes-en mythes-de aspell-en aspell-de languagetool enchant 
yay -S libreoffice-extension-languagetool 
sudo libmythes 
# optional #
# hyphen  # Hyphenation rules

### Tools
# galculator  # GTK+ based scientific calculator
# qalculate-gtk  #  GTK frontend for libqalculate
```

##### Internet
```bash
### Web Browser
pacman -S chromium firefox midori vivaldi vivaldi-ffmpeg-codecs  # midori is a Light Browser
# optional #
# pepper-flash, firefox-i18n, flashplugin, freshplaerplugin (aur), opera, opera-ffmpeg-codecs, seamonkey, seamonkey-i18n (aur), falkon

### Torrent
pacman -S qbittorrent
# optional #
# transmission-gtk/qt
# ktorrent
# deluge
# tixati

### E-Mail
pacman -S thunderbird
# optional #
# evolution + evolution-*
```

##### Multimedia
```bash
### GStreamer - Streaming Videos
pacman -S gst-plugins-base gst-plugins-good gst-plugins-ugly gst-plugins-libav

### Audio Player
# Optional #
# clementine  # (QT) Audio Player
# mixx  # (QT) Audio Player
# quodlibet  # (GTK) Audio Player
# elisa  # (QT) Audio Player
# amarok  # (AUR) (QT) Audio Player
# guayadeque  # (AUR) (GTK) Audio Player
# gmusicbrowser  # (AUR) (GTK) Audio Player

### Video Player
pacman -S celluloid 
# optional #
# vlc  # (QT) Video Player
# smplayer  # (QT) Video Player
# smtube  # (QT) Youtube Player
# mpv  # (GTK) Recommended for smplayer

### Video Tools
# optional #
# avidemux-qt  # (QT) Video Editor
# simplescreenrecorder  # (QT) Screen Recorder

### Burner
# optional #
# k3b (QT)
# xfburn (GTK)
# brasero (GTK)
# gnomebaker (AUR)
```

##### Graphic
```bash
pacman -S gimp
# optional #
# inkscape (GTK)
# dia (GTK)
# krita (QT)
```

##### Dev
```bash
pacman -S code  # vscode
# optional #
# notepadqq (QT)
# geany (GTK)
# geany-plugins (GTK)
# kdevelop (QT)
# gource (Git code animation)	
```

##### System
```bash
pacman -S gparted bleachbit virtualbox
yay -S virtualbox-ext-oracle  # (AUR) Ext Pack
# optional #
# keepassxc  # (QT) Password Manager
# keepass  # (MONO) Password Manager
# wine + wine_gecko + wine-mono # MS Windows App Support
# zenity + winetricks  # Recommended for Wine
# q4wine
# gdmap
# k4dirstat (aur)
# conky  # [Link to Tutorial Video](https://www.youtube.com/watch?v=QB8cjKpdVQY&ab_channel=ChrisTitusTech)

### Pacman GUI
# optional #
# octopo  # (AUR) (QT)
# pamac-aur  # (AUR) (GTK)
```

### Config

#### Bash
```bash
/etc/profile.d/editor.sh  # set default global editor
nano /etc/profile.d/alias.sh
```
![enter image description here](https://normannator.de/archlinux/IMG/Alias.PNG)
```bash
nano /etc/profile.d/ps1.sh  # (optional)
# Minimal				/home #
# User					user:/home #
# User and Hostname		user@hostname:/home #

# Now update .bashrc  # (optional)
```

#### Firewall (optional)
```bash
# edit IPv4			/etc/iptables/iptables.rules
# edit IPv6			/etc/iptables/ip6tables.rules

# load Rules
# iptables-restore & ip6tables-restore

# Start At Boot
# systemctl enable iptables & systemctl enable ip6tables

# Generate Default Rules
# /etc/iptables/iptables.rules & /etc/iptables/ip6tables.rules
```
**!!! For a simple Firewall !!!**
```bash
sudo pacman -S ufw
sudo ufw enable
sudo ufw status verbose
sudo systemctl enable ufw.service
```

#### Accounts
```bash
useradd -m -g users -s /bin/bash <name>
passwd <name>
EDITOR=nano visudo
--> Uncomment: # %wheel ALL=(ALL) ALL
gpasswd -a <user> wheel  # add user to wheel group
gpasswd -a <user> audio,video,games,power
```

#### Systemd

##### Services
```bash
Enable or Disable Services
systemctl enable NetworkManager
systemctl enable cups.service # previously installed
systemctl enable openssh.service
```
###### Set up Services
```bash
pacman -S acpid dbus avahi

systemctl enable acpid
systemctl enable avahi-daemon
```

##### Timedatectl  (optional)
The hardware clock can be queried and set with the _timedatectl_ command
```bash
# Enable  # timedatectl set-ntp true
# Edit  # /etc/systemd/timesyncd.conf
systemctl enable --now systemd-timesyncd.service  # Sync System Time with atomic clock
```

#### Boot (optional and redundant -> done above)
```bash
### initcpio
# edit /etc/mkinitcpio.conf
# mkinitcpio -p linux
```


### Install other optional Packages
```bash
pacman -S  wireless_tools wpa_supplicant  mtools  linux-<lts->headers git bash-completion jre8/11-openjdk 
xdg-utils  # Home Directories

pacman -S  gst-libav gst-plugins-good  # Multimedia graph framework
pacman -S icedtea-web  # Additional components for OpenJDK - Browser plug-in and Web Start implementation (optional)
```

### Exit and Reboot
```bash
exit
umount -a
reboot
```

## Config - The XFCE4 Desktop Environment

### Connect to Internet

#### With Wifi
```bash
ip a # search Wifi Adapter
nmtui
-> Activate Connection->...
```

#### With Ethernet
```bash
ip a # search for Ethernet adapter
dhcpcd <adapter>
```

### Use Reflector as above
```bash
reflector --verbose --country 'Germany' -l 200 -p https --sort rate --save /etc/pacman.d/mirrorlist
```

### Update System
```bash 
pacman -Syu
```

### (optional) Encrypt Home Partition
```bash
# !!! Backup all your Data before this step !!!
# Logged out. 
# Switch to a console with Ctrl+Alt+F2. 
# Login as a root and check that your user own no processes: 
ps -U username 

# Install the necessary applications: 
sudo pacman -S rsync lsof ecryptfs-utils 

# Then encrypt your home directory: 
modprobe ecryptfs 
ecryptfs-migrate-home -u username

# Mount your encrypted home. 
ecryptfs-mount-private 

# Unwrap the passphrase and save it somewhere where only you can access it. 
ecryptfs-unwrap-passphrase 

# Run 
ls .ecryptfs 

# Edit /etc/pam.d/system-auth: 
After the line "auth required pam_unix.so" add: auth required pam_ecryptfs.so unwrap 
Above the line "password required pam_unix.so" insert: password optional pam_ecryptfs.so 
After the line "session required pam_unix.so" add: session optional pam_ecryptfs.so unwrap 
```
![enter image description here](https://normannator.de/archlinux/IMG/Encrypt_Home.PNG)
```bash
# Reboot and make sure that you can login to your desktop
# If everything workes fine, feel free to remove your Backup (done before)
```

### Install Timeshift
```bash
yay -S timeshift
```

### Set up VirtualBox
```bash
sudo pacman -S virtualbox-host-modules-arch virtualbox-guest-iso
sudo gpasswd -a <user> vboxusers
systemctl enable dkms
sudo nano /etc/modules-load.d/my-modules.conf
--> add: vboxdrv
reboot # optional
```

### Generate SSH-Key
```bash
ssh-keygen
```

### Main Maintenance
```bash
sudo pacman -S xfce4-session xrandr
sudo pacman -S xfce4-whiskermenu-plugin
sudo pacman -S alacarte 
yay -S menulibre xame
sudo pacman -S gambas3-gb-gtk3 libcanberra libcanberra-pulse sound-theme-freedesktop
# Description #
# alacarte  # Menu editor for gnome
# menulibre  # An advanced menu editor that provides modern features in a clean, easy-to-use interface
# xame  # XFCE Applications Menu Editor
# gambas3-gb-gtk3  # GTK3 toolkit component
# libcanberra + libcanberra-pulse  # A small and lightweight implementation of the XDG Sound Theme Specification
# sound-theme-freedesktop  # Freedesktop sound theme

yay -S sound-theme-smooth xfce4-volumed-pulse xfce4-mixer 
sudo pacman -S xfce4-pulseaudio-plugin xscreensaver udevil udiskie thunar thunar-volman thunar-archive-plugin thunar-media-tags-plugin
# Description #
# udevil  # Mount and unmount without password
# udiskie  # Removable disk automounter using udisks

sudo pacman -S networkmanager-openvpn
systemctl restart networkmanager

sudo pacman -S ffmpegthumbnailer tumbler raw-thumbnailer libgsf catfish mlocate
yay -S env-modules env-modules-tcl
yay -S xfce4-sensors-plugin-nvidia
# Description #
# ffmpegthumbnailer  # Lightweight video thumbnailer that can be used by file managers.
# tumbler  # D-Bus service for applications to request thumbnails
# raw-thumbnailer  # A lightweight and fast raw image thumbnailer that can be used by file managers.
# libgsf  # An extensible I/O abstraction library for dealing with structured file formats
# catfish  # Versatile file searching tool
# mlocate  # Merging locate/updatedb implementation
# env-modules/-tcl  # Provides for an easy dynamic modification of a user's environment via modulefile.

sudo pacman -S xarchiver file-roller ark archlinux-wallpaper xwallpaper archivetools archlinux-menu archlinux-themes-slim archlinux-xdg-menu fastjar
```

### XFCE4 Settings

#### Popup Menu on Windows-Icon
In Keyboard Shortcuts set ```xfce4-popup-whiskermenu``` run on windows-key

#### setup Theme and Fonts
```bash
sudo pacman -S arc-gtk-theme arc-icon-theme
yay -S flat-remix catarell catarell-fonts
sudo pacman -S futurist
```
#### setup undercover
```bash
yay -S kali-undercover
alias ku=/usr/bin/kali-undercover

#### cmatrix is the opposite ;) 

```
#### add additional packages
```bash
sudo systemctl enable sshd
sudo systemctl start sshd

sudo pacman -S redshift
(sudo systemctl enable redshift)

yay -S xfce4-screensaver mugshot gnome-system-tools

# To Install this, you need to do some more complicated stuff
git clone https://aur.archlinux.org/gnome-system-tools.git
cd gnome-system-tools/
nano PKGBUILD (change FTP (first to HTTP/S)
makepkg -si
yay -S gnome-doc-utils
yay -S liboobs -> git clone https://aur.archlinux.org/liboobs.git
cd liboobs
makepkg -si (maybe edit PKGBUILD)
yay -S system-tools-backends
makepkg -si

yay -S gnome-system-tools
# Now Clean Your AUR Clones Directory

sudo pacman -S openvpn networkmanager-openvpn
# import openvpn config with:
openvpn import --config <config-incl-path>
nmcli connection import type openvpn file <file-to-ovpn>

yay -S pycharm-professional
```

#### Make XFCE4 look good!

##### Docky/Plank
```bash
yay -S docky/plank  # Dock like OSX
```
##### Desktop
1. Right-Clock on Desktop -> Settings:
2. Select *usr/share/backgrounds/archlinux* as background folder and choose a nice background
3. Symbols -> Default Icons: 
	* Deselect Home and Filesystem.
	* Iconsize: 40
	* Show icon tooltips. Size: 100

##### Theme, Icon and Font
Settings->Appearance:
1. Style: Arc-Dark
2. Icons: Flat-Remix-Blue-Dark
3. Fonts: 
	* Default Font: Cantarell Regular - 11
	* Default Monospace Font: Fira Code Medium - 10
	* Rendering:
		* Hinting: Low (Gering)
		* Sub-Pixel order: RGB

Settings->Window Manager:
1. Style: 
	* Style: Arc-Dark
	* Title Font: Cantarell Bold - 9
	* Hide arrow up

For more Themes visit [this Page](https://www.xfce-look.org/)

##### Terminal
![enter image description here](https://normannator.de/archlinux/IMG/term-1.png)
![enter image description here](https://normannator.de/archlinux/IMG/term-2.png)
![enter image description here](https://normannator.de/archlinux/IMG/term-3.png)
![enter image description here](https://normannator.de/archlinux/IMG/term-4.png)
![enter image description here](https://normannator.de/archlinux/IMG/term-5.png)
![enter image description here](https://normannator.de/archlinux/IMG/term-6.png)

##### LightDM Settings
Theme: Arc-Dark
Icon: Arc
Font: Cantarell Bold - 10

##### Screensaver
Settings->Screensaver: 
* Floating XFCE
* Disable: Lock Screen with System Sleep
Settings->Keyboard: xflock4 (default Ctrl+Alt+L/Delete)

##### Whisker-Menu
Delete the Old Menu Button and add the installed Wisker Menu to the Top Panel.
Right Click and Open Settings of Whisker Menu:
* List view
* Symbol-Size: Small
* Symbol Category: Smaller
* Head->Symbol only + Symbol -> ArchLinux Symbol 
* Behaviour: Select on Hover

Settings->Keyboard->Shortcuts: add on pressing *super* (Windows Button): xfce4-popup-whiskermenu

#### More Configuration
Open Whisker-Menu and Klick on the User Icon:
-> Mugshot should open: Enter your Credentials

In Settings: Type "User" and Open: *User Settings* (gnome-system-tools)
Administrate User and Groups ...

Logout and Login again that the changes take effect.

Edit the ~/.bashrc file to add alias permanently 

Usefull Shortcuts:
- Super + T: Terminal
- Super + C: Chrome
- Super + F: File Manager


### And More Programms
```bash
# Version Control
yay -S github-desktop
yay -S gitkraken

# KVM Integration
sudo pacman -S qemu virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat

# Python PIP
sudo pacman -S python-pip

# For Python Code Checking
pip3 install pylint

# install Zoom Client
yay -S zoom

# Write IMAGES to USB-Drives
yay -S balena-etcher

# Music Streaming
yay -S spotify

# communication
yay -S slack-desktop
```

### Additional Optional Packages

#### Config Light DM Greeter
```bash
sudo pacman lightdm-webkit2-greeter

# Open /etc/lightdm/lightdm.conf
user-session=awesome # your user-session -> ls /usr/share/xsessions/*.desktop

greeter-session=lightdm-webkit2-greeter
```

#### Alternative Terminal: Terminator
For More information click [here](https://www.youtube.com/watch?v=iaXQdyHRL8M&t=215s&ab_channel=ChrisTitusTech)


## Post Steps and Maintenance

### Remove Orphans
```bash
sudo pacman -Rns $(pacman -Qtdq)  # Removes unused Packages
yay -Sc  # Removes Cache of YAY

# Also: https://ostechnix.com/recommended-way-clean-package-cache-arch-linux/
```

### Optimize pacman's database access speeds
```bash
sudo pacman-optimize
```

### Make Arch Stable (Golden Rules)
1. Do Backups with Timeshift and BtrFS
2. Update only every Week
3. Install Linux and Linux-LTS but us the LTS Kernal. If the System breaks, you can switch to the non LTS Kernal when the system is not booting. (use: ```uname -sr``` to check the current used kernel)
4. Ignore Linux Packages on Update (pacman.conf -> ignorepkg = Linux)
5. Do not install out of date packages! Use the AUR Page and search under package actions: *Flagged out-of-date (2019 <year not updeated>)*. You can also see this in the aur-tool logs!


### Check for errors
```bash
sudo systemctl --failed sudo journalctl -p 3 -xb
```

### (optional) Backup your System manually
```bash
sudo rsync -aAXvP --delete --exclude=/dev/* --exclude=/proc/* --exclude=/sys/* --exclude=/tmp/* --exclude=/run/* --exclude=/mnt/* --exclude=/media/* --exclude=/lost+found --exclude=/home/.ecryptfs / /mnt/backupDestination/
```
