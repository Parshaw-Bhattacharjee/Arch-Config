# Arch Linux: Install & Configure
A step-by-step guide for installing and configuring Arch Linux with Hyprland as the desktop environment.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation Procedure](#installation-procedure)
- [System Configuration](#system-configuration)
- [Post-Installation & Configuration](#post-installation--configuration)

---

## Prerequisites

### 1. Installation Image

Download the latest Arch Linux ISO from the official site:
[Download Arch Linux ](https://archlinux.org/download/)

### 2. Verify Signature

Verify the integrity of the downloaded ISO before proceeding:

| Operating System | Command |
|-----------------|---------|
| Windows | `certutil -hashfile <file_path> SHA256` |
| Linux | `sha256sum <file_path>` |

Compare the output with the official SHA256 signature:
[Arch Linux SHA256 sums](https://archlinux.org/iso/2025.12.01/sha256sums.txt)

### 3. Installation Medium

Flash the ISO to a USB drive using one of the following tools:
- [Rufus](https://rufus.ie/en/)
- [Ventoy](https://www.ventoy.net/en/index.html)

> **Reference:** [Arch Linux Official Installation Guide](https://wiki.archlinux.org/title/Installation_guide)

---

## Installation Procedure

### 1. Connect to Network

```bash
$ iwctl
> station list
> station wlan0 connect <ssid>
# Enter passphrase
> CTRL + D
$ ping -c 5 archlinux.org
```

### 2. Update System Clock

```bash
$ timedatectl status
$ timedatectl set-timezone Asia/Kolkata
$ timedatectl set-ntp true
```

### 3. Disk Partitioning

```bash
$ lsblk

# find the drive for installation
$ $ DISK="/dev/nvme0n1"

# Wipe the NVMe drive
$ nvme format --force ${DISK} -s 1 -n 0xffffffff
$ nvme sanitize ${DISK} -a 2

# Partition with gdisk
$ gdisk ${DISK}
# o          → create new GPT partition table
# n          → new partition (+1G → ef00 for EFI, +32G → 8304 for root, ALL → 8302 for home)
# p          → print partition table
# w          → write and exit
```

### 4. Format Partitions

```bash
$ mkfs.fat -F 32 -n EFI ${DISK}p1   # EFI partition
$ mkfs.btrfs -L root -f ${DISK}p2   # root partition
$ mkfs.btrfs -L home -f ${DISK}p3   # home partition
```

### 5. Mount Partitions

```bash
$ mount ${DISK}p2 /mnt
$ btrfs subvolume create /mnt/@
$ btrfs subvolume create /mnt/@log
$ btrfs subvolume create /mnt/@cache
$ umount /mnt

$ mount ${DISK}p3 /mnt
$ btrfs subvolume create /mnt/@home
$ umount /mnt

$ OPTS="defaults,noatime,nodiratime,compress=zstd:1,ssd,discard=async,space_cache=v2"

$ mount -t btrfs -o ${OPTS},subvol=@ ${DISK}p2 /mnt
$ mkdir -p /mnt/{boot,home,var/log,var/cache}
mount -o ${OPTS},subvol=@log   ${DISK}p2 /mnt/var/log
mount -o ${OPTS},subvol=@cache ${DISK}p2 /mnt/var/cache

$ mount -t btrfs -o ${OPTS},subvol=@home /dev/nvme0n1p3 /mnt/home

$ mount -t vfat -o defaults,noatime,nodiratime,umask=0077 /dev/nvme0n1p1 /mnt/boot
```

### 6. Update Mirrorlist

```bash
$ reflector --verbose --protocol https --age 12 --latest 20 --sort rate -n 10 --save /etc/pacman.d/mirrorlist
```

### 7. Install Firmware Packages

```bash
$ pacstrap -K /mnt base linux linux-headers linux-firmware-intel intel-ucode mesa vulkan-intel intel-media-driver linux-firmware-nvidia nvidia-open nvidia-utils nvidia-prime linux-firmware-realtek pipewire pipewire-audio wireplumber sof-firmware btrfs-progs neovim sudo efibootmgr
```

---

## System Configuration

### 1. Generate fstab

```bash
$ genfstab -U -p /mnt >> /mnt/etc/fstab
$ cat /mnt/etc/fstab
```

### 2. Chroot into the New System

```bash
$ arch-chroot /mnt
```

### 3. Configure Locale & Timezone

```bash
$ pacman -Syu

$ ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
$ hwclock --systohc

$ sed -i 's/^#en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
$ locale-gen

$ echo "LANG=en_US.UTF-8" >> /etc/locale.conf
$ echo "KEYMAP=us" >> /etc/vconsole.conf
$ echo "<your-hostname>" >> /etc/hostname
$ echo "EDITOR=nvim" > /etc/environment
```

### 4. Install Packages

```bash
$ pacman -S --needed reflector networkmanager bluez bluez-utils blueman base-devel git python uv zram-generator curl alacritty fastfetch 7zip zsh awww hyprland waybar mako hyprlauncher hyprpolkitagent hyprpwcenter easyeffects qpwgraph playerctl xdg-desktop-portal xdg-desktop-portal-hyprland brightnessctl greetd greetd-tuigreet thunar thunar-volman gvfs tumbler thunar-archive-plugin file-roller btrfs-assistant noto-fonts noto-fonts-emoji ttf-jetbrains-mono-nerd power-profiles-daemon upower btop
```

### 5. Configure zram

<p>Edit /etc/systemd/zram-generator.conf</p>

```bash
# allocate same size as RAM of zram
[zram0]
zram-size=<same size as RAM>
compression-algorithm=zstd
swap-priority=100
```

<p>Edit /etc/sysctl.d/99-zram.conf</p>

```bash
vm.swappiness=150
vm.page-cluster=0
```

### 6. Configure Initramfs

```bash
$ nvim /etc/mkinitcpio.conf
# Set: MODULES=(i915 nvidia nvidia_modeset nvidia_uvm nvidia_drm)
$ mkinitcpio -P
```

### 7. User Creation & Password

```bash
$ useradd -m -G wheel -s /bin/zsh <username> # to create a user with the username
$ passwd <username> # set password for username

# Set up sudo access
$ EDITOR=nvim visudo
# Uncomment: %wheel ALL=(ALL:ALL) NOPASSWD:ALL
```

### 8. Install Bootloader (systemd-boot) - UEFI Only

```bash
$ bootctl install
```

<p>Edit /boot/loader/loader.conf</p>

```bash
default  arch.conf
timeout  3
console-mode max
editor   no
```

<p>Edit /boot/loader/entries/arch.conf</p>

```bash
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=UUID=<ROOT-UUID> rootflags=subvol=@ rw quiet nvidia_drm.modeset=1
```

<p>Edit /boot/loader/entries/arch-fallback.conf</p>

```bash
title   Arch Linux (fallback)
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux-fallback.img
options root=UUID=<ROOT-UUID> rootflags=subvol=@ rw
```

<p>Edit /etc/greetd/config.toml</p>

```bash
[terminal]
# Virtual terminal greetd runs on
vt = 1

[default_session]
# Enables numlock on tty1, then starts tuigreet which launches Hyprland.
# --remember pre-fills the last username ('user'), so you only type the password.
command = "sh -c 'setleds -D +num < /dev/tty1; tuigreet --time --remember --remember-session --asterisks --cmd Hyprland'"
user = "greeter"
```

```bash
$ mkdir -p /var/cache/tuigreet && chown greeter:greeter /var/cache/tuigreet
```

### 9. Enable Services

```bash
$ systemctl enable systemd-boot-update.service systemd-zram-setup@zram0.service NetworkManager systemd-timesyncd bluetooth firewalld reflector.timer fstrim.timer greetd.service nvidia-suspend nvidia-resume nvidia-hibernate pipewire pipewire-audio wireplumber power-profiles-daemon upower
```

### 10. Reboot

```bash
$ umount -R /mnt
$ reboot
```

---

## Post-Installation & Configuration

### 1. Connect to network

```bash
$ sudo nmcli dev wifi connect "SSID" password "Password"
```

### 1. Update & Upgrade

```bash
$ pacman -Syu
```

### 2. Install package managers

```bash
# install AUR helper
$ git clone https://aur.archlinux.org/paru.git
$ cd paru
$ makepkg -si
$ cd .. && rm -rf paru

# install flatpak
sudo pacman -S flatpak

# install snap store
paru snapd
sudo systemctl enable --now snapd.socket
```

### 3. Install useful Packages

```bash
$ sudo pacman -S <package-name>

# install zen-browser
$ flatpak install flathub app.zen_browser.zen

# install vscode
$ sudo snap install code --classic

# install clipboard app & screenshot tool
$ paru -Sy cursor-clip-git flameshot
```

### 5. Install OhMyZsh

- Check [OhMyZsh](https://ohmyz.sh).

### 6. Configure Hyprland

```bash
$ git clone https://github.com/Parshaw-Bhattacharjee/arch-conf.git
```

### 7. Path of the respective settings

> fastfetch: ~/.config/

> hypr: ~/.config/
<p>Note: Always make a backup of the old setting of hypr.</p>

> mako: ~/.config/

> waybar: ~/.config/

> zsh: ~/
<p>Note: Always paste the files inside the zsh folder.</p>
---

<p align="center">Kindly modify the commands & configuration as per your use-case.</p># Arch-Config
