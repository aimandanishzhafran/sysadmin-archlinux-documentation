# Arch Linux VM Installation 
**Author:** Aiman Danish Zhafran Akmal Hizam  
**System:** Windows 11 (VMware Workstation Pro)  
**Hostname:** Aiman-ArchVM  

A simple guide to recreate my customized Arch Linux virtual machine.

---

## Step 1: ISO Download & Verification

1. Go to the official [Arch Download Page](https://archlinux.org).
2. Scroll to your country and download the latest `.iso` file.
3. Open Windows PowerShell and verify the file integrity:
```powershell
Get-FileHash "$HOME\Downloads\archlinux-2026.11.01-x86_64.iso" -Algorithm SHA256
```
*Make sure the output hash matches the hash listed on the Arch website.*

---

## Step 2: VMware Virtual Machine Setup

1. Open VMware Workstation and click **New Virtual Machine**.
2. Choose **Typical**, click Next.
3. Select **Installer disc image file (iso)** and browse to your downloaded Arch ISO.
4. Set OS to **Linux** and Version to **Other Linux 6.x kernel 64-bit**.
5. Name the VM (e.g: `Aiman-ArchVM`).
6. Set Disk Capacity to **30 GB** and select **Store virtual disk as a single file**.
7. Click **Customize Hardware**, change Memory to **2 GB (2048 MB)**, click Close, then Finish.
8. Power on the VM to boot into the Arch Live environment.

---

## Step 3: Pre-Installation Checks

Run these commands inside the live terminal to check your environment:
```bash
# Verify UEFI mode (Must display files without errors)
ls /sys/firmware/efi/efivars
# If fail, then shut down the VM and change the VM settings from BIOS to UEFI in VMware

# Test network connection
ping -c 3 archlinux.org

# Set timezone and synchronize the system clock
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
hwclock --systohc
timedatectl set-ntp true
```

---

## Step 4: Disk Partitioning & Formatting

Run `cfdisk /dev/sda` to partition the disk. Select **gpt**, then create this layout:
* `/dev/sda1` -> 500 MB (Type: EFI System)
* `/dev/sda2` -> Remaining space (Type: Linux filesystem)

```bash
# Format the EFI partition to FAT32
mkfs.fat -F 32 /dev/sda1

# Format the Root partition to Ext4
mkfs.ext4 /dev/sda2

# Mount the partitions
mount /dev/sda2 /mnt
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```

---

## Step 5: Base System Installation

```bash
# Install core packages, editors, network tools, and sudo privileges
pacstrap /mnt base linux linux-firmware nano vim networkmanager sudo

# Generate the storage table map (fstab)
genfstab -U /mnt >> /mnt/etc/fstab

# Log into the newly installed system as root (Chroot jail)
arch-chroot /mnt
```

---

## Step 6: System Configuration

```bash
# Set up locale and system language
# (Remember to uncomment 'en_US.UTF-8 UTF-8' in /etc/locale.gen first using the `nano` command)
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Set system network identity
echo "Aiman-ArchVM" > /etc/hostname

# Enable the network manager daemon to start on boot
systemctl enable NetworkManager

# Create a password for the root user
passwd
```

---

## Step 7: Users & Privileges 

```bash
# Create users and assign a password
useradd -m -G wheel -s /bin/bash aiman
passwd aiman

useradd -m -G wheel -s /bin/bash codi
passwd codi

# wheel is the sudo group
```
Run `EDITOR=nano visudo` and uncomment this line to give the `wheel` group admin permissions:
```text
%wheel ALL=(ALL:ALL) ALL
```

---

## Step 8: Shell, SSH, & Customizations 

```bash
# Install Zsh, completions, and OpenSSH
pacman -S zsh zsh-completions openssh --noconfirm

# Set Zsh as the default shell for created users
chsh -s /bin/zsh aiman
chsh -s /bin/zsh codi

# Enable SSH service to start on boot
systemctl enable sshd
```

### Pacman Visuals 
Open `/etc/pacman.conf` with the `nano` command and uncomment or add these two lines under `[options]`:
```text
Color
ILoveCandy
```

### Custom Aliases 
Add these shortcuts to the end of both `/home/aiman/.zshrc` and `/home/codi/.zshrc` using the `nano` command:
```bash
alias ls='ls --color=auto'
alias grep='grep --color=auto'
alias ..='cd ..'
alias la='ls -lah'
alias rm='rm -i'
alias update='sudo pacman -Syu'
```

---

## Step 9: Desktop Environment & Login GUI 

```bash
# Install Xorg, LXQt desktop environment, terminal, icons, and SDDM login manager
pacman -S xorg-server lxqt qterminal oxygen-icons sddm --noconfirm

# Enable the graphical login manager to launch on boot
systemctl enable sddm

# Force the system to boot directly into the GUI instead of the terminal
systemctl set-default graphical.target
```

---

## Step 10: Bootloader & Restart

```bash
# Install systemd-boot to the EFI partition
bootctl install
```

Create and configure the boot entry file `/boot/loader/entries/arch.conf`:
```text
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options root=/dev/sda2 rw
```

Exit the environment and reboot the machine:
```bash
exit
umount -R /mnt
reboot
```

---

## Troubleshooting & Lessons Learned

* **Empty Partition Table:** Inside `cfdisk`, I forgot to select **[ Write ]** and type `yes` before hitting **[ Quit ]**. This made `/dev/sda` look blank to `lsblk`. Fix: Re-opened `cfdisk`, wrote the changes properly, and formatting worked fine.
* **Missing Package Repositories:** When booting into a freshly installed system, `pacman` database caches are empty. Fix: Always run `sudo pacman -Syu` before trying to install packages like `zsh` or `lxqt`.
* **Typos:** Made small syntax errors by mispelling `firmware` as `firmare`, or trying to use `systemd` commands directly instead of `systemctl`. Fix: Double-check spellings against the official Arch Wiki guide.
* **Syntax:** Capitalization and whitespace are crucial.
