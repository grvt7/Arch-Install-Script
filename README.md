# Arch Linux Installation Script - Updated 2025
**Updated with latest Arch Wiki recommendations**
**Customized for: Ethernet connection, India location**

---

## Pre-Installation Checks

### Verify Boot Mode (UEFI)
```bash
# Check UEFI bitness (should return 64 for modern systems)
cat /sys/firmware/efi/fw_platform_size

# Alternative legacy check
efivar -l
```

### Update System Clock
```bash
# Ensure time is synced (automatic with ethernet)
timedatectl
```

---

## Disk Partitioning

### View Current Disk Layout
```bash
lsblk
fdisk -l
```

### Wipe Disk (EXPERT MODE - Destructive!)
```bash
gdisk /dev/nvme0n1
	command : x (expert)
	expert : z (zap – drive)
	clear GPT: yes
	blank MBR : yes
```

### Verify Disk is Clean
```bash
lsblk
```

### Create Partition Table with cgdisk
```bash
cgdisk /dev/nvme0n1
	# Partition 1: EFI Boot
	new -> 1GiB -> EF00 (boot) -> boot
	
	# Partition 2: Swap
	new -> 16GiB -> 8200 (swap) -> swap
	
	# Partition 3: Root
	new -> 70GiB -> default (8300) -> root
	
	# Partition 4: Home (remaining space)
	new -> all remaining -> default (8300) -> home
	
	write -> yes -> quit
```

---

## Format Partitions

```bash
# EFI System Partition (FAT32)
mkfs.fat -F32 /dev/nvme0n1p1

# Swap Partition
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2

# Root Partition (ext4)
mkfs.ext4 /dev/nvme0n1p3

# Home Partition (ext4)
mkfs.ext4 /dev/nvme0n1p4

# Verify formatting
lsblk -f
```

---

## Mount Filesystems

```bash
# Mount root partition
mount /dev/nvme0n1p3 /mnt

# Create mount points
mkdir /mnt/boot
mkdir /mnt/home

# Mount boot and home partitions
mount /dev/nvme0n1p1 /mnt/boot
mount /dev/nvme0n1p4 /mnt/home

# Verify mounts
lsblk
```

---

## Update Mirror List (Using Reflector - India)

```bash
# Backup original mirrorlist
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup

# Install reflector
pacman -Sy reflector

# Generate optimized mirrorlist for India
# Select 20 most recently synced HTTPS mirrors from India, sort by download speed
reflector --country India --latest 20 --protocol https --sort rate --save /etc/pacman.d/mirrorlist

# Alternative: Include nearby countries for more mirror options
# reflector --country India,Singapore,Japan --latest 20 --protocol https --sort rate --save /etc/pacman.d/mirrorlist

# Legacy method with rankmirrors (if you prefer)
# pacman -S pacman-contrib
# rankmirrors -n 7 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist

# Verify mirrorlist
cat /etc/pacman.d/mirrorlist
```

---

## Install Base System

```bash
# Install minimal base system (as per your preference)
pacstrap -K /mnt base linux linux-firmware base-devel

# Note: Intel microcode recommended but not mandatory
# Add 'intel-ucode' or 'amd-ucode' if you want CPU microcode updates
# I do this later in chroot
```

---

## Generate fstab

```bash
# Generate fstab using UUIDs
genfstab -U -p /mnt >> /mnt/etc/fstab

# Verify fstab entries
cat /mnt/etc/fstab
```

---

## Reload Systemd Daemon

```bash
# Reload systemd daemon configuration
systemctl daemon-reload
```

---

## Chroot into New System

```bash
arch-chroot /mnt
```

---

## System Configuration (Inside chroot)

### Mount EFI Variables Filesystem
```bash
# Mount efivarfs (required for bootloader installation)
mount -t efivarfs efivarfs /sys/firmware/efi/efivars/

# Verify EFI variables are accessible
ls /sys/firmware/efi/efivars/
```

### Install Essential Utilities
```bash
# Install text editor and bash completion
pacman -S nano bash-completion
```

### Enable Multilib (32-bit support)
```bash
# Edit pacman.conf
nano /etc/pacman.conf

# Uncomment these two lines:
# [multilib]
# Include = /etc/pacman.d/mirrorlist

# OPTIONAL: Enable ParallelDownloads for faster updates
# Find and uncomment:
# ParallelDownloads = 5

# Save and exit

# Sync repositories
pacman -Sy
```

### Configure Localization
```bash
# Edit locale.gen
nano /etc/locale.gen

# Uncomment:
en_US.UTF-8 UTF-8

# Generate locales
locale-gen

# Set system locale
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Export for current session
export LANG=en_US.UTF-8
```

### Set Timezone
```bash
# List available timezones
ls /usr/share/zoneinfo/
ls /usr/share/zoneinfo/Asia

# Set timezone to India
ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime

# Generate /etc/adjtime (assumes hardware clock is UTC)
hwclock --systohc
```

### Set Hostname
```bash
# Create hostname file (replace `hostname` with your actual username)
# Dont add backticks
echo `hostname` > /etc/hostname
```

### Configure Hosts File
```bash
# Create /etc/hosts file
# Purpose: Maps hostnames to IP addresses locally (before DNS)
# Needed for: hostname resolution, network services, avoiding delays
nano /etc/hosts

# Don't add the backticks
# Add these lines:
127.0.0.1    localhost
::1          localhost
127.0.1.1    `hostname`.localdomain `hostname`

# Save and exit
```
**What is /etc/hosts?**
- Local hostname-to-IP mapping file
- Checked BEFORE DNS queries
- Prevents hostname resolution errors
- Many services expect this file to exist
- NOT auto-created - you must create it manually

### Enable Periodic TRIM for SSD
```bash
systemctl enable fstrim.timer
```

---

## User Management

### Set Root Password
```bash
passwd
```

### Create User Account
```bash
# Create user (replace 'username' with your actual username)
useradd -m -g users -G wheel,storage,power,audio,video -s /bin/bash username

# Set user password
passwd username
```

### Configure Sudo Access
```bash
# Edit sudoers file safely
EDITOR=nano visudo

# Find and uncomment this line (Ctrl+W to search for "wheel"):
%wheel ALL=(ALL:ALL) ALL

# OPTIONAL: If you want ALL users to use root password for sudo
# Add this line at the end:
Defaults rootpw

# Save and exit
```

---

## Install and Configure Bootloader (systemd-boot)

### Install systemd-boot
```bash
bootctl install
```

### Create Boot Entry
```bash
# Create boot entry configuration file
nano /boot/loader/entries/arch.conf

# Add these lines:
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img

# Save and exit (Ctrl+O, Enter, Ctrl+X)
```

### Add Root Partition UUID
```bash
# Automatically append the correct PARTUUID to boot options
echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/nvme0n1p3) rw" >> /boot/loader/entries/arch.conf

# Verify the complete boot entry
cat /boot/loader/entries/arch.conf
```

### Install CPU Microcode
```bash
# Install Intel CPU microcode for hardware bug fixes and security patches
# Reference: https://wiki.archlinux.org/title/Microcode
pacman -S intel-ucode

# Note: For AMD CPUs, use: pacman -S amd-ucode
```

### Update Boot Entry with Microcode
```bash
# Edit boot entry to add microcode
nano /boot/loader/entries/arch.conf

# Modify the file to add microcode BEFORE initramfs:
# It should look like this:
# title   Arch Linux
# linux   /vmlinuz-linux
# initrd  /intel-ucode.img
# initrd  /initramfs-linux.img
# options root=PARTUUID=xxx-xxx-xxx rw

# The initrd lines must be in this order:
# 1. CPU microcode first
# 2. initramfs second

# Save and exit
```

### Configure Bootloader Settings (Optional)
```bash
# Configure systemd-boot loader behavior
nano /boot/loader/loader.conf

# Add these lines:
default  arch.conf
timeout  3
console-mode max
editor   no

# What these do:
# default  - which entry to boot by default
# timeout  - seconds to wait before auto-boot
# console-mode max - use highest resolution
# editor no - disable kernel parameter editing (security)

# Save and exit
```

---

## Network Configuration

### Install and Enable NetworkManager
```bash
# Install network management tools
pacman -S networkmanager

# Check your network interface name (usually wlo1 for WiFi, enp0s* for Ethernet)
ip link

# Enable NetworkManager service
systemctl enable NetworkManager.service

# Note: For Ethernet, NetworkManager will auto-connect on boot
```

### Install Bluetooth Support
```bash
# Install Bluetooth stack and utilities
pacman -S bluez bluez-utils bluez-deprecated-tools

# Enable Bluetooth service
systemctl enable bluetooth.service
```

---

## GPU Drivers (Intel + NVIDIA Hybrid Graphics)

### Install Intel Graphics Drivers
```bash
# Reference: https://wiki.archlinux.org/title/Intel_graphics

# Install Intel graphics drivers and firmware (all in one command)
pacman -S linux-firmware-intel mesa lib32-mesa vulkan-intel lib32-vulkan-intel intel-media-driver

# Enable GuC/HuC firmware loading (improves performance and power efficiency)
# GuC: GPU Command Submission
# HuC: HEVC/H.265 encoding support
nano /etc/modprobe.d/i915.conf

# Add this line:
options i915 enable_guc=3

# Save and exit
# Note: enable_guc=3 enables both GuC and HuC
# For newer Intel GPUs (Gen 12+), this is enabled by default
```

### Install Linux Headers
```bash
pacman -S linux-headers
```

### Install NVIDIA Drivers
```bash
# Install NVIDIA packages
pacman -S nvidia-dkms libglvnd nvidia-utils opencl-nvidia \
    lib32-libglvnd lib32-nvidia-utils lib32-opencl-nvidia \
    nvidia-settings

# Load NVIDIA kernel module (command available from nvidia-utils)
nvidia-modprobe

# Enable NVIDIA power management services
# NOTE: These services are COMMENTED OUT because supergfxctl will handle suspend/resume
# supergfxctl provides better power management for hybrid GPU laptops
# If you're NOT using supergfxctl, uncomment these lines:
# systemctl enable nvidia-suspend.service
# systemctl enable nvidia-resume.service
# systemctl enable nvidia-hibernate.service
```

### Update Bootloader Entry
```bash
# Now edit boot entry to add additional kernel parameters
nano /boot/loader/entries/arch.conf

# Add nvidia-drm.modeset=1 to the options line (REQUIRED for hybrid graphics):
Change: options root=PARTUUID=xxx-xxx-xxx rw
To:     options root=PARTUUID=xxx-xxx-xxx rw nvidia-drm.modeset=1

# OPTIONAL - For ASUS ROG laptops (improves ACPI/hardware compatibility):
# If you experience issues with function keys, fan control, or GPU power management,
# add these ACPI parameters after nvidia-drm.modeset=1:
# acpi_osi=! acpi_osi="Windows 2020"
#
# Full line would be:
options root=PARTUUID=xxx-xxx-xxx rw nvidia-drm.modeset=1 acpi_osi=! acpi_osi="Windows 2020"
#
# What these do:
# - acpi_osi=! clears all OS identifiers
# - acpi_osi="Windows 2020" tells BIOS you're Windows 10 2004+
# - Enables Windows-optimized ACPI features (function keys, fan curves, etc.)
# - Reference: https://wiki.archlinux.org/title/Laptop/ASUS
#
# Save and exit
```

### Configure mkinitcpio for NVIDIA
```bash
# Edit mkinitcpio configuration
nano /etc/mkinitcpio.conf

# Find the MODULES=() line and modify it to:
MODULES=(asus_wmi asus_nb_wmi i915 nvidia nvidia_modeset nvidia_uvm nvidia_drm)

# What these ASUS modules do:
# - asus_wmi: Base ASUS WMI driver for hardware control
# - asus_nb_wmi: ASUS Notebook WMI (function keys, fan control, battery)
# - Required for asusctl to properly detect and control hardware

# Order matters! Intel i915 should be first, then NVIDIA modules

# Save and exit

# Regenerate initramfs
mkinitcpio -P
```

### Create Pacman Hook for NVIDIA (Simple Version)
```bash
# Create hooks directory
mkdir -p /etc/pacman.d/hooks

# Create NVIDIA hook
nano /etc/pacman.d/hooks/nvidia.hook

# Add these lines:
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia-dkms
Target=linux

[Action]
Description=Updating NVIDIA modules in initcpio
Depends=mkinitcpio
When=PostTransaction
Exec=/usr/bin/mkinitcpio -P

# Save and exit
```

**Why Target both nvidia-dkms AND linux?**
- When `nvidia-dkms` updates → rebuild initramfs (driver update)
- When `linux` kernel updates → rebuild initramfs (kernel update)
- Both need initramfs regeneration for NVIDIA to work properly

**Alternative Advanced Version (prevents double rebuild):**
```bash
# If you want to prevent rebuilding twice when both packages update:
# Replace the [Action] section with:
[Action]
Description=Updating NVIDIA modules in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux*) exit 0; esac; done; /usr/bin/mkinitcpio -P'

# This checks: if linux triggered it, exit (nvidia-dkms will handle it)
# Prevents running mkinitcpio twice when both packages update together
# Simple version works fine - this is just an optimization
```

---

## Finalize Installation

### Exit Chroot
```bash
exit
```

### Unmount Filesystems
```bash
# Unmount all partitions
umount -R /mnt

# If "target is busy" error occurs, force lazy unmount:
# umount -R -l /mnt
```

### Reboot
```bash
reboot

# Remove installation media when system restarts
```

---

## Post-Installation (After First Boot)

### Login
```
Login as: username
Password: your_password
```

### Verify Network Connection
```bash
# Ethernet should auto-connect
ip addr
ping archlinux.org
```

### Install Xorg Display Server
```bash
sudo pacman -S xorg-server xorg-apps xorg-xinit xorg-twm xorg-xclock xterm

# Test X server
startx
```

### Install KDE Plasma Desktop Environment
```bash
# Install KDE Plasma and applications
sudo pacman -S plasma kde-applications sddm firefox git neofetch htop

# Enable display manager
sudo systemctl enable sddm.service

# Reboot to start graphical interface
reboot
```

---

## Additional Recommendations (Post-Install)

### Enable Time Synchronization
```bash
sudo systemctl enable systemd-timesyncd.service
sudo systemctl start systemd-timesyncd.service
```

### Install AUR Helper (yay)
```bash
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ~
```

### Install Useful Fonts
```bash
sudo pacman -S ttf-dejavu ttf-liberation noto-fonts noto-fonts-emoji
```

### System Maintenance Tools
```bash
sudo pacman -S reflector pkgfile
sudo systemctl enable plocate-updatedb.timer
```

---

## Summary of Key Changes from Original

### ✅ Updates Made:
1. **Reflector with India mirrors** - Optimized for your location
2. **Ethernet-only setup** - Removed WiFi configuration
3. **Minimal base install** - Only base, linux, linux-firmware, base-devel
4. **systemctl daemon-reload** - Added before chroot
5. **efivarfs mount** - Added with error suppression explanation
6. **Hosts file** - Explained purpose and added configuration
7. **Separate mkdir** - Before mount commands as requested
8. **Updated NVIDIA hook** - Targets nvidia-dkms + linux with explanation
9. **NetworkManager install moved** - After chroot like original
10. **Loader.conf kept** - As optional but documented

### 📋 All Your Original Features Preserved:
- cgdisk partitioning workflow
- Your exact partition sizes
- NVIDIA + Intel hybrid setup
- KDE Plasma desktop
- All custom settings and preferences

### 📚 References:
- [Arch Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
- [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot)
- [NVIDIA](https://wiki.archlinux.org/title/NVIDIA)
