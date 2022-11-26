# My perfect minimal Gentoo linux with Xmonad desktop installation

```
#!/bin/bash

# Xmonad Perfect Desktop - EFI F2FS SSD Version
# Fast Universal Gentoo Instalation Script - Created by Realist - (c) 2022 v 1.5 Final


# SCRIPT Variables
TARGET_DISK=/dev/sda
TARGET_LAN=enp0s3
TARGET_WIFI=enp0s8

# Uncomment block named # static IP if don't use DHCP
TARGET_IP=192.168.0.23
TARGET_MASK=255.255.255.0
TARGET_GATE=192.168.0.1
TARGET_DNS='192.168.0.1 8.8.8.8'

GENTOO_ROOT_PASSWORD=toor
GENTOO_USER=realist
GENTOO_USER_PASSWORD=toor
GENTOO_HOSTNAME=xmonad
GENTOO_DOMAINNAME=gentoo.dev
GENTOO_RELEASES_URL=https://mirror.dkm.cz/gentoo/releases
GENTOO_INSTALLER_URL=http://78.45.232.18:55/xmonad/desktop

GRUB_GFX_MODE=1920x1080x32
GENTOO_KEYMAP=us
GENTOO_CONSOLEFONT=ter-v16b
GENTOO_ZONEINFO=Europe/Prague

# Choose default version php7.4 or php8.1 for CLI or APACHE
TARGET_CLI_PHP=php7.4
TARGET_APACHE_PHP=php7.4


# DISK Setup
parted -s ${TARGET_DISK} mklabel gpt
parted -a optimal ${TARGET_DISK} << END
unit mib
mkpart primary fat32 1 150
name 1 UEFI
set 1 bios_grub on
mkpart primary 150 -1
name 2 ROOT
p
quit
END


# DISK Filesystems
yes | mkfs.fat -n UEFI -F32 ${TARGET_DISK}1
yes | mkfs.f2fs -l ROOT -O extra_attr,inode_checksum,sb_checksum -f ${TARGET_DISK}2
mkdir -p /mnt/gentoo
mount -t f2fs ${TARGET_DISK}2 /mnt/gentoo
mkdir -p /mnt/gentoo/boot
mount ${TARGET_DISK}1 /mnt/gentoo/boot


# Actual Stage3 & Portage Install
cd /mnt/gentoo
STAGE3_PATH_URL=$GENTOO_RELEASES_URL/amd64/autobuilds/latest-stage3-amd64-openrc.txt
STAGE3_PATH=$(curl -s $STAGE3_PATH_URL | grep -v "^#" | cut -d" " -f1)
STAGE3_URL=$GENTOO_RELEASES_URL/amd64/autobuilds/$STAGE3_PATH
wget $STAGE3_URL
tar xpf $(basename $STAGE3_URL) --xattrs-include='*.*' --numeric-owner
mkdir -p /mnt/gentoo/var/db/repos/gentoo
mkdir -p /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/
cp /etc/resolv.conf /mnt/gentoo/etc/
rm /mnt/gentoo/stage3*

# MOUNTING System FS
mount -t proc none /mnt/gentoo/proc
mount -t sysfs none /mnt/gentoo/sys
mount --rbind /sys /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --rbind /run /mnt/gentoo/run
mount --make-rslave /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/run

test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm

# CHROOT - CONFIG SETUP AND START INSTALL PACKAGES
cat > /mnt/gentoo/root/gentoo-chroot.sh << END
#!/bin/bash

emerge-webrsync
cd /etc/portage/
rm make.conf
rm -R package.use
rm -R package.accept_keywords
rm -R package.mask
wget -q $GENTOO_INSTALLER_URL/make.conf
wget -q $GENTOO_INSTALLER_URL/package.accept_keywords
wget -q $GENTOO_INSTALLER_URL/package.use
wget -q $GENTOO_INSTALLER_URL/package.license
wget -q $GENTOO_INSTALLER_URL/package.mask

sed -i 's/UTC/local/g' /etc/conf.d/hwclock

cat >> /etc/fstab << IEND
/dev/sda1   /boot   vfat    noatime     0 2
/dev/sda2   /       f2fs    defaults,rw 0 0
IEND

sed -i 's/localhost/$GENTOO_HOSTNAME/g' /etc/conf.d/hostname
sed -i 's/default8x16/$GENTOO_CONSOLEFONT/g' /etc/conf.d/consolefont
sed -i 's/us/$GENTOO_KEYMAP/g' /etc/conf.d/keymaps

sed -i 's/127.0.0.1/#127.0.0.1/g' /etc/hosts
echo "127.0.0.1 $GENTOO_HOSTNAME.$GENTOO_DOMAINNAME $GENTOO_HOSTNAME localhost" >> /etc/hosts

cat >> /etc/locale.gen << IEND
cs_CZ ISO-8859-2
cs_CZ.UTF-8 UTF-8
IEND

cat >> /etc/env.d/02locale << IEND
LANG="cs_CZ.UTF-8"
LC_COLLATE="C"
IEND

locale-gen
echo "$GENTOO_ZONEINFO" > /etc/timezone
eselect locale set 7
. /etc/profile

# Static IP
# cat >> /etc/conf.d/net << IEND
# config_$TARGET_LAN="$TARGET_IP netmask $TARGET_MASK"
# routes_$TARGET_LAN="default via $TARGET_GATE"
# #dns_servers_$TARGET_LAN="$TARGET_DNS"
# IEND
# cd /etc/init.d/
# ln -s net.lo net.$TARGET_LAN

echo "root:$GENTOO_ROOT_PASSWORD" | chpasswd -c SHA256
useradd -m -G audio,video,usb,cdrom,portage,users,wheel -s /bin/bash $GENTOO_USER
echo "$GENTOO_USER:$GENTOO_USER_PASSWORD" | chpasswd -c SHA256


# ----------------------------------------------------------------------------

#emerge -g linux-firmware && emerge -g gentoo-kernel-bin
#emerge -g linux-firmware gentoo-sources genkernel && genkernel all

emerge -g linux-firmware zen-sources genkernel && genkernel all
emerge -g dhcpcd grub usbutils terminus-font sudo f2fs-tools app-misc/mc ranger dev-vcs/git python xmonad xmonad-contrib xmobar imagemagick ueberzug oh-my-zsh gentoo-zsh-completions zsh-completions exa ubuntu-font-family numlockx trayer-srg setxkbmap volumeicon xdotool lxrandr xorg-server alsa-utils lsof htop lxappearance lxmenu-data gnome-themes-standard rxvt-unicode urxvt-perls elementary-xfce-icon-theme notify-osd neofetch picom rofi qt5ct adwaita-qt nitrogen nm-applet pcmanfm xprop i3lock pipewire xsetroot eix roboto gentoolkit file-roller clang firefox mpv audacious pulsemixer bpytop youtube-viewer --noreplace nano

git clone https://github.com/romkatv/powerlevel10k.git /usr/share/zsh/site-contrib/oh-my-zsh/custom/themes/powerlevel10k
git clone https://github.com/zsh-users/zsh-autosuggestions.git /usr/share/zsh/site-contrib/oh-my-zsh/custom/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git /usr/share/zsh/site-contrib/oh-my-zsh/custom/plugins/zsh-syntax-highlighting

emerge -g phpmyadmin dev-db/mysql =dev-lang/php-8.1.12 =dev-lang/php-7.4.33 nodejs composer vscode sublime-text
eselect php set cli $TARGET_CLI_PHP
eselect php set apache2 $TARGET_APACHE_PHP

emerge -gUNDu @world

# ----------------------------------------------------------------------------

cat >> /etc/default/grub << IEND
GRUB_GFXMODE=$GRUB_GFX_MODE
GRUB_GFXPAYLOAD_LINUX=keep
GRUB_BACKGROUND="/boot/grub/grub.png"
GRUB_DISABLE_OS_PROBER=0
GRUB_DEFAULT=0
GRUB_TIMEOUT=10
IEND

sed -i 's/# %wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/g' /etc/sudoers

cd /home/$GENTOO_USER
rm .bashrc
wget -q $GENTOO_INSTALLER_URL/dotfiles.zip
unzip -oq dotfiles.zip
chown -R $GENTOO_USER:$GENTOO_USER /home/$GENTOO_USER/
cd i3lock-fancy
make install

cd /root
wget -q $GENTOO_INSTALLER_URL/root_dotfiles.zip
unzip -oq root_dotfiles.zip
chown -R root:root /root

cd /usr
wget -q $GENTOO_INSTALLER_URL/usr.zip
unzip -oq usr.zip

cd /opt/sublime_text
mv sublime_text sublime_text_backup
wget -q $GENTOO_INSTALLER_URL/sublime_text
chmod +x sublime_text

chsh -s /bin/zsh root
chsh -s /bin/zsh $GENTOO_USER

sed -i 's/interface/$TARGET_LAN/g' /home/$GENTOO_USER/.config/xmobar/xmobarrc0
sed -i 's/interface/$TARGET_LAN/g' /home/$GENTOO_USER/.config/xmobar/xmobarrc1
sed -i 's/wifiadapter/$TARGET_WIFI/g' /home/$GENTOO_USER/.config/xmobar/xmobarrc0
sed -i 's/wifiadapter/$TARGET_WIFI/g' /home/$GENTOO_USER/.config/xmobar/xmobarrc1

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=XMONAD --recheck ${TARGET_DISK}
cd /boot/grub
wget -q $GENTOO_INSTALLER_URL/grub.png
grub-mkconfig -o /boot/grub/grub.cfg

rm -R /usr/lib/tmpfiles.d/mysql.conf
cat >> /usr/lib/tmpfiles.d/mysql.conf << IEND
d /run/mysqld 0755 mysql mysql -
IEND
sed -i 's/SSL_DEFAULT_VHOST/PHP/g' /etc/conf.d/apache2
echo "ServerName localhost" >> /etc/apache2/httpd.conf
rm -R /var/www/localhost/htdocs/index.html
echo "<?php phpinfo(); ?>" > /var/www/localhost/htdocs/index.php
cp /var/www/localhost/htdocs/phpmyadmin/config.sample.inc.php /var/www/localhost/htdocs/phpmyadmin/config.inc.php
mkdir /var/www/localhost/htdocs/phpmyadmin/tmp/
chown -R $GENTOO_USER:apache /var/www/localhost/htdocs
chmod -R 775 /var/www/localhost/htdocs
chmod -R 777 /var/www/localhost/htdocs/phpmyadmin/tmp
echo "#cfg['blowfish_secret'] = 'WntN0150l71sLq/{w4V0:ZXFv7WcB-Qz';" >> /var/www/localhost/htdocs/phpmyadmin/config.inc.php

rc-update add consolefont default
rc-update add numlock default
rc-update add dhcpcd default
rc-update add sshd default
rc-update add dbus default
rc-update add alsasound default
rc-update add apache2 default
rc-update add mysql default

#rc-update add NetworkManager default

alsactl store

rm -R /usr/usr.zip
rm -R /root/root_dotfiles.zip
rm -R /home/$GENTOO_USER/dotfiles.zip

emerge --config mysql
END

chmod +x /mnt/gentoo/root/gentoo-chroot.sh
chroot /mnt/gentoo /root/gentoo-chroot.sh
```
