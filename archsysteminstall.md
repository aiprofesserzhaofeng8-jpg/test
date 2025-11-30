# Arch Linux Installation Guide (VirtualBox / UEFI)

## 1\. Preparation: Download ISO

  * **Download:** [Arch Linux ISO](https://archlinux.org/download/)
  * **Reference:** [ArchWiki (CN)](https://wiki.archlinuxcn.org/)

-----

## 2\. Host Environment Setup (Ubuntu)

*(Optional: Only run this if you need to install VirtualBox on your Ubuntu host)*

### Restore Ubuntu Official Sources & Install VirtualBox

If you need to fix your apt sources to install the official VirtualBox version:

```bash
# Backup existing sources
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup

# Overwrite with Ubuntu Noble (24.04) official sources
sudo tee /etc/apt/sources.list > /dev/null <<EOF
deb http://archive.ubuntu.com/ubuntu/ noble main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu/ noble-updates main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu/ noble-backports main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu noble-security main restricted universe multiverse
EOF

# Update package list
sudo apt update
```

### Install VirtualBox Extension Pack

```bash
# 1. Remove repository version if conflicting
sudo apt remove --purge virtualbox-ext-pack

# 2. Download official Extension Pack (Ensure version matches your VirtualBox, e.g., 7.0.16)
wget https://download.virtualbox.org/virtualbox/7.0.16/Oracle_VM_VirtualBox_Extension_Pack-7.0.16.vbox-extpack

# 3. Install via VBoxManage
sudo VBoxManage extpack install Oracle_VM_VirtualBox_Extension_Pack-7.0.16.vbox-extpack

# 4. Cleanup
rm Oracle_VM_VirtualBox_Extension_Pack-7.0.16.vbox-extpack
```

-----

## 3\. Create & Configure Virtual Machine

### Create VM (GUI or CLI)

Create a new Virtual Machine in VirtualBox.

  * **Type:** Linux
  * **Version:** Arch Linux (64-bit)

### Attach ISO (Command Line method)

```bash
# Assuming VM name is "Arch Linux" and ISO is at ~/arch.iso
VBoxManage storageattach "Arch Linux" \
    --storagectl "IDE" \
    --port 0 \
    --device 0 \
    --type dvddrive \
    --medium ~/arch.iso
```

### Essential Settings for UEFI

Go to **Settings** -\> **System**:

1.  **Motherboard:** Check **Enable EFI (special OSes only)**.
2.  **Processor:** Check **Enable PAE/NX**.

-----

## 4\. Arch Linux Installation Steps

### 4.1 Partitioning (UEFI/GPT)

Verify the disk:

```bash
fdisk -l
```

Enter `fdisk` to partition `/dev/sda`:

```bash
fdisk /dev/sda
```

**fdisk Interactive Commands:**

1.  `g` : Create a new **GPT** disk label (Required for UEFI).
2.  `n` : Create new partitions.

| Partition | Type Code (in fdisk) | Size | Mount Point | Description |
| :--- | :--- | :--- | :--- | :--- |
| `/dev/sda1` | `1` (EFI System) | 1G | `/boot` | EFI Boot partition |
| `/dev/sda2` | `19` (Linux Swap) | 4G+ | `[SWAP]` | Swap partition |
| `/dev/sda3` | `20` (Linux Filesystem)| Rest | `/` | Root partition |

3.  `w` : Write changes and exit.

### 4.2 Formatting

Format the partitions created above:

```bash
# Root Partition (XFS)
mkfs.xfs /dev/sda3

# EFI System Partition (FAT32 is mandatory)
mkfs.fat -F 32 /dev/sda1

# Swap Partition
mkswap /dev/sda2
```

### 4.3 Mounting

Mount partitions to prepare for installation. **Order matters.**

```bash
# 1. Mount Root
mount /dev/sda3 /mnt

# 2. Create boot directory and mount EFI partition
mount --mkdir /dev/sda1 /mnt/boot

# 3. Enable Swap
swapon /dev/sda2

# Check mount status
df -h
free -h
```

### 4.4 Select Mirror (China)

Optimize download speed by selecting a local mirror.

```bash
curl -L 'https://archlinux.org/mirrorlist/?country=CN&protocol=https' -o /etc/pacman.d/mirrorlist
# Uncomment your preferred server (e.g., USTC, TUNA, 163)
vim /etc/pacman.d/mirrorlist
```

### 4.5 Install Base Packages

Install the kernel, firmware, and essential tools.

```bash
pacstrap -K /mnt base linux linux-firmware intel-ucode vim vi nano wget base-devel zip unzip man-db man-pages htop openssh git p7zip gzip xfsprogs
```

*Note: Added `xfsprogs` because you formatted root as XFS.*

### 4.6 Generate Fstab

**Important:** Do this *before* entering chroot.

```bash
# Generate filesystem table
genfstab -U /mnt > /mnt/etc/fstab

# Verify content
cat /mnt/etc/fstab
```

-----

## 5\. System Configuration (Chroot)

Enter the new system environment:

```bash
arch-chroot /mnt
```

### 5.1 Time Zone & Clock

```bash
# Set Timezone to Shanghai
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# Sync hardware clock
hwclock --systohc
```

### 5.2 Localization

1.  **Edit `locale.gen`:**
    ```bash
    vim /etc/locale.gen
    # Uncomment: en_US.UTF-8 UTF-8
    # Uncomment: zh_CN.UTF-8 UTF-8
    ```
2.  **Generate locales:**
    ```bash
    locale-gen
    ```
3.  **Set system language:**
    ```bash
    echo "LANG=en_US.UTF-8" > /etc/locale.conf
    ```
    *Tip: Avoid setting Chinese as system language in the console (TTY) to prevent display errors (squares).*

### 5.3 Hostname & Network

```bash
echo "myarch" > /etc/hostname
```

### 5.4 Set Root Password

```bash
passwd
```

### 5.5 Install & Configure Bootloader (GRUB)

```bash
# Install GRUB and EFI boot manager
pacman -S grub efibootmgr

# Install GRUB to EFI partition
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# Generate main configuration file
grub-mkconfig -o /boot/grub/grub.cfg
```

-----

## 6\. Finish Installation

```bash
# 1. Exit chroot environment
exit

# 2. Unmount all partitions
umount -R /mnt

# 3. Reboot system
reboot
```

*Remember to remove the installation ISO from VirtualBox storage settings after rebooting.*

-----

## 7\. Troubleshooting

### How to Reset Forgotten Root Password

If you forget your password, follow these steps:

1.  Restart the system.
2.  In the GRUB boot menu, select "Arch Linux" and press `e` to edit.
3.  Find the line starting with `linux`. append `init=/bin/bash` (or `/bin/sh`) at the end of that line.
4.  Press `Ctrl + x` to boot.
5.  Remount the root filesystem as Read/Write:
    ```bash
    mount -n -o remount,rw /
    ```
6.  Change password:
    ```bash
    passwd
    ```
7.  Reboot:
    ```bash
    reboot -f
    ```
