# EBA Unattended Ubuntu Install
Create a custom Ubuntu installation image for an unattended install for EBA. I have tested with with Ubuntu 14.04.1

See also: [EBA Setup Scripts](https://github.com/HarlemSquirrel/eba-setup-scripts)

# Setting up the environment

### Download the ISO
You can grab one from [Download Ubuntu Desktop](http://www.ubuntu.com/download/desktop)

### Install needed software
```sh
sudo apt-get install squashfs-tools dchroot
```

### Mount the Live CD
```sh
mkdir /tmp/livecd
sudo mount -o loop ~/Downloads/ubuntu.iso /tmp/livecd
```

### Create a working area and copy contents over to the working area
```sh
mkdir -p ~/livecd/cd
rsync --exclude=/casper/filesystem.squashfs -a /tmp/livecd/ ~/livecd/cd
mkdir ~/livecd/squashfs  ~/livecd/custom
sudo modprobe squashfs
sudo mount -t squashfs -o loop /tmp/livecd/casper/filesystem.squashfs ~/livecd/squashfs/
sudo cp -a ~/livecd/squashfs/* ~/livecd/custom
```

### Get into the folder
#### 64-bit
```sh
cd ~/livecd64
```
#### 32-bit
```sh
cd ~/livecd32
```

### Network access (optional)
```sh
sudo cp /etc/resolv.conf /etc/hosts custom/etc/
```


### Create a pseudo filesystem and chroot into it
```sh
sudo chroot custom /bin/bash -l
mount -t proc none /proc/ && mount -t sysfs none /sys/ && mount -t devpts none /dev/pts
```


# Customize

### add sources to /etc/apt/sources.list
```sh
nano /etc/apt/sources.list
```

```
deb http://archive.ubuntu.com/ubuntu/ trusty main restricted
deb http://security.ubuntu.com/ubuntu/ trusty-security main restricted

deb http://us.archive.ubuntu.com/ubuntu/ trusty-updates main restricted
deb-src http://us.archive.ubuntu.com/ubuntu/ trusty-updates main restricted

deb http://us.archive.ubuntu.com/ubuntu/ trusty universe
deb-src http://us.archive.ubuntu.com/ubuntu/ trusty universe
deb http://us.archive.ubuntu.com/ubuntu/ trusty-updates universe
deb-src http://us.archive.ubuntu.com/ubuntu/ trusty-updates universe

deb http://us.archive.ubuntu.com/ubuntu/ trusty multiverse
deb-src http://us.archive.ubuntu.com/ubuntu/ trusty multiverse
deb http://us.archive.ubuntu.com/ubuntu/ trusty-updates multiverse
deb-src http://us.archive.ubuntu.com/ubuntu/ trusty-updates multiverse

deb http://us.archive.ubuntu.com/ubuntu/ trusty-backports main restricted universe multiverse
deb-src http://us.archive.ubuntu.com/ubuntu/ trusty-backports main restricted universe multiverse

deb http://archive.canonical.com/ubuntu trusty partner
deb-src http://archive.canonical.com/ubuntu trusty partner
```

### Add repositories
```sh
add-apt-repository ppa:webupd8team/java
add-apt-repository ppa:linrunner/tlp
```

### Install some packages
*If no Internet, set up resolv.conf*
```sh
echo "nameserver 127.0.1.1" > /etc/resolv.conf
apt-get update
apt-get install bleachbit openssh-server oracle-java7-installer audacity mixxx libportaudio2 vlc gimp musescore openshot openscad edubuntu-artwork ubuntu-edu-secondary ubuntu-restricted-extras python-appindicator tlp tlp-rdw ttf-ancient-fonts hyphen-en-us mythes-en-us firefox-locale-es gimp-help-es language-pack-es language-pack-gnome-es libreoffice-help-es libreoffice-l10n-es myspell-es wspanish
```

### Setup restricted DVD playback
```sh
/usr/share/doc/libdvdread4/install-css.sh
```

### Disable Apport error reporting
```sh
sed -i "s/enabled=1/enabled=0/g" /etc/default/apport
```

### Remove unwanted packages
```sh
apt-get purge unity-scope-imdb unity-scope-musicstores unity-scope-zotero unity-scope-click-autopilot unity-scope-deviantart unity-scope-gallica unity-scope-gdocs unity-scope-github unity-scope-googlenews unity-scope-launchpad unity-scope-mediascanner unity-scope-onlinemusic unity-scope-openweathermap unity-scope-soundcloud unity-scope-sshsearch unity-scope-yahoostock unity-lens-photos unity-lens-video unity-scope-audacious unity-scope-chromiumbookmarks unity-scope-clementine unity-scope-click unity-scope-colourlovers unity-scope-gdrive unity-scope-gmusicbrowser unity-scope-gourmet unity-scope-guayadeque unity-scope-mediascanner2 unity-scope-musique unity-scope-openclipart unity-scope-texdoc unity-scope-tomboy unity-scope-video-remote unity-scope-virtualbox unity-scope-yelp unity-webapps-service account-plugin-ubuntuone ubuntu-purchase-service deja-dup indicator-messages empathy gwibber thunderbird transmission-gtk pidgin unity-control-center-signon landscape-* webbrowser-app;
```

### Update system packages
```sh
apt-get dist-upgrade
```

#### Cleanup
Run this after you are happy with the customizations.
```sh
apt-get autoremove && apt-get clean && rm -rf /tmp/* && rm -f /var/lib/dbus/machine-id
umount /proc/ && umount -l /sys/ && umount /dev/pts && exit
```


# Setting up the new ISO
```sh
chmod +w ./cd/casper/filesystem.manifest

sudo chroot ./custom dpkg-query -W --showformat='${Package} ${Version}\n' > ./cd/casper/filesystem.manifest

sudo cp ./cd/casper/filesystem.manifest ./cd/casper/filesystem.manifest-desktop
```


### Regenerate squashfs file, remove first
```sh
sudo rm -f ./cd/casper/filesystem.squashfs && sudo mksquashfs custom ./cd/casper/filesystem.squashfs -comp xz
```


### Make filesystem.size
```sh
sudo su -c 'printf $(du -sx --block-size=1 ./custom | cut -f1) > ./cd/casper/filesystem.size'
```

### Edit `txt.cfg`
Set the boot command line to use preseed files, change the first line and insert an auto-install
```sh
sudo nano ./cd/isolinux/txt.cfg
```
```
default unattended-EBA-install
label unattended-EBA-install
  menu label ^Install for EBA unattended
  kernel /casper/vmlinuz
  append  file=/cdrom/preseed/EBA.seed keyboard-configuration/layoutcode=us and console-setup/ask_detect=false boot=casper automatic-ubiquity noprompt initrd=/casper/initrd.lz --
```

### Set the EFI boot to use preseed and auto-install (64-bit only)
```sh
sudo nano ./cd/boot/grub/grub.cfg
```
```
menuentry "EBA Auto Install" {
	set gfxpayload=keep
	linux	/casper/vmlinuz.efi  file=/cdrom/preseed/EBA.seed keyboard-configuration/layoutcode=us and console-setup/ask_detect=false boot=casper automatic-ubiquity noprompt --
	initrd	/casper/initrd.lz
}
```


### Set menu timeout to 5 seconds
```sh
sudo nano ./cd/isolinux/isolinux.cfg
```
```
timeout 50
```


### Set disc name in README.diskdefines
```sh
sudo nano ./cd/README.diskdefines
```
define DISKNAME  Ubuntu 14.04.1 LTS amd64 for EBA by KMack


### Copy preseed file and scripts to preseed folder
The `EBA.seed` file is included in this repository. The other files can be found in my [EBA Setup Scripts](https://github.com/HarlemSquirrel/eba-setup-scripts) repository.
```sh
sudo cp ~/Dropbox/Scripts/{unattended-install/EBA.seed,eba-setup-netrun.sh,eba-setup.sh,loaner-setup.sh,set-hostname.sh,ProxyEBA.sh,pupil-setup.sh,configs/10-network-manager.pkla} ./cd/preseed/
```

### Update md5sums
```sh
sudo su -c 'find ./cd -type f -exec md5sum {} + > ./cd/md5sum.txt'
```


# Create the new iso
### 32-bit
```sh
cd ./cd && sudo mkisofs -D -r -V "Edu14x32-EBA-Auto" -cache-inodes -J -l -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -o ../Edu14x32_EBA_$(date +"%Y-%m-%d").iso . && cd ..
```

### 64-bit with EFI support
```sh
cd ./cd && sudo mkisofs -U -A "Edu14x64-EBA-Auto" -V "Edu14x64-EBA-Auto" -volset "Edu14x64-EBA-Auto" -J -joliet-long -r -v -T -o ../Edu14x64_EBA_$(date +"%Y-%m-%d").iso -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -eltorito-alt-boot -e boot/grub/efi.img -no-emul-boot . && cd ..
```

# Install with UNetBootin 608-1 or with Ubuntu Startup Disc Creator
```sh
sudo usb-creator-gtk
```


# Sources
http://askubuntu.com/questions/122505/how-do-i-create-a-completely-unattended-install-of-ubuntu
http://askubuntu.com/questions/48535/how-to-customize-the-ubuntu-live-cd
https://help.ubuntu.com/community/LiveCDCustomization
