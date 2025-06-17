Based on material from the UNIXoid's Diary channel:
https://t.me/thmUNIX
https://youtube.com/@thmUNIX

Tutorial version: 1.1
## Step 0 - Change the font
To make the commands easier to see:
```sh
setfont /usr/share/kbd/consolefonts/sun12x22.psfu.gz
```

## Step 1 - Connect to the Internet
- *Ethernet* → nothing needs to be done
- *Wi-Fi* → enter the **iwctl** command
```sh
station list
station adapter get-networks
station adapter connect “SSID_неты”
# (wait 10-15 seconds)
quit
```
If any problem occurs, try running the command `rfkill unblock all` and try connecting to Wi-Fi again

## Step 2 - Configuring the package manager
- nano /etc/pacman.d/mirrorlist
Check that Reflector has generated a list of mirrors. If not, comment out all the mirrors and uncomment the ones we want to use.
- If there is no mirror list, you can generate it:
```sh
sudo reflector --country "Russia" --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
```
- nano /etc/pacman.conf
Uncomment the `ParallelDownloads` parameter and set the desired value (for example, 15)

## Step 3 - Disk partitioning
- Look at the partition names and find the desired disk
```sh
fdisk -l
lsblk
```
- Partition the desired disk (for example, `/dev/nvme0n1`)
```sh
cfdisk disk
```
>- Even if the disk partitioning already exists, it must be deleted (except `/home`) and a new one created
>- For partitions, select `Delete`, `New`, `Type` (select from the table). At the end, select the `Write` item
>- Mount points will be used later

| # | Type | Size | Mount point |
| --- | ---------------- | ------------------- | ------------------ |
| 1 | EFI system | 256M | /boot/efi |
| 2 | Linux filesystem | All free space | / |
or

| # | Type | Size | Mount point |
| --- | ---------------- | ------------------- | ------------------ |
| 1 | EFI system | 256M | /boot/efi |
| 2 | Linux filesystem | 25G ~ 30G | / |
| 3 | Linux filesystem | All free space | /home |

## Step 4 - Format partitions
```sh
mkfs.vfat /dev/disk1 # p1 efi
mkfs.ext4 /dev/disk2 # p2 linux
# if /home was moved to a separate partition:
mkfs.ext4 /dev/disk3 # p3 linux
```

## Step 5 - Mount partitions
```sh
mount /dev/disk2 /mnt # p2 linux
mkdir -p /mnt/boot/efi
mount /dev/disk1 /mnt/boot/efi # p1 efi

# if /home was moved to a separate partition:
mkdir -p /mnt/home
mount /dev/disk3 /mnt/home # p3 linux
```

## Step 6 - Install packages
- Example command (see below to choose what you need)
```
pacstrap /mnt base base-devel linux linux-firmware linux-headers nano vim bash-completion grub efibootmgr xorg ttf-ubuntu-font-family ttf-hack ttf-dejavu ttf-opensans lxdm xfce
```
`pacstrap` allows you to choose where to install packages
- You can install packages in parts if they do not install well in the live cd. Some packages can be installed after `arch-chroot` using regular `pacman`:
```sh
arch-chroot /mnt
pacman -S packages
exit
umount -R /mnt
```
- Packages
Minimum required
```
base base-devel linux linux-firmware linux-headers bash-completion grub efibootmgr networkmanager network-manager-applet
```
Text console editors (you can choose one or more)
```
nano vim neovim
```
Xorg display server
```
xorg
```
Fonts
```
ttf-ubuntu-font-family ttf-hack ttf-dejavu ttf-opensans
```
Display manager (choose ONE)
```
sddm / lightdm / lxdm / gdm
```
Desktop environment (select ONE)
```
gnome / plasma / cinnamon /
budgie / xfce4 / lxqt / lxde / ...
```
NVIDIA proprietary driver
```
nvidia
```

## Step 7 - Generate fstab
```sh
genfstab /mnt >> /mnt/etc/fstab
```

## Step 8 - Change system root
```sh
arch-chroot /mnt
```

## Step 9 - Enable services
- Enable NetworkManager
```sh
systemctl enable NetworkManager
```
- Enable display manager
```sh
systemctl enable DM_name
# sddm / lxdm / gdm / ...
```

## Step 10 - Users & passwords
- Create user
```
useradd -m username
```
- Set a password for the user
```
passwd username
```
- Set a password for the root user
```
passwd
```
- Give yourself sudo rights
Recommended (if you know how to use vim):
```
visudo
```
Edit directly:
```
nano /etc/sudoers
```
Add:
```
root ALL=(ALL:ALL) ALL
username ALL=(ALL:ALL) ALL
```

## Step 11 - Locales & system language
- nano /etc/locale.gen
Uncomment the locales you need. For example:
```
en_US.UTF-8 UTF-8
ru_RU.UTF-8 UTF-8
```
- nano /etc/locale.conf
Enter the line:
```
LANG= ”locale”
```
Specify the desired system language. For example, **”en_US.UTF-8”**
- Generate uncommented locales
```sh
locale-gen
```

## Step 12 - Install bootloader
- Install bootloader on disk
```sh
grub-install /dev/disk # /dev/nvme0n1
```
- **nano /etc/default/grub**
Remove the `quiet` parameter from the `GRUB_CMDLINE_LINUX_DEFAULT` parameter - this is optional
- Save the bootloader config
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

## Step 13 - Reboot to the new system
- Go back to the Live CD shell
```sh
exit
```
- Unmount all disk partitions
```sh
umount -R /mnt
```
- Reboot
```sh
reboot
```

## Step 14 - Connect to the Internet
- Ethernet → nothing needs to be done
- Wi-Fi →
If DE has a GUI for connecting to Wi-Fi, then connect as usual
If not, then use **nmtui**
- Select `Radio`, check that Wi-Fi is enabled. If not, enable it.
- Go back, select `Activate a connection`
- Select `AP`, enter the password, connect
- Exit

## Step 15 - Setting up date & time
- Set the time zone
```sh
sudo timedatectl set-timezone time_zone
# For example, Europe/Moscow
```
Time zones are in `/usr/share/zoneinfo`
- Enable NTP (network time)
```sh
sudo timedatectl set-ntp true
```

## Step 16 - Create a swap file
For example, 6G. It is recommended to create two times smaller than the size of RAM
```sh
sudo mkswap -U clear --size size --file /swapfile
sudo swapon /swapfile
```
- **nano /etc/fstab**
Add to the end (separate columns with Tab):
```
/swapfile none swap defaults 0 0
```
- Everything is ready, reboot
```sh
reboot
```
