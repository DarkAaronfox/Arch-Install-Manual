# Arch Linux Pre-installation Guide (with NVIDIA)

This is a step-by-step guide for a manual Arch Linux installation, based on a custom configuration including NVIDIA 

---

# Pre-installation
## 0. Keyboard layout and Timezone

### 0.1 Keymap

> List of the keymaps
```
localectl list-keymaps
```

> Example to set the Hungarian keyboard layout
```
loadkeys hu
```

### 0.2 Timezone (Optional)

> List of the timezones
```
timedatectl list-timezones
```

> Example to set the Europe/Budapest timezone
```
timedatectl set-timezone Europe/Budapest
```
## 1. Disk Partitioning
> Identify your drive using `lsblk` or `fdisk -l`. This guide assumes `/dev/sda`.

Use this to start editing the disk
```
fdisk /dev/sda
```
> Inside `fdisk` commands:

`g` - (Create a new GPT partition table.)

`n` -> `1` -> (Press Enter) -> `+1G` (EFI Partition)

`n` -> `2` -> (Press Enter) -> `+8G` (Swap Partition)

`n` -> `3` -> (Press Enter) -> (Press Enter) (Root Partition)

`t` -> `1` -> `EFI System`

`t` -> `2` -> `Linux Swap`

`w` - (Write changes and exit.)


## 2. Formatting partitions
```
mkfs.fat -F 32 /dev/sda1
mkswap /dev/sda2
mkfs.ext4 /dev/sda3
```

## 3. Mounting
```
mount /dev/sda3 /mnt
mount --mkdir /dev/sda1 /mnt/boot
swapon /dev/sda2
```

## 4. Mirrorlist Optimization

```
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
pacman -Sy pacman-contrib
rankmirrors -n 6 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist
```

## 5. Install Base System
```
pacstrap -K /mnt base linux linux-firmware base-devel
```

## 6. Configure Fstab
```
genfstab -U -p /mnt >> /mnt/etc/fstab
```

> Verify the output
```
nano /mnt/etc/fstab
```
> exit

## 7. System Configuration (Chroot)
```
arch-chroot /mnt
sudo pacman -S nano bash-completion
```

### 7.1 Localization

uncomment (#) the `en_US.UTF-8 UTF-8` line for the English - US language in
```
nano /etc/locale.gen
```

save, exit, and after (to generate the language)
```
locale-gen
```

to save your language
```
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8
```

to save your keyboard layout
```
echo KEYMAP=hu > /etc/vconsole.conf
export KEYMAP=hu
```

### 7.2 Time
```
ln -sf /usr/share/zoneinfo/Europe/Budapest /etc/localtime
hwclock --systohc --utc
```

### 7.3 Hostname
```
echo archlinux > /etc/hostname
```

### 7.4 Trim support for SSDs
```
systemctl enable fstrim.timer
```

### 7.5 Enable Multilib (32-bit support)

uncomment (#) the `[multilib]` section headers.
```
nano /etc/pacman.conf
```
save, exit and after
```
sudo pacman -Sy
```
### 7.6 Users and Security

Set Root Password
```
passwd
```

Create User 'aaron'

```
useradd -m -g users -G wheel,storage,power -s /bin/bash aaron
passwd aaron
```

Enable Sudo

```
EDITOR=nano visudo
```

Uncomment (#): `%wheel ALL=(ALL:ALL) ALL`, save, exit

### 7.7 Bootloader (systemd-boot)
```
mount -t efivarfs efivarfs /sys/firmware/efi/efivars/
bootctl install
```

Configure Entry

```
nano /boot/loader/entries/arch.conf
```
Plaintext
```
title Arch
linux /vmlinuz-linux
initrd /initramfs-linux.img
```
Append the Root UUID:
```
echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/sda3) rw" >> /boot/loader/entries/arch.conf
```

### 7.8 Networking
Network Services
```
sudo pacman -S dhcpcd
ip link (to check for the name of it, like enp0s3)
sudo systemctl enable dhcpcd@enp0s3.service
sudo pacman -S networkmanager
sudo systemctl enable NetworkManager.service

```

### 7.9 Graphics
NVIDIA Drivers
```
sudo pacman -S nvidia-dkms libglvnd nvidia-utils opencl-nvidia lib32-libglvnd lib32-nvidia-utils lib32-opencl-nvidia nvidia-settings linux-headers
```

Kernel Parameters (KMS):
```
sudo nano /etc/mkinitcpio.conf
```
edit the modules: 
```
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

save, exit

for the boot options:
```
sudo nano /boot/loader/entries/arch.conf
```

add `nvidia-drm.modeset=1` to a new line and after save, exit

NVIDIA Pacman Hook:

Create 
```
sudo mkdir /etc/pacman.d/hooks/nvidia.hook
```

Edit

```
sudo nano /etc/pacman.d/hooks/nvidia.hook
```

Write this in it

```
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia

[Action]
Depends=mkinitcpio
When=PostTransaction
Exec=/usr/bin/mkinitcpio -P
```
save, exit

## 8. Completion
```
exit
umount -R /mnt
reboot
```
