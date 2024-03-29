
This is a guide for installing Debian GNU/Linux on a Pogoplug Pro.

This is not the official source for any of the mentioned binaries, it is simply an unofficial mirror and an alternate guide. The official source is [bohdi's posts on the doozan forum](https://forum.doozan.com/read.php?2,16044).

# Sources

All of this was gathered from other sources. The only thing I made is this readme file.

* [noisebridge wiki page](https://www.noisebridge.net/wiki/Pogo)
* [doozan forum u-boot page](https://forum.doozan.com/read.php?3,16017)
* [doozan forum rootfs and kernel page](https://forum.doozan.com/read.php?2,16044)
* [serial console pinout source](http://folderform.tk/archive/Pogoplug-e02-serial-console-windows6050.html)

# Wiring up the serial connection

You can use an old-school CD-ROM to soundcard cable or a Seeed studio groove connector. I'm not sure what the actual names of these are. When looking at the board with the text printed on the circuit board right-side up, the pins left to right are:

```
Ground, RX, TX, VCC
```

It's using 3.3 V 115200 8n1.

# Serial console

The serial console drops you straight into a root shell with no password prompt.

Set root password:

```
passwd
```

Make a note of the MAC address. You'll need it later. It can also be found on the label under the PogoPlug device:

```
ifconfig eth0
```

You're looking for something like:

```
ether 12:3b:54:af:43:11
```

or:

```
HWaddr 12:3b:54:af:43:11
```

Check for bad blocks:

```
dmesg | grep -i 'bad'
```

If you see a bad block in the first 2 MB then do not proceed. That would look something like this:

```
[    2.413231] Scanning device for bad blocks
[    2.417731] Bad eraseblock 3 at 0x000000060000
```

Start dropbear:

```
/etc/init.d/dropbear.sh start
```

Either run a DHCP server (like dnsmasq) on your computer and look for the DHCP lease to see the PogoPlug's IP:

```
tail -f /var/log/syslog
```

Then connect via ethernet and the IP show up in the log. Or you can set a static IP on the PogoPlug with `ifconfig`:

```
ifconfig eth0 192.168.1.2 netmask 255.255.255.0 up
```

Now scp the uboot image:

```
scp uboot.2015.10-tld-2.ox820.bodhi.tar root@192.168.1.2:/tmp/
```

Create the file `/etc/fw_env.config` with the following content:

```
# pogoplug v3
  /dev/mtd1               0x00100000      0x20000         0x20000
```

Extract the uboot archive:

```
cd /tmp
tar xvf uboot.2015.10-tld-2.ox820.bodhi.tar
```

Erase 6 blocks on mtd1:

```
/usr/sbin/flash_erase /dev/mtd1 0x0 6
```

Expected output:

```
Erase Total 6 Units                                                             
Performing Flash Erase of length 131072 at offset 0xa0000 done
```

Flash encoded spl stage1 to 0x0:

```
/usr/sbin/nandwrite /dev/mtd1 uboot.spl.2013.10.ox820.850mhz.mtd0.img
```

Expected output:

```
Writing data to block 0 at offset 0x0
```

Flash uboot to 0x40000:

```
/usr/sbin/nandwrite -s 262144 /dev/mtd1 uboot.2015.10-tld-2.ox820.mtd0.img
```

Expected output:

```
Writing data to block 2 at offset 0x40000
Writing data to block 3 at offset 0x60000
Writing data to block 4 at offset 0x80000
Writing data to block 5 at offset 0xa0000
```

Erase 1 block starting 0x00100000:

```
/usr/sbin/flash_erase /dev/mtd1 0x00100000 1
```

Expected output:

```
Erase Total 1 Units                                                             
Performing Flash Erase of length 131072 at offset 0x100000 done
```

Flash uboot environment to 0x00100000

```
/usr/sbin/nandwrite -s 1048576 /dev/mtd1 uboot.2015.10-tld-2.ox820.environment.img
```

Expected output:

```
Writing data to block 8 at offset 0x100000
```

You're now done flashing the new uboot.

# Creating a bootable Debian USB stick

On your computer get a USB stick and create a single Linux partition on it. (e.g. use `fdisk /dev/sdX` then `d` to delete each partition, then `n` to create a new partition which will automatically created as a Linux partition, then `w` to write the changed).

Then format the partition with an ext4 filesystem labeled 'rootfs' (this is important):

```
sudo mkfs.ext4 -L rootfs  /dev/sdX1
```

Mount the partition:

```
sudo mount /dev/sdX1 /mnt
```

Download the rootfs archive from the releases page of [this repository](https://github.com/Juul/pogoplug-pro-debian).

Extract the rootfs archive:

```
cd /mnt
sudo tar xvjf /path/to/Debian-4.14.180-oxnas-tld-1-rootfs-bodhi.tar.bz2
```

Adjust the rootfs line in fstab by editing `/mnt/etc/fstab`, changing 'ext3' to 'ext4':

```
LABEL=rootfs    /               ext4    noatime,errors=remount-ro 0 1
```

Unmount:

```
cd /
sudo umount /mnt
```

Plug the usb stick into your PogoPlug and reboot it.

# First boot into the new system

It should boot into Debian and ask you to log in. The default credentials are:

```
username: root
password: root
```

The first thing to do is generate an ssh key:

```
rm /etc/ssh/ssh_host*
ssh-keygen -A
```

You might run out of ram during the next step, so first stop a few running services:

```
/etc/init.d/ssh stop
/etc/init.d/avahi-daemon stop
```

Ensure you have an internet connection (plug the ethernet cable into something appropriate) then:

```
apt-get update
apt-get upgrade
```

Set the correct MAC address. This is the address you wrote down earlier. If you didn't, it should be written on the label under the PogoPlug device:

```
fw_setenv ethaddr 12:3b:54:af:43:11
```

# Upgrading the kernel

From your compter, scp the kernel to the PogoPlug:

```
scp linux-5.4.224-oxnas-tld-1.bodhi.tar.bz2 root@192.168.1.2:/boot/
```

On the PogoPlug, extract the files:

```
cd /boot
tar xvjf linux-5.4.224-oxnas-tld-1.bodhi.tar.bz2
tar xf linux-dtb-5.4.224-oxnas-tld-1.tar
```

Install the new kernel:

```
dpkg -i linux-image-5.4.224-oxnas-tld-1_1.0_armel.deb
```

Backup old uImage and uInitrd:

```
mv uImage uImage.bak
mv uInitrd uInitrd.bak
```

Create new uImage and uInitrd:

```
mkimage -A arm -O linux -T kernel -C none -a 0x60008000 -e 0x60008000 -n Linux-5.4.224-oxnas-tld-1 -d vmlinuz-5.4.224-oxnas-tld-1 uImage
mkimage -A arm -O linux -T ramdisk -C gzip -a 0x60000000 -e 0x60000000 -n initramfs-5.4.224-oxnas-tld-1  -d initrd.img-5.4.224-oxnas-tld-1 uInitrd
```

Clean up:

```
rm *.deb
rm *.tar
rm *.tar.bz2
```

Sync the filesystem:

```
sync
```

Now reboot:

```
reboot
```

That's it!

# WiFi

The wifi adapter needs a closed firmware.

To get the wifi working, add the following line to `/etc/apt/sources.list`:

```
deb http://ftp.us.debian.org/debian stretch main non-free
```

Then run:

```
apt update
```

Then install lspci and iw:

```
apt install pciutils iw
```

Run lspci:

```
lspci
```

If you have a Ralink RT3090 you should do:

```
apt install firmware-misc-nonfree
```

If you have an RT8xxx card you shoul do:

```
apt install firmware-realtek
```

You can install both if you're not sure.

Now you'll need a few more packages:

```
apt install wpasupplicant hostapd bridge-utils wireless-tools iw
```

If you want to use the wifi card as a client, edit `/etc/network/interfaces` adding the wollowing:

```
auto wlp0s0
iface wlp0s0 inet dhcp
  wpa-ssid your_wifi_ssid
  wpa-psk your_wifi_password
```

or if your wifi has no password you can use:

```
auto wlp0s0
iface wlp0s0 inet dhcp
  wireless-essid your_wifi_ssid
```

Don't put quotes around the ssid or password, not even if it has spaces.

Then test it with:

```
ifup -v wlp0s0
```

# Running an anonymous ftp server

Install the vsftpd sever:

```
sudo apt install vsftpd
```

Use the `vsftpd.conf` from this git repo:

```
sudo cp ./vsftpd.conf /etc/
```

The directory shared via ftp will be `/srv/ftp`. You can change this by altering the `ftp` users' home directory in `/etc/passwd`.

If you want to share an external drive then first partition it, then format it and give it a unique label. Let's assume it's called `/dev/sdb`.

__WARNING THE FOLLOWING WILL DELETE EVERYTHIN ON THE DRIVE__

To partition run `sudo fdisk /dev/sdb`, then hit `d` and enter again until there are no partitions left. Now hit `c` to create a partition and hit enter for each question to accept the defaults, then hit `w` to write the changes to disk.

Now format the drive:

```
# WARNING THIS WILL DELETE EVERYTHINg ON THE DRIVE

sudo mkfs.ext4 -L my-labeled-drive /dev/sdb1
```

To make the drive automatically mount in the ftp directory on boot add the following line to `/etc/fstab`:

```
LABEL=my-labeled-drive /srv/ftp        ext4    noatime,errors=remount-ro 0 2
```

Now mount it to ensure that the fstab line works:

```
sudo mount /srv/ftp
```

While the drive is mounted, set the permissions:

```
sudo chown root.ftp /srv/ftp
sudo chmod 755 /srv/ftp
```

If you want users to be able to upload to an `uploads` directory then do:

```
sudo mkdir /srv/ftp/uploads
sudo chown ftp.ftp /srv/ftp/uploads
sudo chmod 755 /srv/ftp/uploads
```

by default no-one will be able to delete from the `uploads/` directory. To change this modify the line in `/etc/vsftpd.conf` saying `chown_uploads=YES` to `chown_uploads=NO`. Note that this will allow anyone to delete anything in the uploads directory.

Now restart vsftpd to make it use the new configuration:

```
sudo /etc/init.d/vsftpd restart
```

Create a test file to download:

```
sudo echo "Welcome to my ftp server!" > /srv/ftp/README
```

Test that you can connect to the ftp server using e.g. `ncftp` from another computer and download and upload

```
sudo apt install ncftp
echo "TEST UPLOAD CONTENT" > test
ncftp <ip_address_of_pogogplug>
get README
cd uploads/
put test
```

