## My NixOS Laptop w/ an Encrypted Amnesia-tic ZFS Filesystem

*Note: I'm very aware of how old and dusty my computer looks, get off my lawn*

I have had a personal Lenovo T480s running around the house for quite a while. Traditionally I've run Ubuntu on it, but it's also run NixOS, Fedora, and a few other various setups over the years. I've decided after spending quite a bit of time using my work machine (16" MBP M1 Pro) for personal and work that I would revive the Lenovo and set back up a NixOS w/ i3 configuration. The primary driver behind this? The keyboard is **so** much better to type on.

I will be setting the computer up to meet the requirements I list below. I'm going to assume you will have read [Erase your darlings](https://grahamc.com/blog/erase-your-darlings) and [NixOS on Encrypted ZFS](https://blog.lazkani.io/posts/nixos-on-encrypted-zfs/) before attempting my guide, since I mostly merged their two guides and added in some of my own flair. 

My requirements:

- Encrypted ZFS root partition
- Erase on every boot, a la [Erase your darlings](https://grahamc.com/blog/erase-your-darlings)
- i3 Window Manager
- Functioning hardware, including touchscreen, bluetooth, trackpad
- Sync of config with private Gitlab [Covered in future write-up]
- Borgbase backups [Covered in future write-up]
- Wireguard to home [Covered in future write-up]

## Base Install

Let's get the OS installed on a NVMe with the `/boot` partition formatted with `VFAT` and then an encrypted `ZFS` partition with a bunch of datasets.

#### Boot from USB 

First things first, go ahead and head out to [nixos.org](nixos.org) and download the most recent Minimal ISO image (which is `21.11` as of the time of this writing).

Go ahead and image that ISO to a thumb drive using `dd` or whatever tool you're comfortable with and then boot your laptop from it.

#### Partition Drive

I like to work on partitioning the hard drive first. In my case, it's a 500GB NVMe. The next few steps will require knowing the vendor id:

```
ls -la /dev/disk/by-id
```

Should result in a list of vendor IDs and the symlinks. You'll most likely be looking for one that links directly to the root of your NVMe and doesn't include partition numbers, in my case `../../nvme0n1`.

```
...
nvme-SAMSUNG_PARTNUMBER_SERIAL -> ../../nvme0n1
...
```

Now, let's put the two partitions on the drive:

```
sgdisk -n3:1M:+1024M -t3:EF00 /dev/disk/by-id/nvme-SAMSUNG_PARTNUMBER_SERIAL
sgdisk -n1:0:0 -t1:BF01 /dev/disk/by-id/nvme-SAMSUNG_PARTNUMBER_SERIAL
```

The first `sgdisk` command creates the `EFI` 1GB partition for `/boot` while the second creates the `ZFS` partition using the rest of the drive. Feel free to adapt this to what works best for you.

#### Format Boot Partition with VFAT Filesystem

Let's get that `/boot` partition formatted (you'll note I add `part3` to the drive id, as that's the partition number for boot from the previous section):

```
mkfs.vfat /dev/disk/by-id/nvme-SAMSUNG_PARTNUMBER_SERIAL-part3
```

#### Setup Luks for ZFS partition

Time to setup `luks` for the ZFS root partition:

```
cryptsetup luksFormat /dev/disk/by-id/nvme-SAMSUNG_PARTNUMBER_SERIAL-part1
```

Follow the prompts to set a passphrase, then mount the encrypted filesystem:

```
cryptsetup open --type luks /dev/disk/by-id/nvme-SAMSUNG_PARTNUMBER_SERIAL-part1 crypt
```

You'll see a mount of the filesystem here: `/dev/mapper/crypt`.

#### Configure ZFS Pool and Datasets

Time to hop into the pool and configure ZFS:

```
zpool create -O mountpoint=none rpool /dev/mapper/crypt
```

Now we'll configure the datasets for ZFS. First is the most important one, `root`, which will have a blank snapshot taken of it. This is important, because we'll use a section in the `configuration.nix` file to restore that snapshot on boot.

```
# root zfs dataset
zfs create -p -o mountpoint=legacy rpool/local/root

# blank snapshot of root
zfs snapshot rpool/local/root@blank

# mount root
mount -t zfs rpool/local/root /mnt
```

Now then, let's create the mountpoints for the rest of our datasets, as well as the datasets themselves:

```
mkdir -p /mnt/{boot,nix,home,persist}

# boot partition
mount /dev/nvme0n1p3 /mnt/boot

# nix dataset
zfs create -p -o mountpoint=legacy rpool/local/nix
mount -t zfs rpool/local/nix /mnt/nix

# home
zfs create -p -o mountpoint=legacy rpool/safe/home
mount -t zfs rpool/safe/home /mnt/home

# persist
zfs create -p -o mountpoint=legacy rpool/safe/persist
mount -t zfs rpool/safe/persist /mnt/persist
```

A lot happened in that last code block, most of which I'll cover later in more detail, but suffice to say you have a bunch of mountpoints that the nix configuration generator will take into account. 

#### Generate NixOS Configuration

This is pretty straightforward, but you want a nice configuration to start your modifications from.

```
nixos-generate-config --root /mnt
```

You'll see two `.nix` files populate in `/mnt/etc/nixos`.

### Modify configuration.nix

You'll need a few configuration changes to meet our goal of being amnesia-tic, so let's roll up our sleeves and make some modifications to`/mnt/etc/nixos/configuration.nix`:

#### Add lib and etc packages

We'll need `lib` to run the command after boot to restore to the `root@blank` snapshot, as well as `etc` to create a symlink to preserve the contents of `/etc/nixos` after reboot.

```
{ config, pkgs, lib, etc, ... }:
```

#### Configure Grub and Boot

Add the following section in after `boot.loader.efi.canTouchEfiVariables`, make sure to replace the UUID of your large ZFS partition (`ls blkid /dev/nvme0n1p1`).

```
boot.supportedFilesystems = [ "zfs" ];
  boot.loader.grub = {
    enable = true;
    version = 2;
    device = "nodev";
    efiSupport = true;
    enableCryptodisk = true;
  };

  boot.initrd.luks.devices = {
    root = {
      device = "/dev/disk/by-uuid/PUTYOURUUIDHERE";
      preLVM = true;
    };
  };
```

Next, we put in the configuration section to rollback to the `root@blank` snapshot, right below the previous section.

```
  boot.initrd.postDeviceCommands = lib.mkAfter ''
    zfs rollback -r rpool/local/root@blank
  '';
```

You'll also want to set a few other miscellaneous items:

```
networking.hostId = "8DIGITHEXVALUE";
networking.hostName = "yournamehere";
```

### Make NixOS configurations persistent

If you're not careful, you'll have to go through some gymnastics to get back your `configuration.nix` files without making them persistent. So, we'll want to use the `etc` module to symlink the directory to the `persist` dataset.

First:

```
mkdir -p /mnt/persist/etc/nixos
```

Then add the following to `/mnt/etc/nixos/configuration.nix`:

```
  environment.etc."nixos" = {
    source = "/persist/etc/nixos/";
  };
```

Copy your `*.nix` files manually to the target nixos directory (otherwise they won't persist through the install and symlink:

```
cp -r /mnt/etc/nixos/*.nix /mnt/persist/etc/nixos/
```

### Nix-install

At this point you should be ready to install the OS and get moving! 

```
nixos-install
sudo reboot
```

### Fin

So that's all there is to it! I've used this guide a few times now, including to get my workstation up and going with this setup too. Look forward to more adventures in NixOS, including:

- NixOS w/ Encrypted ZFS on Qnap 
- NixOS w/ i3wm on PinePhone

<3