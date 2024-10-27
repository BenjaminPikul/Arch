# Arch Linux VM Installation

Course: CYB 3353 - System Administration

Student: Benjamin Pikul

Instructor: Justin Miller

Institution: University of Tulsa

## Step 1: Preparation

### Step 1.1 Arch Linux Wiki and Download

Review the Arch Linux installation guide.

Arch Linux ISO: archlinux-2024.10.01-x86_64.iso

Download location: University of Arizona Mirror

SHA256 Hash:

    Downloaded: B72DD6FFEF7507F8B7CDDD7C69966841650BA0F82C29A318CB2D182EB3FCB1DB
    
    Expected: b72dd6ffef7507f8b7cddd7c69966841650ba0f82c29a318cb2d182eb3fcb1db

Note: Hashes are case-insensitive, so capitalization doesn't matter.

### Step 1.2 VMware Workstation Pro

Download location: Broadcom

SHA256 Hash:

    Downloaded: F95429E395A583EB5BA91F09B040E2F8C53A5E7AA37C4C6BFCAF82115A8D3FA4
    
    Expected: f95429e395a583eb5ba91f09b040e2f8c53a5e7aa37c4c6bfcaf82115a8d3fa4

Once verified, run VMware Workstation Pro.

## Step 2: Create a New Virtual Machine (VM)

Select ISO: Use the downloaded Arch Linux ISO as the disc image.

Choose Kernel: Select "Other Linux 6.x kernel 64-bit" for best compatibility.

Storage: Allocate 25 GB for the VM.

Memory & CPUs: 4 GB of Ram and 2 CPUs

Switch to UEFI: VM Settings -> Options -> Advanced -> Change BIOS to UEFI.

Launch the VM and review your next steps while it boots.

## Step 3: Install Arch Linux via Command Line

### Step 3.1: Verify Boot Mode
To ensure the system booted in UEFI mode, run:

    ls /sys/firmware/efi/efi/vars or cat /sys/firmware/efi/fw_platform_size
    
The expected outputs should be a large list of files or the number 64 respecitvly

### Step 3.2: Check Network Availability
Run the following commands to verify network interfaces:

    ip link

    ip addr

The goal here is to identify interfaces like ens33 (VMware Workstation by default should provide ens33 if you have networking enabled)

### Step 3.3: Sync Network Time
    
    timedatectl set-ntp true
    
### Step 3.4: Configure Storage

#### Step 3.4.1: Write Partition Table
Use fdisk to create a new GPT partition table:

    fdisk /dev/sda  
    
    g  
    
    w  

#### Step 3.4.2: Create Primary Partitions
Use parted to define the partitions:

    parted /dev/sda  
    
    mkpart ESP fat32 1MiB 513MiB  
    
    mkpart primary 513MiB 1025MiB  
    
    mkpart primary 1025MiB 100%  
    
    name 1 efi  
    
    name 2 boot 
    
    name 3 lvm  
    
    set 1 boot on  
    
    set 3 lvm on  
    
    quit

This creates three partitions: efi (512MiB), boot (512MiB), lvm (remaining space).

Verify changes with parted -l and lsblk.

#### Step 3.4.3: Create LVM Volumes

    pvcreate /dev/sda3  
    
    vgcreate arch /dev/sda3  
    
    lvcreate arch -n root -L 5G  
    
    lvcreate arch -n var -L 5G  
    
    lvcreate arch -n home -L 12G  
    
    lvcreate arch -n swap -l 100%FREE  

#### Step 3.4.4: Format Partitions

    mkfs.fat -F32 /dev/sda1  
    
    mkfs.ext4 /dev/sda2       
    
    mkfs.ext4 /dev/arch/root   
    
    mkfs.ext4 /dev/arch/var    
    
    mkfs.ext4 /dev/arch/home   
    
    mkswap /dev/arch/swap    
    
    swapon /dev/arch/swap

These commands format the Partitions in this order : EFI, Boot, Root, Var, Home, and then makes and enables swap

#### Step 3.4.5: Mount Filesystems

    mount /dev/arch/root /mnt  
    
    mkdir /mnt/{efi,boot,var,home}  
    
    mount /dev/sda1 /mnt/efi  
    
    mount /dev/sda2 /mnt/boot  
    
    mount /dev/arch/var /mnt/var  
    
    mount /dev/arch/home /mnt/home
    
Verify the mounts with: 

    mount|tail -n 5

### Step 3.5: Install Essential Packages
Use pacstrap to install essential packages:

base , linux , and linux-firmware

At this point also install: 

grub, efibootmgr , lvm2 , and nano

This command installs all of the above packages:

    pacstrap /mnt base linux linux-firmware grub efibootmgr lvm2 nano

Update the filesystem table: 

    genfstab -U /mnt >> /mnt/etc/fstab

### Step 3.6: Chroot into the New System

    arch-chroot /mnt

## Step 4: Configure System

### Step 4.1: Set Root Password

    passwd

### Step 4.2: Adjust Time Settings
Set the timezone and synchronize system clocks:

    ln -sf /usr/share/zoneinfo/Region/City /etc/localtime  
    
    hwclock --systohc

### Step 4.3: Set Locale
Edit /etc/locale.gen to uncomment en_US.UTF-8 UTF-8

Generate locales:
    
    locale-gen  
    
    echo "LANG=en_US.UTF-8" > /etc/locale.conf

### Step 4.4: Configure Network

Identify the network interface with:

    ip link
    
Edit /etc/systemd/network/20-wired.network for DHCP
Then enable networking:

    systemctl enable systemd-networkd systemd-resolve

Set hostname and update /etc/hosts by:

editing /etc/hostname and saving the perfered hostname to it

edit /etc/hosts and save the following lines:

    127.0.0.1 localhost
    ::1 localhost
    127.0.1.1 arch
    
### Step 4.5: Update Initial Ramdisk
Edit /etc/mkinitcpio.conf and replace:

    HOOKS=(base udev autodetect modconf block filesystems keyboard fsck) 

with 

    HOOKS=(base udev autodetect modconf block lvm2 filesystems keyboard fsck)
    
The order matters here, make sure lvm2 is between block and filesystems
Finish updating the Ramdisk

    mkinitcpio -P 

## Step 5: Configure Bootloader

### Step 5.1: Install GRUB:
Install grub with a target efi directoroy

    grub-install --target=x86_64-efi --efi-directory=/efi  

### Step 5.2 Configure Grub:
These commands configure grub for the boot and efi

    grub-mkconfig -o /boot/grub/grub.cfg  
    grub-mkconfig -o /efi/EFI/arch/grub.cfg

## Step 6: Finalize Installation
Exit the chroot environment and reboot:

    exit
    reboot
    
login as root to finish customization

## Step 7: Customization & Project Requirements

### Step 7.1: Desktop Environment
Install Cutefish, Kwin, Xorg, and SDDM:

    pacman -S cutefish kwin xorg sddm
    
Note: Despite not being one of cutefish's dependancies on the arch wiki Kwin actually required for the cutefish DE, as it without it cutefish lacks critical features.

Enable SDDM to boot into a graphical interface:

    systemctl enable sddm

### Step 7.2: Create User Accounts
Create 3 users and assign them to the wheel group:

    useradd -m -G wheel -s /bin/bash username  

    echo 'username:password' | chpasswd

Force users to change passwords on first login:

    chage -d 0 username

### Step 7.3 Install and configure sudo:

    pacman -S sudo  
    
    EDITOR=nano visudo

Uncomment %wheel ALL=(ALL) ALL
This grants sudo privileges to wheel group members.

### Step 7.4: Install Fish Shell and SSH

    pacman -S fish openssh  
    
    chsh -s /usr/bin/fish username  
    
    systemctl enable sshd  
    
    systemctl start sshd

### Step 7.5: Color Coding in Fish
Edit Fish config files (/home/username/.config/fish/config.fish) for colorized output.

I added the following:

    #Colorize ls and grep
    alias ls='ls --color=auto'
    alias grep='grep --color=auto'
    #Global color schemes
    set -g fish\_color\_command --bold cyan
    set -g fish\_color\_param white
    set -g fish\_color\_comment yellow
    set -g fish\_color\_error red
    set -g fish\_color\_operator magenta
    set -g fish\_color\_escape cyan
    set -g fish\_color\_autosuggestion '555' # Dim gray for autosuggestions
    set -g fish\_color\_selection --background=brwhite --foreground=black

### Step 7.6: Aliases
To enhance your terminal experience, add some useful aliases to the Fish shell's configuration file. Edit /home/username/.config/fish/config.fish

I added the following:

    alias update='sudo pacman -Syu'
    alias clean='sudo pacman -Sc'
    alias reboot='sudo reboot'
    alias shutdown='sudo shutdown now'
    alias ll='ls -lh'
    alias la='ls -la'
    alias ..='cd ..'
    alias gst='git status'
    alias gaa='git add --all'
    alias gc='git commit -m'
    alias gp='git push'
    alias home='cd ~'
    alias open='xdg-open .'
    alias myip="ip addr show | grep 'inet ' | grep -v '127.0.0.1'"
    alias pingg='ping google.com'
    alias install='sudo pacman -S'
    alias remove='sudo pacman -R'
    alias search='pacman -Ss'
    alias psf='ps aux | grep'
    alias killp='pkill'

To activate the changes made:

    source /home/username/.config/fish/config.fish

## Step 8: Post-Installation and Final setup

### Step 8.1: Reboot and Customization
Reboot the VM and login through the graphical SDDM menu

Verify that everything is working

Customize parts of the DE and GUIs

Install the package tree using pacman.

### Step 8.2 Install a web browser (Floorp) of the AUR
Install git and base-devel using pacman -S.
clone the Floorp AUR package

    git clone https://aur.archlinux.org/floorp.git

Move into the floorp/bin directory and use makepkg to build Floorp

    makepkg -si
