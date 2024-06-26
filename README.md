![alt text](https://github.com/treesgreens/debian-btrfs/main/debian-logo.png)

Manually install Debian Luks with BTRFS subvolumes: 

 - Single LUKS2 encrypted partition which contains the full installation
 - Single BTRFS filesystem (integrated home partition)
 - Encrypted swapfile in BTRFS subvolume (supports laptop suspend but not hibernate)
 - Uses systemd-boot bootloader (instead of Grub2, also optional rEFInd instructions)
 - Minimal Gnome install (plus instructions for any other DE you wish)
 - Proper user groups for common security tools like sudo-less Wireshark, etc...
 - Optional removal of crypto keys from RAM during laptop suspend
 - Optional configurations for laptops (including fingerprint readers)
 - Optional dualboot to Windows
 - Emergency recovery steps for failed bootloaders or other problems

Originally created in 2021-11, check GitHub Gist Revisions for last update

# Debian installation (debootstrap)

## Pre-installation setup

Boot from latested stable or testing release of Debian/Ubuntu LiveISO.

Installed needed packages

    apt install debbootstrap cryptsetup arch-install-scripts

## Partitions - Part 1

### Creating partitions

Perform formatting and partitioning with Gnome Disks or `parted`.  

 - Format drive using `gpt` partition table
 - Make partitions but do not create filesystems (If using Gnome Disks, choose filesystem `other` then `No Filesystem`)
 - Label your partitions

Suggested partitions on example drive `/dev/nvme0n1`:

 - /dev/nvme0n1p1 with label `EFI` for UEFI (suggested 1GB, systemd-boot stores kernel+initrd here)
 - /dev/nvme0n1p2 with label `Debian` install (will include /boot and swap)
 - optional extra unpartitioned free space for Windows
 
Verify partition names with `lsblk` and write down partion UUIDs from `blkid` before proceeding.

Open a terminal and `sudo -s` to root.

Install the needed packages:

    apt install debootstrap

### Preparing partitions

Format the first partition as EFI (boot) and set needed flags:

    mkfs.fat -F 32 -n EFI /dev/nvme0n1p1
    parted /dev/nvme0n1 set 1 esp on
    parted /dev/nvme0n1 set 1 boot on

Prepare the main encrypted partition

    cryptsetup -y -v --type luks2 luksFormat --label Debian_Sid /dev/nvme0n1p2

Respond with a "YES" and enter a passphrase twice (-y provides this).

Open the main partition with a name "cryptroot"

    cryptsetup open /dev/nvme0n1p2 cryptroot
    <passphrase>

Format the main partition as btrfs

    mkfs.btrfs /dev/mapper/cryptroot



## Partitions - Part 2

### Creating BTRFS subvolumes

Mount main partition temporarily

    mount /dev/mapper/cryptroot /mnt

Create subvolumes for rootfs, home, var and snapshots

    btrfs subvolume create /mnt/@
    btrfs subvolume create /mnt/@home
    btrfs subvolume create /mnt/@snapshots
    btrfs subvolume create /mnt/@swap
    
Unmount the partition

    umount /mnt

### Re-mounting subvolumes as partitions to /mnt

    mount -o noatime,compress=zstd:1,subvol=@ /dev/mapper/cryptroot /mnt
    mkdir -p /mnt/{boot,home,.snapshots}
    mkdir /mnt/boot/efi
    mount /dev/nvme0n1p1 /mnt/boot/efi
    mount -o noatime,compress=zstd:1,subvol=@home /dev/mapper/cryptroot /mnt/home
    mount -o noatime,compress=zstd:1,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
    
### Setup your swapfile, change the `count` value in the `dd` command to meet your needs

    mkdir -p /mnt/swap
    mount -o subvol=@swap /dev/mapper/cryptroot /mnt/swap
    touch /mnt/swap/swapfile
    chmod 600 /mnt/swap/swapfile
    chattr +C /mnt/swap/swapfile
    btrfs property set ./swapfile compression none
    dd if=/dev/zero of=/mnt/swap/swapfile bs=1M count=16384
    mkswap /mnt/swap/swapfile
    swapon /mnt/swap/swapfile



## Base installation

### Bootstrapping a base-system

With all partitions mounted, run

    debootstrap --arch amd64 sid /mnt

Create a child subvolme for `/var/log` contained inside of your `@` subvolme.  These will be auto-mounted with `@` so there is no need to add it separately to `/etc/fstab`.  This will keep our log files from being included in our snapshots so our log files remain untouched if we ever restore our rootfs from snapshot.  __WARNING:__ Do not do this for all of `/var`, as many subvolders in `/var` such as `/var/lib` need to stay sinked with applications installed in `/usr`.  Creating a separate subvolume for `/var` is common in ZFS but ZFS handles snapshots and restorations differently than BTRFS.  So do not blindly following ZFS subvolume suggestions for BTRFS.

    btrfs subvolume create /mnt/var/log

### Preparing the `chroot` environment

Copy the mounted file systems table

    cp /etc/mtab /mnt/etc/mtab

Bind the pseudo-filesystems for chroot

    mount -o bind /dev /mnt/dev
    mount -o bind /dev/pts /mnt/dev/pts
    mount -o bind /proc /mnt/proc
    mount -o bind /sys /mnt/sys

### Generating fstab

Run `genfstab` from arch-install-scripts we installed earlier to help setup fstab for us

    genfstab -U /mnt >> /mnt/etc/fstab

### Changing root to the new system

    chroot /mnt



## Configuration

### Setting timezone

    dpkg-reconfigure tzdata

Choose an appropriate region.

### Setting locale

    dpkg-reconfigure locales

Choose `en_US.UTF-8` from the list of locales, or whatever is appropriate for you.

### Setting `HOSTNAME` for your machine, this example uses `gtfo`

    echo 'gtfo' > /etc/hostname
    echo '127.0.1.1 gtfo.localdomain gtfo' >> /etc/hosts

### Setting up `apt` sources

Update `/etc/apt/sources.list` to contain the following:

    deb https://deb.debian.org/debian sid main contrib non-free

### Install your your desktop environment

    apt update
    apt install linux-headers-amd64 firmware-iwlwifi firmware-linux firmware-linux-nonfree sudo vim bash-completion command-not-found plocate systemd-timesyncd usbutils hwinfo v4l-utils
    
Install whichever desktop environment you want, my prefernce is a basic, bare-bones install of Gnome:
    
    apt install gnome-core
    
Other Desktop Environment (DE) options are:

    # task-cinnamon-desktop
    # task-gnome-desktop
    # task-kde-desktop
    # task-lxde-desktop
    # task-lxqt-desktop
    # task-mate-desktop
    # task-xfce-desktop
    
If you are installing this on a laptop:

    apt install task-laptop powertop gnome-power-manager linux-cpupower cpupower-gui

## Creating users and groups

### Setting root password

    passwd

### Creating a non-root user

Create your main user.  Replace `meeas` with your username below.

    useradd meeas -m -c "Justin Searle" -s /bin/bash

Set password for your user

    passwd meeas

Add user to sudo group

    usermod -aG sudo,adm,dialout,cdrom,floppy,audio,dip,video,plugdev,users,netdev,bluetooth,wireshark meeas



## Setting up bootloader

### Installing bootloader utils

    apt install efibootmgr btrfs-progs cryptsetup-initramfs
    
### Optional package for extra protection suspsended laptops

One attack vector for encrypted filesystems on laptops is to steal the crypto keys from RAM while the system is suspected. The keys are left in RAM to maintain access to the filesystem upon waking.  Linux has LUKS2 support to remove the cypto keys from memory to close this vulnerability, but it does require you to type your LUKS2 passphrase when you wake it from suspend.  If you want this extra protection:
    
    apt install cryptsetup-suspend

### Setting up encryption parameters

Use `blkid` to get the `UUID` of your encrypted partition, which is `/dev/nvme0n1p2` in this example

Create an entry in the `/etc/crypttab` file

    cryptroot <tab> UUID=uuid-of-encrypted-partition <tab> none <tab> luks

### Setup systemd-boot (instead of Grub)

If you want to use rEFInd as your bootloader, you will still need to do these steps as rEFInd will need systemd-boot to access the LUKS2 encrypted root filesystem.

    bootctl install
    mkdir -p /boot/efi/`cat /etc/machine-id`
    kernel-install add `uname -r` /boot/vmlinuz-`uname -r`
    
    # Previous comamnds just in case someone needs them
    bootctl install
    V=`ls /boot/vmlinuz-* | cut -d - -f 2-`
    kernel-install add $V /boot/vmlinuz-$V
    
### Setup auto-updating of system-boot

Create and set permissions on the auto-install script

    touch /etc/kernel/postinst.d/zz-update-systemd-boot
    chmod +x /etc/kernel/postinst.d/zz-update-systemd-boot
    
Edit this file to contain the following text:

    #!/bin/sh
    set -e
    /usr/bin/kernel-install add "$1" "$2"
    exit 0
    
Create and set permissions on the auto-remove script
    
    touch /etc/kernel/postrm.d/zz-update-systemd-boot
    chmod +x /etc/kernel/postrm.d/zz-update-systemd-boot
    
Edit this file to contain the following text:

    #!/bin/sh
    set -e
    /usr/bin/kernel-install remove "$1"
    exit 0

### Testing your new bootloader

Exit chroot

    exit

Unmount all mounted partitions

    umount -a

Reboot

    reboot

If your bootloader fails and does not allow you to boot into your new system, follow the istructrions in the [Emergency Recovery] sections below to try this section again.



# Post installation

## Basic system configuration

### Console setup

Assuming you are now logged into your newly created useraccount (and not root)

    sudo dpkg-reconfigure console-setup

### Connecting to the internet

As long as you installed gnome-core as recommended in the previous steps, you can use the GUI to setup your Internet.

If you didn't install a DE like Gnome, you can use `nmtui` to setup your network at the command prompt.

### Install whatever other software you need or want.  I'd suggest this at a minimum:

    sudo apt install git curl wget wireshark

### If you have a fingerprint reader:
    
    sudo apt install libpam-fprintd
    sudo pam-auth-update    ### Enable fingerprint authentication
    fprintd-enroll
    fprintd-verify

### If you want to use rEFInd as your bootloader

Personally, I recommend just staying with systemd-boot in most scenarios since it supports multi-boot out of the box and rEFInd still needs it to boot into the encrypted rootfs.  But the one scenario I personally use rEFInd is if you have dual hard drives, each with an EFI partition with separate bootloaders.  In this instance, rEFInd will auto-detect all the bootloaders on both drives allowing an easy path to boot whatever OS is there.  For me, this second drive is a 1TB USB module that I can insert into the removable USB modules of my Framework laptop, which contains several other Linux distro installs that I play with.  rEFInd allows to me boot to any of them directly if it detects the drive is inserted.  This also allows me to install anything I with to the USB module and and its EFI partition without modifying my main drive's EFI partition.

If you decide this is for you, then:

    sudo apt install refind
    sudo refind-install
    sudo refind-mkdefault

Reboot and verify rEFInd comes up and that it can see/use your systemd-boot loader.

## Working with BTRFS snapshots

### When to create BTRFS subvolumes

BTRFS snapshots allow you to backup and restore all files in a BTRFS subvolume.  Sometimes there are subdirectory trees that you do not want included in your backup/restore because they are variable data like logs, because they are backed up elsewhere, or because you want to have a different BTRFS backup scheme for that subdirectory tree..  I have found this to be the primariy reason for creating BTRFS subvolumes. 

### Subvolumes for the main OS

For our root filesystem, this guide has you create the following subvolumes for the following reasons:

 - Subvolume `@`, mounted to `/` for our root filesystem so we can back it up upon package installs/updates
 - Subvolume `@home`, mounted to `/home/` for your main user so we can back it up on a fixed schedule, separate from our root filesystem.  If your are installing this on a machine with mulitple different users, you could instead create a separate BTRFS subvolume for each user under home instead of `/home/` itself. 
 - Subvolume `@/var/log` so we don't loose log history of our root filesystem restorations.  Unlike the other two, this is a child subvolme to `@` so does not have its own entry in `/etc/fstab`.

If you use container technology such as Docker or LXC/LXD, these tools will probably create a few more child subvolumes under `@` for each of your containers.

### Subvolumes in home folders

For my home folder, I create a separate subvolume for `~/Downloads` so keep thoes disposable files from being backedup.  BTW, you do not need to use `sudo` when creating subvolumes in your user's home directory.

    rm ~/Downloads
    btrfs create subvolume ~/Downloads
   
If you use a product to sync files with Google Drive or some other cloud storage, I recommend keeping a separate subvolume to keep them from your home folder backups.  I personally use Insync to sync certain files from Google Drive, so I create:
    
    btrfs create subvolume ~/Google
    
If you do any software development, you are probably are saving your sourcecode elsewhere using tools like git.  Therefore I usually exclude those folders from my home user backups with:

    btrfs create subvolume ~/GitHub
    btrfs create subvolume ~/go
    
If you use virtualization products, at a minimum you need to turn off CoW features of BTRFS to avoid their virtualdisk block writes from thrashing your BTRFS filesystem and SSD.  I also like creating separate subvolumes so my home folder backups don't backup the VMs as I consider all of my VMs disposable.  Here is what I do for the three virtualization products I use.  This will automatically get Gnome Boxes saving VMs to the right place, but you will need to manually configure VMware and VirtualBox to create their VMs in the folders we create for them below.
    
    btrfs create subvolume ~/VMs
    mkdir ~/VMs/VirtualBox ~/VMs/VMware ~/VMs/Gnome-Boxes
    chattr +C ~/VMs ~/VMs/*
    ln -s ~/VMs/Gnome-Boxes ~/.local/share/gnome-boxes/images


# Emergency Recovery

Boot from a Debian/Ubuntu Live CD, open a terminal, `sudo -s` to root, then:

    sudo -i
    cryptsetup open /dev/nvme0n1p2 cryptroot
    mount -o noatime,compress=zstd:1,subvol=@ /dev/mapper/cryptroot /mnt
    mount /dev/nvme0n1p1 /mnt/boot/efi
    mount -o noatime,compress=zstd:1,subvol=@home /dev/mapper/cryptroot /mnt/home
    mount -o noatime,compress=zstd:1,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
    mount -o subvol=@swap /dev/mapper/cryptroot /mnt/swap
    swapon /mnt/swap/swapfile
    cp /etc/mtab /mnt/etc/mtab
    mount -o bind /dev /mnt/dev
    mount -o bind /dev/pts /mnt/dev/pts
    mount -o bind /proc /mnt/proc
    mount -o bind /sys /mnt/sys
    chroot /mnt

