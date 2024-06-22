# VMware Arch Linux Set-Up 
 
## Overview
Hi! This will be a full walktrough with a detailed explanation about how to **install and configure** an Arch Linux distribution from zero. I will install it on VMware, so it will differ a bit from a bare metal installation or a Virtual Box one. Once Arch is installed, I am going to configure it and start creating and editing my dotfiles, for this installation I will use [Qtile](https://qtile.org/) as my tiling window manager.

##  Prerequisites
- Familiarity with bash and solid understanding of the Linux structure.
  
- Understand the hypervisor's options to set-up a virtual machine.
  
- Having already tried to rice a Linux distro (this will be much easier this way).

## Vm set-up

1. Open **VMware Workstaiton Pro**.
   
2. Go to *File > New Virtual Machine*, or do *CTRL + N*.
   
3. On the **New Virtual Machine Wizzard**, select the *Custom (advanced)* option.
   
4. Click *Next >*.
   
5. On the **Guest Operating System Installation**, you can select either *Installer disc image file (iso)* or *I will install the operating system later*, this won't affect further stepts. Click *Next >*.
   
6. Select *Linux* on the **Guest Operating System** panel, for the next option, check out the current Arch kernel version on [releases](https://archlinux.org/releng/releases/), depending on the first number, select on Vmware the *Other Linux y.x Kernel 64-bit*. Selecting this correctly will optimize Vmware for that kernel version (if you select another option, Arch might run slower or with problems). Click *Next >*.
   
7.  Set a name for your VM and choose the location folder where all of your VM files necessary for it to work will be stored, then click *Next >*.
   
8.  To choose the number of processors and their cores we have to understand that *number of processors* correspond to the physical CPU threads and *Cores per processor* correspond to the physical CPU cores. A vCPU is only the way that the hypervisor will share resources with our CPU. Take into account that the number of cores for the guest needs to be less than the available on the system. Consider as well that VMware will not use all the assigned cores at runtime, it will wait for free resources, so when assigning the cores to our VM, do not assign too much to avoid experiencing bad performance on the host.  
As I have a Ryzen 5 processor with 6 cores and 12 threads, I will assign my Vm 1 core and 4 processors, as this will make me possible to set-up more VMs and make this environment more scalable.

9. Select the amount of RAM, as I have 32Gb fo RAM, I will assign 8Gb to the VM.
10. For the **Network Connection** panel, select *Use network address translation (NAT)*.
    
11. The SCSI controller refers to 'ports' that allows the pc to comunicate with hardware such as hard drives, disk drives, etc (refer to [this](https://www.reddit.com/r/vmware/comments/v2qzc5/comment/iau895o/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button) link to have a better comparison between options). I will select the *Paravirtualized SCSIl* as it has native support in most Linux releases, it also got grater troughput and lower CPU utilization, then create a SCSI disk on the following option.
    
12. on the **Disk** section, select *Create a new virtual disk* to have an easier set-up to use, as the disk's space I will assign 80Gb.

13. Click *Finish*.

Once the VM is created, we will adjust some settings as follows:

1. Click on *Edit virtual machine settings*.
   
2. Go to *Display* and check *Accelerate 3D graphics* to speed up a bit the machine's performance.
   
3. Go to *CD/DVD (SATA)*, check the *Use ISO image file:* and select the Arch Linux ISO.

4. Go to the **Options** tab, then select **Advanced** and change the *Firmware type* and select *UEFI* and check *Disable side channel mitigations for Hyper-V enabled hosts*.

5. Boot up the machine.


## Arch Linux Configuration

*based on [Arch Linux wiki](https://wiki.archlinux.org/title/Installation_guide)*

Verify the system clock is synchronized:

```bash
timedatectl
```

If not synchronized, list the timezones and select the apropiate for you:

```bash 
timedatectl list-timezones | grep {Your zone}
timedatectl set-timezone {TIMEZONE}
```

List devices (disks) on the computer's storage device, look for the one that has the size you assigned to the VM:

```bash
lsblk
```

As we selected UEFI firmware before, and as I have UEFI firmware on my pc, we will need to use a UEFI partition. This is important because it contains all the bootloader files (a bootloader such as grup is a software that will load the selected OS files into memory and execute them).

Start partitioning the disk (based on the arch wiki recomendations):

```bash
cfdisk
```

| Mount point | Partition | Suggested size |
| --- | --- | --- |
| /boot | /dev/efi_partition | 1Gb |
| swap | /dev/swap_partition | At least 4Gb |
| / | /dev/root_partition | Remainder |

**Note:** The *swap* partition is like a 'RAM-backup' partition, or a space where once you shut-down your pc, RAM contents that were being executed at that moment will be saved on the swap partition to be able to use them when booting up again.

Format the created partitions:

```bash
mkfs.btrfs /dev/root_partition #I'll go for btrfs as my file system
mkswap /dev/swap_partiton
mkfs.fat -F 32 /dev/efi_partition
```

Mount the created volums to access to them from the live session:

```bash
mount /dev/root_partition /mnt
mount --mkdir /dev/efi_partition /mnt/boot #mount the volume on a new created directory
swapon /dev/swap_partition #enables the swap partition
```

**Note:** To install the Gnu/Linux kernel and other packages is necessary to have *mirror servers* defined at /etc/pacman.d/mirrorlist, packages will be downloaded from the mirror servers.

Install the kernel and other useful packages:

```bash
pacstrap /mnt base linux base-devel linux-firmware amd-ucode grub efibootmgr man-db man-pages net-tools dhcp dhcpcd iproute2 vim git xorg networkmanager firefox qtile xterm ly kitty rofi open-vm-tools gtkmm3 xf86-input-vmmouse xf86-video-vmware mesa
```

`man-db man-pages net-tools dhcp dhcpcd iproute2 vim git xorg networkmanager firefox qtile xterm ly kitty rofi open-vm-tools gtkmm3 xf86-input-vmmouse xf86-video-vmware mesa` All this packages are just **extra**. some of them correspond to the qtile set-up of this walktrough while some others correspond to packages that the display server Xorg needs to be able to run on a vm. Other packages also will enable the Vmware Tools. 

generate the *fstab* file, it will tell the system how to mount each partition during the system's boot process.

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Chroot into the /mnt mount point:

```bash
arch-chroot /mnt
```

Set the timezone (again):

```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
```

Generate 'locales', used to define language and cultural settings, go first to edit the /etc/locale.gen and uncomment en_US.UTF-8:

```bash
locale-gen

vim /etc/locale.conf
-----
LANG=en_US.UTF-8
```

Create a hostname and enable the networkmanager service:

```bash
vim /etc/hostname
systemctl enable NetworkManager

#configure basic entries on the /etc/hosts
127.0.0.1   localhost
::1         localhost
```

install the grub bootloader, exit the live session, umount the mount points and reboot:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot
grub-mkconfig -o /boot/grub/grub.cfg
exit
umount -R /mnt
reboot
```

Once inside Arch Linux, assign a password for the root user and create a new user:

```bash
passwd
useradd -m user
passwd user
usermod -aG wheel user
```

enable the Vmware Tools:

```bash
sudo systemctl enable vmtoolsd.service
```

Make sure you type **enable** as **start** will only run the vmware tools while the session is open, once you shutdown your machine, the service will not run again until you start it.

Edit the /etc/sudoers to run with root privileges as our new user by using sudo:

```bash
# uncomment the following line inside the file
# %wheel ALL=(ALL) ALL
```

## Qtile Rice basics

Path to the config file: `~/.config/qtile/config.py`

To make the configuration process easier, I will use Vscode to edit the config file, the following table are the shortcuts that will help us to navigate into qtile, keep in mind that in order to fully customize this wm you need a bit of Python knoweledge:

| Key | Action |
|--|--|
| mod + return  | launch terminal |
| mod + k | next window |
| mod + j | previous window |
| mod + w | kill window |
| mod + [asdfuiop] | go to workspace [i] |
| mod + ctrl + r | restart qtile |
| mod + ctrl + q | logout |

The *launch terminal* key is defined in the following line:

```python
Key([mod], "Return", lazy.spawn(terminal)),
```

Change the value of the *terminal* variable to the name of the terminal you'd like to launch, in my case, it will be Kitty:

```python
terminal = "kitty"

# ...

Key([mod], "Return", lazy.spawn(terminal)),
```
