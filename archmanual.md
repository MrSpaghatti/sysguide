# Modern Arch Linux Installation Guide

## Table of Contents

- [Introduction](#introduction)
- [Preliminary Steps](#preliminary-steps)
- [Main Installation](#main-installation)
  - [Disk Partitioning](#disk-partitioning)
  - [Disk Formatting](#disk-formatting)
  - [Disk Mounting](#disk-mounting)
  - [Package Installation](#package-installation)
  - [Fstab Generation](#fstab-generation)
  - [Chroot into New System](#chroot-into-new-system)
  - [Time Zone Setup](#time-zone-setup)
  - [Localization and Keyboard Layout](#localization-and-keyboard-layout)
  - [Hostname and Hosts Configuration](#hostname-and-hosts-configuration)
  - [Root and User Accounts](#root-and-user-accounts)
  - [GRUB Bootloader Configuration](#grub-bootloader-configuration)
  - [Final Steps Before Reboot](#final-steps-before-reboot)
  - [Automatic Snapshot Boot Entries Update](#automatic-snapshot-boot-entries-update)
  - [VirtualBox Guest Additions](#virtualbox-guest-additions)
  - [AUR Helper and Additional Packages](#aur-helper-and-additional-packages)
  - [Finalization](#finalization)
- [Video Drivers](#video-drivers)
  - [AMD](#amd)
    - [32-Bit Support (AMD)](#32-bit-support-amd)
  - [Nvidia](#nvidia)
  - [Intel](#intel)
- [Setting up a Graphical Environment](#setting-up-a-graphical-environment)
  - [Option 1: KDE Plasma](#option-1-kde-plasma)
  - [Option 2: Hyprland (WIP)](#option-2-hyprland-wip)
- [Adding a Display Manager](#adding-a-display-manager)
- [Gaming Setup](#gaming-setup)
  - [Gaming Clients](#gaming-clients)
  - [Windows Compatibility Layers](#windows-compatibility-layers)
  - [Generic Optimizations](#generic-optimizations)
  - [Overclocking and Monitoring](#overclocking-and-monitoring)
- [Additional Notes](#additional-notes)
- [Future Additions](#future-additions)

---

## Introduction

The goal of this guide is to help new users set up a modern and minimal installation of **Arch Linux** with **BTRFS** on an **UEFI system**. This guide covers the basic terminal installation, video driver setup, desktop environment installation, and basic gaming configuration. It is intended to be used alongside the [Arch Wiki](https://wiki.archlinux.org/title/Installation_guide) for the most up-to-date information. External references are provided for further details on specific choices.

### Important Considerations:

*   **Secure Boot:** This guide does **not** cover Secure Boot setup due to the risks involved with custom key enrollment ([potential bricking](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Creating_and_enrolling_keys)) and the questionable security of default OEM keys ([more info](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Implementing_Secure_Boot)).
*   **System Encryption:** Full system encryption is **not** covered as it can introduce boot time overhead. If encryption is required, refer to the [Arch Wiki dm-crypt page](https://wiki.archlinux.org/title/Dm-crypt) and implement it **immediately after** [disk partitioning](#disk-partitioning). Remember to set the partition type to LUKS instead of a standard Linux partition in `fdisk`.
*   **Arch ISO Preparation:** The creation of the Arch Linux installation media is **skipped**.
*   **Network Connection:** This guide assumes a **wired** internet connection. For Wi-Fi, use `wifi-menu` (TUI) or [`iwctl`](https://wiki.archlinux.org/title/Iwd#iwctl).

---

## Preliminary Steps

### 1. Set Keyboard Layout

List available keymaps and load the desired one.
```bash
# List Italian keymaps
ls /usr/share/kbd/keymaps/**/*.map.gz | grep it

# List all keymaps (interactive)
ls /usr/share/kbd/keymaps/**/*.map.gz | less
# Alternative listing
localectl list-keymaps

# Load Italian keymap (replace 'it' with your layout)
loadkeys it
```

### 2. Verify UEFI Boot Mode
```bash
# Output of 32 or 64 confirms UEFI mode
cat /sys/firmware/efi/fw_platform_size
```

### 3. Check Internet Connection
```bash
ping -c 5 archlinux.org
```

### 4. Synchronize System Clock
```bash
# Check status and time
timedatectl

# If NTP is not active:
timedatectl set-ntp true
# Or ensure the service is enabled:
sudo systemctl enable systemd-timesyncd.service
```

---

## Main Installation

### Disk Partitioning

This guide uses a 2-partition scheme:

| Partition | Type             | Size                               |
| :-------- | :--------------- | :--------------------------------- |
| 1         | EFI System       | 512M                               |
| 2         | Linux filesystem | Remaining space (e.g., 99.5GB)     |

```bash
# List block devices to identify your drive (e.g., /dev/nvme0n1, /dev/sda)
fdisk -l

# Start fdisk (replace /dev/nvme0n1 with your drive)
sudo fdisk /dev/nvme0n1
```
Inside `fdisk`, follow these steps (press Enter after each command character):
1.  `g` (Create a new empty GPT partition table)
2.  `n` (Add a new partition - for EFI)
3.  `ENTER` (Default partition number)
4.  `ENTER` (Default first sector)
5.  `+512M` (Size for EFI partition)
6.  `t` (Change partition type)
7.  `ENTER` (Default partition number, should be 1)
8.  `1` (Select EFI System type)
9.  `n` (Add a new partition - for root)
10. `ENTER` (Default partition number)
11. `ENTER` (Default first sector)
12. `ENTER` (Default last sector - use all remaining space)
    *   *Alternatively, specify size like `+100G` for a 100GB partition.*
13. `p` (Print the partition table to verify)
14. `w` (Write table to disk and exit)
    *   *Use `q` to quit without saving if you made a mistake.*

> **Tip:** For a more user-friendly TUI, consider using `cfdisk /dev/nvme0n1` instead of `fdisk`.

### Disk Formatting

This guide uses [**BTRFS**](https://wiki.archlinux.org/title/Btrfs) for its features like Copy-on-Write (CoW), snapshots, and built-in compression.

```bash
# Format EFI partition (e.g., /dev/nvme0n1p1)
sudo mkfs.fat -F 32 /dev/nvme0n1p1

# Format root partition (e.g., /dev/nvme0n1p2) with BTRFS
sudo mkfs.btrfs /dev/nvme0n1p2

# Mount the BTRFS volume to create subvolumes
sudo mount /dev/nvme0n1p2 /mnt
```

### Disk Mounting

A **flat** subvolume layout is used. For details on flat vs. nested layouts, see [this old BTRFS Sysadmin Guide section](https://archive.kernel.org/oldwiki/btrfs.wiki.kernel.org/index.php/SysadminGuide.html#Layout).

```bash
# Create BTRFS subvolumes for root and home
sudo btrfs subvolume create /mnt/@
sudo btrfs subvolume create /mnt/@home

# Unmount the temporary BTRFS mount
sudo umount /mnt
```

Mount subvolumes with **Zstd compression** ([benchmark reference](https://www.phoronix.com/review/btrfs-zstd-compress)):
```bash
# Mount root subvolume with Zstd compression
sudo mount -o compress=zstd,subvol=@ /dev/nvme0n1p2 /mnt

# Create home directory mount point and mount home subvolume
sudo mkdir -p /mnt/home
sudo mount -o compress=zstd,subvol=@home /dev/nvme0n1p2 /mnt/home
```

Mount the EFI partition to `/efi`. Using `/efi` instead of `/boot` avoids issues with restoring the root subvolume (`@`) after kernel updates, as `/boot` contents would not be part of the `@` snapshot. More info [here](https://wiki.archlinux.org/title/EFI_system_partition#Typical_mount_points).
```bash
sudo mkdir -p /mnt/efi
sudo mount /dev/nvme0n1p1 /mnt/efi
```

### Package Installation
```bash
# Bootstrap the system with essential packages.
# Adjust the list as needed.
# - base, linux, linux-firmware: Core system. (Use linux-lts for Long Term Support kernel)
# - base-devel: Basic development tools.
# - git: Version control system.
# - btrfs-progs: BTRFS utilities.
# - grub: Bootloader.
# - efibootmgr: For GRUB installation on UEFI.
# - grub-btrfs: GRUB support for BTRFS snapshots.
# - inotify-tools: Used by grub-btrfsd for automatic snapshot detection.
# - timeshift: GUI for BTRFS snapshots.
# - amd-ucode: Microcode for AMD CPUs (use intel-ucode for Intel).
# - vim: Text editor (or nano).
# - networkmanager: Network management.
# - pipewire, pipewire-alsa, pipewire-pulse, pipewire-jack: Modern audio framework.
# - wireplumber: PipeWire session manager.
# - reflector: Pacman mirror management.
# - zsh, zsh-completions, zsh-autosuggestions: Shell and enhancements.
# - openssh: SSH client and server.
# - man: Manual pages.
# - sudo: Privilege escalation.
sudo pacstrap -K /mnt base base-devel linux linux-firmware git btrfs-progs grub efibootmgr grub-btrfs inotify-tools timeshift amd-ucode vim networkmanager pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber reflector zsh zsh-completions zsh-autosuggestions openssh man sudo
```

### Fstab Generation
```bash
# Generate fstab file
sudo genfstab -U /mnt >> /mnt/etc/fstab

# Verify the generated fstab
cat /mnt/etc/fstab
```

### Chroot into New System
```bash
sudo arch-chroot /mnt
```

### Time Zone Setup
```bash
# Find your time zone, e.g., /usr/share/zoneinfo/Europe/Rome
# Create a symbolic link to /etc/localtime
sudo ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime

# Sync hardware clock
sudo hwclock --systohc
```

### Localization and Keyboard Layout

Edit `/etc/locale.gen` and uncomment your desired locales (e.g., `en_US.UTF-8 UTF-8`, `it_IT.UTF-8 UTF-8`).
```bash
sudo vim /etc/locale.gen # Or sudo nano /etc/locale.gen
```
Generate locales:
```bash
sudo locale-gen
```

Create `/etc/locale.conf` and set your language variables. For example, to use Italian formats with English display language:
```ini
LANG=it_IT.UTF-8
LC_MESSAGES=en_US.UTF-8
```
If you want everything in one language (e.g., English):
```ini
LANG=en_US.UTF-8
```
Edit the file:
```bash
sudo vim /etc/locale.conf
```
More on locale variables [here](https://wiki.archlinux.org/title/Locale#Variables).

Make TTY keyboard layout persistent. Create `/etc/vconsole.conf` and set `KEYMAP` (e.g., `KEYMAP=it`).
```bash
# Example for Italian layout
echo "KEYMAP=it" | sudo tee /etc/vconsole.conf
# Or edit manually:
# sudo vim /etc/vconsole.conf
```

### Hostname and Hosts Configuration
```bash
# Set your hostname (e.g., Arch)
echo "Arch" | sudo tee /etc/hostname # Replace "Arch" with your desired hostname

# Edit /etc/hosts
sudo vim /etc/hosts
```
Add the following to `/etc/hosts`, replacing `Arch` with **your** hostname:
```
127.0.0.1 localhost
::1       localhost
127.0.1.1 Arch.localdomain Arch
```

### Root and User Accounts
```bash
# Set root password
sudo passwd

# Add a new user (replace 'mjkstra' with your username)
# -m: create home directory
# -G wheel: add to wheel group (for sudo)
# Add vboxsf to groups if on VirtualBox and needing shared folders: useradd -mG wheel,vboxsf yourusername
sudo useradd -mG wheel mjkstra
sudo passwd mjkstra # Set password for the new user

# Allow users in the wheel group to use sudo
# This command opens /etc/sudoers with visudo.
# Uncomment the line: %wheel ALL=(ALL:ALL) ALL
sudo EDITOR=vim visudo # Or sudo EDITOR=nano visudo
```

### GRUB Bootloader Configuration
Install GRUB for UEFI systems:
```bash
sudo grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
```
Generate GRUB configuration file (this includes microcode):
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### Final Steps Before Reboot
```bash
# Enable NetworkManager to start on boot
sudo systemctl enable NetworkManager

# Exit chroot environment
exit

# Unmount all partitions
sudo umount -R /mnt

# Reboot the system (remove installation media)
sudo reboot
```
After rebooting, log in with your new user account.

Enable time synchronization:
```bash
sudo timedatectl set-ntp true
```

### Automatic Snapshot Boot Entries Update

Configure `grub-btrfsd` to automatically update GRUB boot entries when Timeshift creates snapshots.
Edit the `grub-btrfsd.service` file:
```bash
sudo systemctl edit --full grub-btrfsd
```
Replace the `ExecStart` line with:
`ExecStart=/usr/bin/grub-btrfsd --syslog --timeshift-auto`

If you don't use Timeshift, refer to the [grub-btrfs documentation](https://github.com/Antynea/grub-btrfs).

Enable the service:
```bash
sudo systemctl enable grub-btrfsd
```

### VirtualBox Guest Additions
(Only if running Arch Linux in a VirtualBox VM)
This enables clipboard sharing, shared folders, and resolution scaling.
```bash
# Install guest utilities
sudo pacman -S virtualbox-guest-utils

# Enable the service to load kernel modules
sudo systemctl enable vboxservice.service
```
> **Note:** Guest additions will work after a reboot. They generally require a graphical environment.

### AUR Helper and Additional Packages

Install an AUR (Arch User Repository) helper like `yay`. `yay` also acts as a `pacman` wrapper.
> **Warning:** GUI AUR helpers or package managers like `pamac` (from Manjaro) are not officially supported and can lead to [partial upgrades](https://wiki.archlinux.org/title/System_maintenance#Partial_upgrades_are_unsupported).

> **Note:** `makepkg` cannot be run as root. Log in as your regular user.

```bash
# Install yay dependencies and build yay
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd .. # Go back to previous directory
rm -rf yay # Clean up build files

# Install timeshift-autosnap (automatic snapshots before pacman upgrades)
yay -S timeshift-autosnap
```
Learn more about `timeshift-autosnap` [here](https://gitlab.com/gobonja/timeshift-autosnap).

### Finalization
```bash
# Reboot to apply all changes
sudo reboot
```
> Congratulations! You should now have a bootable Arch Linux system. The basic installation is complete. Continue if you want a graphical environment.

---

## Video Drivers

Install video drivers for optimal graphical performance, especially for gaming. Refer to the [Arch Wiki Xorg page](https://wiki.archlinux.org/title/Xorg#Driver_installation) for guidance.
> **Note:** Skip this section if you are on a Virtual Machine where guest additions handle graphics.

### AMD

This guide uses the open-source **AMDGPU** driver, recommended for GCN 3 architecture cards and newer (e.g., RX 400 series and later).
```bash
# mesa: DRI driver for 3D acceleration.
# vulkan-radeon: Vulkan support.
# libva-mesa-driver: VA-API hardware video decoding.
# mesa-vdpau: VDPAU hardware video decoding.
# xf86-video-amdgpu (optional): DDX driver for 2D acceleration in Xorg.
# This guide omits xf86-video-amdgpu, preferring kernel modesetting.
sudo pacman -S mesa vulkan-radeon libva-mesa-driver mesa-vdpau
```

#### 32-Bit Support (AMD)
Enable the `multilib` repository in `/etc/pacman.conf` by uncommenting the `[multilib]` section (remove `#` from both lines).
```bash
sudo vim /etc/pacman.conf # Or sudo nano /etc/pacman.conf
```
Then, refresh package lists and install 32-bit libraries:
```bash
yay # Refreshes and upgrades system (equivalent to sudo pacman -Syu)

sudo pacman -S lib32-mesa lib32-vulkan-radeon lib32-libva-mesa-driver lib32-mesa-vdpau
```

### Nvidia

You have two main options:
1.  [**NVIDIA proprietary driver**](https://wiki.archlinux.org/title/NVIDIA) (recommended for performance)
2.  [**Nouveau open-source driver**](https://wiki.archlinux.org/title/Nouveau)

This guide does not detail Nvidia driver installation due to its complexity and lack of testing hardware. Please consult the Arch Wiki.

### Intel

Installation is similar to AMD, often replacing `radeon` with `intel` in package names. For hardware video acceleration, specific packages are needed. Consult the [Arch Wiki Intel graphics page](https://wiki.archlinux.org/title/Intel_graphics#Installation).

---

## Setting up a Graphical Environment

Two options are presented:
1.  **KDE Plasma**
2.  **Hyprland** (Tiling Window Manager)

A display manager is also recommended.

### Option 1: KDE Plasma

**KDE Plasma** is a feature-rich, lightweight, and user-friendly desktop environment supporting Xorg and Wayland.
```bash
# Minimal Plasma desktop and essential components:
# plasma-desktop: Core Plasma desktop.
# plasma-pa: Audio applet.
# plasma-nm: Network applet.
# plasma-systemmonitor: Task manager.
# plasma-firewall: Firewall configuration.
# plasma-browser-integration: Browser media control, etc.
# kscreen: Display configuration.
# kwalletmanager, kwallet-pam: Password management.
# bluedevil: Bluetooth manager.
# powerdevil, power-profiles-daemon: Power management.
# kdeplasma-addons: Useful desktop add-ons.
# xdg-desktop-portal-kde: Integration for Flatpak/portals.
# xwaylandvideobridge: Screen sharing compatibility for XWayland apps.
# kde-gtk-config, breeze-gtk: GTK application theming.
# cups, print-manager: Printing support.
# konsole: KDE terminal.
# dolphin, ffmpegthumbs: KDE file manager and video thumbnails.
# firefox: Web browser.
# kate: Text editor.
# okular: Document viewer.
# gwenview: Image viewer.
# ark: Archive manager.
# pinta: Simple image editor (GTK-based).
# spectacle: Screenshot tool.
# dragon: Simple media player. (Haruna is an alternative based on libmpv).
sudo pacman -S plasma-desktop plasma-pa plasma-nm plasma-systemmonitor plasma-firewall plasma-browser-integration kscreen kwalletmanager kwallet-pam bluedevil powerdevil power-profiles-daemon kdeplasma-addons xdg-desktop-portal-kde xwaylandvideobridge kde-gtk-config breeze-gtk cups print-manager konsole dolphin ffmpegthumbs firefox kate okular gwenview ark pinta spectacle dragon
```
> **Do not reboot yet.** Proceed to [Adding a Display Manager](#adding-a-display-manager). If you skip the display manager, you'll need to [manually configure KDE launch](https://wiki.archlinux.org/title/KDE#From_the_console).

### Option 2: Hyprland (WIP)

> **Note:** This section is a work-in-progress and requires further configuration.

**Hyprland** is a dynamic tiling Wayland compositor known for its aesthetics and features. It is based on **wlroots**. Configuration requires reading the [Hyprland Wiki](https://wiki.hyprland.org/) and potentially the [Master Tutorial](https://wiki.hyprland.org/Getting-Started/Master-Tutorial).

```bash
# Install Hyprland and common utilities:
# hyprland: The compositor itself.
# swaylock: Screen locker.
# wofi: Application launcher (Wayland alternative to rofi).
# waybar: Status bar for Wayland.
# dolphin: KDE file manager (can be used with Hyprland).
# alacritty: GPU-accelerated terminal emulator.
sudo pacman -S --needed hyprland swaylock wofi waybar dolphin alacritty

# wlogout: Logout/shutdown menu (AUR package)
yay -S wlogout
```

---

## Adding a Display Manager

A **display manager** (login manager) allows selecting your desktop environment/window manager and display protocol (Wayland/Xorg) at startup. **SDDM** is a popular choice.

> **Note:** While Hyprland doesn't officially support display managers, SDDM is [reported to work well](https://wiki.hyprland.org/Getting-Started/Master-Tutorial/#launching-hyprland).

```bash
# Install SDDM
sudo pacman -S sddm

# Enable SDDM service to start on boot
sudo systemctl enable sddm

# For KDE users: install sddm-kcm to configure SDDM from KDE System Settings
sudo pacman -S --needed sddm-kcm # If you installed KDE

# Reboot to start with the display manager
sudo reboot
```

---

## Gaming Setup

Ensure video drivers are installed. Also, ensure `lib32-mesa`, `lib32-vulkan-radeon` (for AMD), and `lib32-pipewire` are installed (requires `multilib` repository).

Key components for Linux gaming:
1.  **Gaming Client:** (e.g., Steam, Lutris, Bottles)
2.  **Windows Compatibility Layers:** (e.g., Proton, Wine, DXVK, VKD3D)

Optional enhancements:
1.  **Generic Optimizations:** (e.g., Gamemode)
2.  **Overclocking/Monitoring Software:** (e.g., CoreCtrl, MangoHud)
3.  **Custom Kernels**

### Gaming Clients

Install **Steam** and **Bottles** (via Flatpak for managing Wine prefixes).
```bash
# Install Steam and Flatpak
sudo pacman -S steam flatpak

# Install Bottles from Flathub
flatpak install flathub com.usebottles.bottles
```

### Windows Compatibility Layers

**Proton** (by Valve) bundles Wine, DXVK (DirectX 9-11 to Vulkan), and VKD3D-Proton (DirectX 12 to Vulkan). Enable it in Steam: `Steam > Settings > Compatibility > Enable Steam Play for all other titles`.
**Proton GE (GloriousEggroll)** is a custom version of Proton often with newer fixes.
```bash
# Install Proton GE via AUR
yay -S proton-ge-custom-bin
```

### Generic Optimizations

**Gamemode** (by Feral Interactive) applies system optimizations when a game is running.
```bash
# Install Gamemode
sudo pacman -S gamemode
```
Refer to the [Gamemode GitHub page](https://github.com/FeralInteractive/gamemode#requesting-gamemode) for usage.

### Overclocking and Monitoring

**MangoHud** for in-game performance overlay. **GOverlay** for configuring MangoHud.
```bash
# Install GOverlay (includes MangoHud)
sudo pacman -S goverlay
```
Enable MangoHud as per its [documentation](https://github.com/flightlessmango/MangoHud#normal-usage).

For GPU overclocking/tuning:
*   AMD: [**CoreCtrl**](https://gitlab.com/corectrl/corectrl)
*   NVIDIA: [**TuxClocker**](https://github.com/Lurkki14/tuxclocker)
Installation for these tools can be found on their respective pages or via `yay`.

---

## Additional Notes

*   **Mouse Acceleration (KDE):** Disable in `System Settings > Input Devices > Mouse > Advanced > Acceleration Profile: Flat`. For non-KDE environments, see [Arch Wiki: Mouse acceleration](https://wiki.archlinux.org/title/Mouse_acceleration).
*   **Variable Refresh Rate (FreeSync/G-Sync):** Configuration depends on session type (Wayland/Xorg) and GPU. See [Arch Wiki: Variable refresh rate](https://wiki.archlinux.org/title/Variable_refresh_rate). In KDE Wayland, enable "Adaptive Sync" in display settings.
*   **Custom Kernels:**
    *   Require recompilation on updates unless using precompiled versions (e.g., `linux-zen` from official repos, or others from AUR).
    *   Kernels often involve trade-offs (e.g., latency vs. throughput). The generic kernel is usually fine.
    *   For gaming, **TKG** or **CachyOS** kernels are options. TKG requires manual compilation. CachyOS provides precompiled kernels installable via their repository. Performance benefits can be subjective and system-dependent.
*   **Recommended Reads:**
    *   [How to build your KDE environment](https://community.kde.org/Distributions/Packaging_Recommendations)
    *   [Gaming on Wayland](https://zamundaaa.github.io/wayland/2021/12/14/about-gaming-on-wayland.html) (older but still relevant)
    *   [Improving Linux Gaming Performance](https://linux-gaming.kwindu.eu/index.php?title=Improving_performance)
    *   [Arch Linux System Maintenance](https://wiki.archlinux.org/title/System_maintenance)
    *   [Arch Linux General Recommendations](https://wiki.archlinux.org/title/General_recommendations)

---

## Future Additions

1.  Additional `pacman` configuration (paccache, colors, parallel downloads).
2.  `reflector` configuration for automatic mirror ranking.
3.  **Snapper:** Advanced snapshot management as an alternative to Timeshift.
4.  Enhanced BTRFS subvolume layout (e.g., separate subvolumes for `/var/log`, `/var/cache`, `/tmp`, and snapshots like `@log`, `@cache`, `@tmp`, `@snapshots`) to exclude them from root filesystem snapshots.
5.  Optimized `fstab` structure.

---
