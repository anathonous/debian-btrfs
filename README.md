<p align="center"><img src=debian-logo.png width="300"></p><br>

Manually install Debian trixie/testing with LUKS2 and BTRFS with subvolumes and Encrypted Swap: 

 - EFI Boot Partition
 - LUKS2 Swap Partition
 - LUKS2 Container
 - BTRFS Partition
 - 2 BTRFS Subvolumes

# Debian installation (debootstrap)

## Pre-installation setup

Boot from latested stable or testing release of Debian/Gentoo LiveISO. I personally used a working Gentoo Environment.

Installed needed packages

    apt install debbootstrap cryptsetup
On Gentoo
````
sudo emerge -av debootstrap cryptsetup
````

## Partitions

### Creating partitions

Perform formatting and partitioning with cfdisk. 
````
cfdisk /dev/nvme0n1
````
 - create a new partition 2G or more. Change type to EFI System.
 - create a new partition 8G or more. Leave type as Linux.
 - Create a new partition 100%FREE. Use remainder of drive. Leave type as Linux.
 - Save and write to disk
 - Ext

### Preparing partitions

Format the first partition as EFI (boot) and set needed flags:

    mkfs.fat -F 32 /dev/nvme0n1p1
    parted /dev/nvme0n1 set 1 esp on
    parted /dev/nvme0n1 set 1 boot on

Prepare the second encrypted swap partition
````
    cryptsetup luksFormat -s 256 -c aes-xts-plain64 /dev/nvme0n1p10
````
Respond with a "YES" and enter a passphrase twice (-y provides this).

Open the main partition with a name "OPEN"

    cryptsetup open /dev/nvme0n1p2 DEBIANSWAP 
    <passphrase>

Format the partition as swap

````
mkswap /dev/mapper/DEBIANSWAP
swapon /dev/mapper/DEBIANSWAP
````

### Creating BTRFS main and subvolumes
````
cryptsetup luksFormat -s 256 -c aes-xts-plain64 /dev/nvme0n1p3
cryptsetup luksOpen /dev/nvme0n1p3 DEBIANLUKS
<passphrase>
mkfs.btrfs -L DEBIAN /dev/mapper/DEBIANLUKS
mkdir /mnt/debian
mkdir /mnt/debianbtrfs
mount -t btrfs -o defaults,noatime,compress=lzo,autodefrag /dev/mapper/DEBIANLUKS /mnt/debianbtrfs
````

Mount main partition temporarily


Create subvolumes for rootfs, home, var and snapshots

    btrfs subvol create /mnt/debianbtrfs/debianroot
    btrfs subvol create /mnt/debianbtrfs/debianhome

### Re-mounting subvolumes as partitions to /mnt

    mount -t btrfs -o defaults,noatime,compress=lzo,autodefrag,subvol=debianroot /dev/mapper/DEBIANLUKS /mnt/debian/
    mkdir /mnt/debian/home
    mount -t btrfs -o defaults,noatime,compress=lzo,autodefrag,subvol=debianhome /dev/mapper/DEBIANLUKS /mnt/debian/home
    mkdir /mnt/debian/boot
    mount -o noatime,compress=zstd:1,subvol=@home /dev/mapper/cryptroot /mnt/home
    mount -o noatime,compress=zstd:1,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
    
## Base installation

### Bootstrapping a base-system

With all partitions mounted, run

    debootstrap  --arch=amd64 testing /mnt/debian http://deb.debian.org/debian

### Preparing the `chroot` environment

Copy the mounted file systems table

    cp /etc/mtab /mnt/etc/mtab

Bind the pseudo-filesystems for chroot

    mount -o bind /dev /mnt/debian/dev/
    mount -o bind /proc /mnt/debian/proc
    mount -o bind /sys /mnt/debian/sys

### Modifying fstab
````
nano /etc/fstab
````

````
UUID=952C-65CB	/boot	vfat	umask=0077	0 1
LABEL=DEBIAN	/	btrfs	defaults,noatime,compress=lzo,autodefrag,discard=async,subvol=debianroot	0 0
LABEL=DEBIAN	/home	btrfs	defaults,noatime,compress=lzo,autodefrag,discard=async,subvol=debianhome	0 0
shm        /dev/shm        tmpfs        nodev,nosuid,noexec  0 0
/dev/mapper/DEBIANSWAP	none	swap	sw,discard	0 0
````
Something like this should get you by. To find your blkid of you FAT32 EFI partition. Use blkid.


### Changing root to the new system

Copy over resolve.conf for internet.
````
sudo cp /etc/resolv.conf /mnt/debian/etc/resolv.conf

````


    sudo chroot /mnt/debian_root /bin/bash -l

## Configuration

### Setting `HOSTNAME` for your machine, this example uses `gtfo`

    echo 'Astro' > /etc/hostname
    echo '127.0.1.1 astro.localdomain astro' >> /etc/hosts

### Setting up `apt` sources

Update `/etc/apt/sources.list` to contain the following:

    deb https://deb.debian.org/debian testing main contrib non-free non-free-firmware
    apt update

### Setting timezone

    dpkg-reconfigure tzdata

Choose an appropriate region.

### Setting locale

    apt-get install locales
    dpkg-reconfigure locales

Choose `en_US.UTF-8` from the list of locales, or whatever is appropriate for you.

### Install your your desktop environment

    apt update
    apt-get install linux-image-amd64 linux-headers-amd64 firmware-linux-free firmware-misc-nonfree cryptsetup-initramfs console-setup sudo firmware-misc-nonfree  firmware-linux-nonfree firmware-intel-sound firmware-linux cryptsetup-suspend btrfs-progs intel-media-va-driver-non-free

    
Install whichever desktop environment you want, my prefernce is Gnome:
    
    apt install task-gnome-desktop ufw gufw
    
Other Desktop Environment (DE) options are:

    # task-cinnamon-desktop
    # task-gnome-desktop
    # task-kde-desktop
    # task-lxde-desktop
    # task-lxqt-desktop
    # task-mate-desktop
    # task-xfce-desktop
    
If you are installing this on a laptop:

    apt install task-laptop powertop gnome-power-manager linux-cpupower cpupower-gui laptop-mode-tools

## Creating users and groups

### Setting root password

    passwd

### Creating a non-root user

Create your main user.  Replace `meeas` with your username below.

    useradd duder -m -c "Duder" -s /bin/bash

Set password for your user

    passwd duder

Add user to sudo group

    usermod -aG sudo,adm,dialout,cdrom,floppy,audio,dip,video,plugdev,users,netdev,bluetooth,wireshark duder

## Setting up bootloader

### Installing bootloader utils

    apt install efi-grub btrfs-progs cryptsetup-initramfs
    
### Optional package for extra protection suspsended laptops

One attack vector for encrypted filesystems on laptops is to steal the crypto keys from RAM while the system is suspected. The keys are left in RAM to maintain access to the filesystem upon waking.  Linux has LUKS2 support to remove the cypto keys from memory to close this vulnerability, but it does require you to type your LUKS2 passphrase when you wake it from suspend.  If you want this extra protection:
    
    apt install cryptsetup-suspend

### Setting up encryption parameters

Use `blkid` to get the `UUID` of your encrypted partition, which is `/dev/nvme0n1p3` in this example

Create an entry in the `/etc/crypttab` file

    DEBIANSWAP UUID=a06d2110-b918-4e54-be85-f8a91e713ae0 /etc/swap.key luks
    DEBIANLUKS UUID=63f8dc5d-026d-4528-b85c-0a292adac2dd none luks

### Setup Grub

    grub-install --target=x86_64-efi --efi-directory=/boot
Modify /etc/defaults/grub to look like this depending on the UUID of your LUKS2 root partition.

````
GRUB_CMDLINE_LINUX_DEFAULT="crypt_root=UUID=63f8dc5d-026d-4528-b85c-0a292adac2dd quiet splash"
````
Make a grub.cfg file.
````
grub-mkconfig -o /boot/grub/grub.cfg
````
### Enable Intel Video Acceleration
I'm using a Thinkpad X1C 
````
nano /etc/modprobe.d/intel.conf
````
````
options i915 enable_guc=2
````
Write changes.

### Testing your new bootloader

Exit chroot

    exit

Unmount all mounted partitions

    umount -a

Reboot

    reboot

If your bootloader fails and does not allow you to boot into your new system, follow the same instructions to remount all volumes and see if you are having issues with BLKID's.



# Post installation

## Basic system configuration

### Console setup

Assuming you are now logged into your newly created useraccount (and not root)

    sudo dpkg-reconfigure console-setup

### Connecting to the internet

As long as you installed task-gnome-desktop as recommended in the previous steps, you can use the GUI to setup your Internet.

If you didn't install a DE like Gnome, you can use `nmtui` to setup your network at the command prompt.

### Install whatever other software you need or want.  I'd suggest this at a minimum:

    sudo apt install git curl wget

### If you have a fingerprint reader:
    
    sudo apt install libpam-fprintd
    sudo pam-auth-update    ### Enable fingerprint authentication
    fprintd-enroll
    fprintd-verify

### Debian Astro


I use Debian Astro. It takes up about 7 gigs.
````
apt-get install astro-all
````

Done.


