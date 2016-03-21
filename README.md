# raspi-fde
## raspberry pi with full disk encryption and remote unlock

I wanted to have a raspberry pi running raspbian on an encrypted filesystem (everything except /boot) and I wanted to be able to unlock the encryption via ssh.

Ressources I used:  
http://paxswill.com/blog/2013/11/04/encrypted-raspberry-pi/  
https://unix.stackexchange.com/questions/5017/ssh-to-decrypt-encrypted-lvm-during-headless-server-boot/79203#79203

```bash
# raspbian default install
dd if=2016-02-09-raspbian-jessie-lite.img | pv | sudo dd of=/dev/sdX bs=4M
# resize partitions so that there is more space on /boot (180MB) and some free space on / (gksudo gparted)
# put sd into raspi, start raspi
ssh pi@raspi-ip # pw: raspberry
passwd
sudo su; passwd; exit
sudo raspi-config # do not expand filesystem, reboot
sudo apt-get install busybox cryptsetup dropbear
sudo mkinitramfs -v -o /boot/initramfs.gz # creates keys and directories for dropbear
sudo vi /etc/initramfs-tools/initramfs.conf # add DROPBEAR=y and CRYPTSETUP=y, does not enforce including cryptsetup!?
sudo vi /usr/share/initramfs-tools/hooks/cryptroot # enforce setup="yes" so that cryptsetup is included in initramfs
sudo vi /etc/initramfs-tools/root/.ssh/authorized_keys # add your ssh-pubkeys
sudo mkinitramfs -v -o /boot/initramfs.gz # again so that the settings are applied, check output for dropbear and cryptsetup
sudo vi /boot/config.txt # add: initramfs initramfs.gz followkernel
sudo reboot # test reboot
sudo shutdown -h now
# insert sd into another computer
sudo dd if=/dev/sdX2 bs=4M | pv | dd of=raspi-plain-root.img bs=4M
/sbin/e2fsck -f raspi-plain-root.img
/sbin/resize2fs -M raspi-plain-root.img
# resize second partition to its maximum (gksudo gparted)
sudo cryptsetup luksFormat -c aes-xts-plain64 -s 256 -h sha256 -y /dev/sdX2
sudo cryptsetup -v luksOpen /dev/sdX2 raspi_crypt
dd if=raspi-plain-root.img bs=4M | pv | sudo dd of=/dev/mapper/raspi_crypt bs=4M
sudo e2fsck /dev/mapper/raspi_crypt
sudo resize2fs /dev/mapper/raspi_crypt
sudo e2fsck /dev/mapper/raspi_crypt
mkdir /tmp/piroot
sudo mount /dev/mapper/raspi_crypt /tmp/piroot/
sudo mount /dev/sdX1 /tmp/piroot/boot/
sudo vim /tmp/piroot/boot/cmdline.txt # Change root=/dev/mmcblk0p2 to root=/dev/mapper/raspi_crypt and add cryptdevice=/dev/mmcblk0p2:raspi_crypt
sudo vim /tmp/piroot/etc/fstab # change /dev/mmcblk0p2 to /dev/mapper/raspi_crypt
sudo vim /tmp/piroot/etc/crypttab # add raspi_crypt /dev/mmcblk0p2 none luks
sudo umount /tmp/piroot/boot/ /tmp/piroot/
sudo cryptsetup luksClose raspi_crypt
# reboot raspi with lan
ssh root@raspi-ip -o "UserKnownHostsFile=~/.ssh/known_hosts-raspi-dropbear"
/sbin/cryptsetup -v luksOpen /dev/mmcblk0p2 raspi_crypt
ps -eo pid,ppid,comm,args # kill sh that was startet by init (1); exit -> pi boots
ssh pi@raspi-ip
sudo mkinitramfs -v -o /boot/initramfs.gz # check for cryptsetup
sudo reboot
ssh root@raspi-ip -o "UserKnownHostsFile=~/.ssh/known_hosts-raspi-dropbear"
/lib/cryptsetup/askpass "enter luks password: " > /lib/cryptsetup/passfifo # exit, wait for the raspi to boot completely
ssh pi@raspi-ip # this is the "normal" openssh-server, we are done :-)
# do not forget to sudo mkinitramfs -v -o /boot/initramfs.gz before reboot to make sure it is still up to date
```
