
# Create Ubuntu Live CD From Scratch

## I. Setup Host

### 1. Install tools
	sudo apt-get install squashfs-tools chroot

### 2. mout iSO image to customize-live-ubuntu-cd/livecd
	mkdir /tmp/livecd
	mkdir customize-live-ubuntu-cd
	cd customize-live-ubuntu-cd
	sudo mount -o loop ubuntu-18.04.6-desktop-amd64.iso /tmp/livecd 

### 3. Remove rootfs file
	mkdir livecd
	sudo cp -rf /tmp/livecd/* livecd && sync
	sudo rm -rf livecd/casper/filesystem.squashfs

	mkdir squashfs
	mkdir squashfs-custom

### 5. mout squashfs
	sudo mount -t squashfs -o loop /tmp/livecd/casper/filesystem.squashfs squashfs
	sudo cp -a squashfs/* squashfs-custom/

### 6. acess network
	sudo cp -f /etc/resolv.conf /etc/hosts squashfs-custom/etc/

## II. Starting customize

### 1. mount dev/run
	sudo mount --bind /dev squashfs-custom/dev
	sudo mount --bind /run squashfs-custom/run

### 1. Acess chroot
	sudo chroot squashfs-custom

### 3. mount proc/sys
	mount -t proc none /proc/
	mount -t sysfs none /sys/
	mount none -t devpts /dev/pts
	export HOME=/root
	export LC_ALL=C

### 4. Update hostname
	echo "slayder" > /etc/hostname

### 5. Update apt/source.list
	cat <<EOF > /etc/apt/sources.list
	deb http://vn.archive.ubuntu.com/ubuntu/ bionic main restricted
	deb http://vn.archive.ubuntu.com/ubuntu/ bionic-updates main restricted
	deb http://vn.archive.ubuntu.com/ubuntu/ bionic universe
	deb http://vn.archive.ubuntu.com/ubuntu/ bionic-updates universe
	deb http://vn.archive.ubuntu.com/ubuntu/ bionic multiverse
	deb http://vn.archive.ubuntu.com/ubuntu/ bionic-updates multiverse
	deb http://vn.archive.ubuntu.com/ubuntu/ bionic-backports main restricted universe multiverse
	deb http://security.ubuntu.com/ubuntu bionic-security main restricted
	deb http://security.ubuntu.com/ubuntu bionic-security universe
	deb http://security.ubuntu.com/ubuntu bionic-security multiverse
	EOF

	sudo apt-get update

### 6. Install unnecessary packages/apps

#### 6.1 install packages
	apt-get install -y \
	    clamav-daemon \
	    terminator \
	    apt-transport-https \
	    software-properties-common \
	    wget \
	    curl \
	    vim \
	    less \
	    ibus-unikey \
		git 

#### 6.2 install apps 
	// visual code
	sudo apt update
	wget -q https://packages.microsoft.com/keys/microsoft.asc -O- | sudo apt-key add -
	sudo add-apt-repository "deb [arch=amd64] https://packages.microsoft.com/repos/vscode stable main"
	sudo apt install code
	
	// chrome
	sudo apt-get install libxss1 libappindicator1 libindicator7
	wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
	sudo apt install ./google-chrome*.deb
	
	// gparted
	sudo apt-get install gparted
	
	// beyond compare
	sudo dpkg --add-architecture i386
	sudo apt update
	sudo apt install ./bcompare-4.4.2.26348_amd64.deb
	
	// notepad++
	sudo apt-get install snapd snapd-xdg-open

### 7. Remove unnecessary packages
#### 7.1 Remove apps
	
	sudo apt-get remove package-name
		Any of the above commands will remove the specified package, but they will leave behind configuration files, 
		and in some cases, other files that were associated with the package.
	sudo apt-get purge package-name
	
	
	sudo apt-get purge firefox

#### 7.2 remove gnome-games and non-language		
	sudo apt-get remove --purge gnome-games*
	sudo apt-get remove --purge `dpkg-query -W --showformat='${Package}\n' | grep language-pack | egrep -v '\-en'`
	
	// show all pakages
	dpkg-query -W --showformat='${Package}\n' | less
	
#### 7.3 Clean tempotary files and quit
	apt-get clean
	rm -rf /tmp/* ~/.bash_history
	umount /proc
	umount /sys
	umount /dev/pts
	exit
	
## III. Create ISO image
### 1. umout dev/run
	sudo umount squashfs-custom/dev
	sudo umount squashfs-custom/run
	
### 2. Recreate manifest file
	sudo chroot squashfs-custom/ dpkg-query -W --showformat='${Package} ${Version}\n' | sudo tee livecd/casper/filesystem.manifest
	
	sudo cp -v livecd/casper/filesystem.manifest livecd/casper/filesystem.manifest-desktop
	sudo sed -i '/ubiquity/d' livecd/casper/filesystem.manifest-desktop
	sudo sed -i '/casper/d' livecd/casper/filesystem.manifest-desktop
	sudo sed -i '/discover/d' livecd/casper/filesystem.manifest-desktop
	sudo sed -i '/laptop-detect/d' livecd/casper/filesystem.manifest-desktop
	sudo sed -i '/os-prober/d' livecd/casper/filesystem.manifest-desktop
	
### 3. Compress chroot -> filesystem.squashfs
	sudo mksquashfs squashfs-custom livecd/casper/filesystem.squashfs
	
### 4. Recreate md5 checksum
	cd livecd && find . -type f -print0 | xargs -0 md5sum > /tmp/md5sum.txt && cp -f /tmp/md5sum.txt .
	
### 5. Create ISO image
	cd livecd && sudo mkisofs -r -V "phonglt15-Ubuntu-Live-Custom" -b isolinux/isolinux.bin -c isolinux/boot.cat -cache-inodes -J -l -no-emul-boot \
	-boot-load-size 4 -boot-info-table -o ../phonglt15-Ubuntu-Live-Custom.iso .
