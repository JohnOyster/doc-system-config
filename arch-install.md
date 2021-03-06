# Arch Linux Install
Author: John Oyster
Update: 2019-Apr-17

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
  * Note: This is now on by defualt

If you are planning on connecting remotely then enable SSH. (Only keep
this enabled as long as needed):
* `# systemctl start sshd`

It is also good to find a faster mirror; see https://www.archlinux.org/mirrorlist/
* Generate mirrorlist
- OR -
* Edit /etc/pacman.d/mirrorlist

## Installation Procedure
1. Boot from latest iso using medium of your choice.
2. Connect to Internet:
  * https://wiki.archlinux.org/index.php/Network_configuration
  * Use wifi-menu command
  * See [Network Configuration](#Network Stuff) for help.
3. Edit /etc/pacman.d/mirrorlist and update with fastest mirrors
4. Update package indicies:
  * `# pacman -Syyy`
5. Create the EFI partition:
  * `# fdisk /dev/sda`
    * g (creates an empty GPT partition table)
    * n (new)
    * 1 (1st partition)
    * [enter]
    * +300M (partition size)
    * t
    * 1 (for EFI)
    * w (commit to disk)
6. Create the boot partition:
  * `# fdisk /dev/sda`
    * n
    * 2 (should be default)
    * [enter]  (selects first sector)
    * +500M
    * w  (The default type should be 'Linux filesystem' and that is what we want)
7. Create the LVM partition:
  * `# fdisk /dev/sda`
    * n
    * 3
    * [enter]  (first sector should be good)
    * [enter]  (we want the rest of the disk
    * t
    * 3
    * 31   (LVM type)
    * w
8. Make the filesystem for /dev/sda1 and /dev/sda2    
  *  `# mkfs.fat -F32 /dev/sda1`
  *  `# mkfs.ext2 /dev/sda2`   (ext4 can be used too)
  *   Using `lsblk` is helpful here to confirm
9. Enable disk encryption
  * `# cryptsetup luksFormat /dev/sda3`
  * `# cryptsetup open --type luks /dev/sda3 lvm`
10. Setup LVM partition configuration:
  * `# pvcreate --dataalignment 1M /dev/mapper/lvm`
  * `# vgcreate volgroup0 /dev/mapper/lvm`
  * `# lvcreate -L 40GB volgroup0 -n lv_root`
  * `# lvcreate -L 450GB volgroup0 -n lv_home`
  * `# modprobe dm_mod`
  * `# vgscan`
  * `# vgchange -ay`
11. `# mkfs.ext4 /dev/volgroup0/lv_root`
0. `# mkfs.xfs /dev/volgroup0/lv_home`
0. `# mount /dev/volgroup0/lv_root /mnt`
0. `# mkdir /mnt/boot`
0. `# mkdir /mnt/home`
0. `# mount /dev/sda2 /mnt/boot`
0. `# mount /dev/volgroup0/lv_home /mnt/home`
0. `# pacstrap -i /mnt base` (All the default values should be fine)
0. `# genfstab -U -p /mnt >> /mnt/etc/fstab`
0. `# arch-chroot /mnt`
0. `# pacman -S base-devel grub efibootmgr dosfstools openssh os-prober mtools linux-headers vim`
0. Edit `/etc/mkinitcpio.conf` and add `encrypt lvm2` in between `block` and `filesystems`
within `HOOKS=(...)`
0. `# mkinitcpio -p linux`
0. `# vim /etc/locale.gen` (uncomment en_US.UTF-8)
0. `# locale-gen`
0. Enable `root` logon via `ssh`
  * `# systemctl enable sshd.service`
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

## Update locale information 
1. `# localectl set-locale LANG="en_US.UTF-8"`
0. `curl https://ipapi.co/timezone | timedatectl set-timezone`
0.  

## Network Stuff
1. Make sure you have a hostname in /etc/hostname
2. Install NetworkManager to help with starting network on startup
  *  `pacman -S NetworkManager`
  *  `systemctl enable NetworkManager`
3. If you don't:
  *  `systemctl restart dhcpcd`
  *  `ip link set ennnnn up`
  * https://wiki.archlinux.org/index.php/Network_configuration#Set_the_hostname

## Because Lease Privilage
0. Add a user!
  * useradd -m -g wheel <username>
  * passwd <username>
