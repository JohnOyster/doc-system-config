# Arch Linux Install
Author: John Oyster
Update: 2019-Apr-04

## Overview
These steps are designed to help a user install Arch Linux with
some common configurations that I use.

## Configurations
1. LVM
0. Booting with EFI
0. Encrypted disk

## Considerations
If you are installing on a laptop you may want to disable the lid closing
event:
* Edit `/etc/systemd/logind.conf` and set `HandleLidSwitch` to `ignore`.

If you are planning on connecting remotely then enable SSH. (Only keep
this enabled as long as needed):
* `# systemcll start sshd`

It is also good to find a faster mirror; see https://www.archlinux.org/mirrorlist/
* Generate mirrorlist

## Installation Procedure
1. Boot from latest iso using medium of your choice.
0. Connect to Internet:
  * https://wiki.archlinux.org/index.php/Network_configuration
  * Use wifi-menu command
0. Edit /etc/pacman.d/mirrorlist and update with fastest mirrors
0. Update package indicies:
  * `# pacman -Syyy`
0. Create the EFI partition:
  * `# fdisk /dev/sda`
    * g (creates an empty GPT partition table)
    * n (new)
    * 1 (1st partition)
    * [enter]
    * +300M (partition size)
    * t
    * 1 (for EFI)
    * w (commit to disk)
0. Create the boot partition:
  * `# fdisk /dev/sda`
    * n
    * 2
    * [enter]
    * +500M
    * w
0. Create the LVM partition:
  * `# fdisk /dev/sda`
    * n
    * 3
    * [enter]
    * [enter]
    * t
    * 3
    * 31
    * w
0. `# mkfs.fat -F32 /dev/sda1`
0. `# mkfs.ext2 /dev/sda2`
0. Enable disk encryption
  * `# cryptsetup luksFormat /dev/sda3`
  * `# cryptsetup open --type luks /dev/sda3 lvm`
0. Setup LVM partition configuration:
  * `# pvcreate --dataalignment 1M /dev/mapper/lvm`
  * `# vgvreate volgroup0 /dev/mapper/lvm`
  * `# lvcreate -L 40GB volgroup0 -n lv_root`
  * `# lvcreate -L 450GB volgroup0 -n lv_home`
  * `# modprobe dm_mod`
  * `# vgscan`
  * `# vgchange -ay`
0. `# mkfs.ext4 /dev/volgroup0/lv_root`
0. `# mkfs.xfs /dev/volgroup0/lv_home`
0. `# mount /dev/volgroup0/lv_root /mnt`
0. `# mkdir /mnt/boot`
0. `# mkdir /mnt/home`
0. `# mount /dev/sda2 /mnt/boot`
0. `# mount /dev/volgroup0/lv_home /mnt/home`
0. `# pacstrap -i /mnt base`
0. `# genfstab -U -p /mnt >> /mnt/etc/fstab`
0. `# arch-chroot /mnt`
0. `# pacman -S base-devel grub efibootmgr dosfstools openssh os-prober mtools linux-headers`
0. Edit `/etc/mkinitcpio.conf` and add `encrypt lvm2` in between `block` and `filesystems`
0. `# mkinitcpio -p linux`
0. `# mkinitcpio -p linux-lts`
0. `# nano /etc/locale.gen` (uncomment en_US.UTF-8)
0. `# locale-gen`
0. Enable `root` logon via `ssh`
0. `# systemctl enable sshd.service`
0. `# passwd` (for setting root password)
0. Edit `/etc/default/grub`:
  + `cryptdevice=<PARTUUID>:volgroup0` to the `GRUB_CMDLINE_LINUX_DEFAULT` line
  * If using standard device naming, the option will look like this:
    * `cryptdevice=/dev/sda3:volgroup0`
0. `# mkdir /boot/EFI`
0. `# mount /dev/sda1 /boot/EFI`
0. `# grub-install --target=x86_64-efi  --bootloader-id=grub_uefi --recheck`
0. `# cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo`
0. `# grub-mkconfig -o /boot/grub/grub.cfg`
0. Create swap file:
  * `# fallocate -l  8G /swapfile` (Set to ~ 1.5x amount of RAM)
  * `# chmod 600 /swapfile`
  * `# mkswap /swapfile`
  * `# echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab`
0. `# exit`
0. `# umount -a`
0. `# reboot` (keep your fingers crossed)

## Post-Install Steps
1. `# localectl set-locale LANG="en_US.UTF-8"`
0. 
