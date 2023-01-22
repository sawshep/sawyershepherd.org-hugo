---
title: "Hibernating to an Encrypted Swapfile on BTRFS with NixOS"
date: 2023-01-15T15:39:26-05:00
---

**Note that BTRFS swapfiles across multiple devices (pools,
RAIDs) are not supported[^swapfile_on_raid].**

BTRFS on Linux kernels before version 5.0 can result in
filesystem corruption[^arch_wiki_btrfs]. From the Arch Wiki:
    
>Btrfs on Linux kernel before version 5.0 does
>not support swap files. Failure to heed this warning may
>result in file system corruption. While a swap file may
>be used on Btrfs when mounted through a loop device,
>this will result in severely degraded swap performance.

You must make a BTRFS subvolume and disable copy-on-write:

```sh
sudo btrfs subvolume create /swap
sudo chattr +C /swap
```

Create a swapfile of your chosen size. I chose the size of
my memory +4kB for swap headers and such

```sh
sudo fallocate --length 15698384KiB /swap/swapfile
```

Format the swapfile

```sh
sudo mkswap /swap/swapfile
```

Enable the swapfile

```sh
sudo swapon /swap/swapfile
```

Add the following to your NixOS hardware configuration
file[^nixos_setup_swapfile]:

```nixos
fileSystems."/swap" = {
  device = "/dev/disk/by-uuid/[uuid-of-your-btrfs-partition]";
  fsType = "btrfs";
  options = [ "subvol=swap" "noatime" ];
};

swapDevices = [ { device = "/swap/swapfile"; } ];
```

The UUID for the swap subvolme is the exact same as the UUID
for the whole BTRFS partition. You don't need to specify a
`subvol` option to mount the root BTRFS partition.

Next, we need to find some values to add to our kernel boot
parameters. Specifically, we need the UUID of the swap file
and the swap file offest. Find the UUID of the swap device
with:

```sh
findmnt -no UUID -T /swap/swapfile
```

Note that this UUID should point to your mounted, mapped
BTRFS filesystem. In `/dev/disk/by-uuid/[...]`, it's probably a
symlink to /dev/dm-*X*.

Now we need to find the physical swap file offset. In
filesystems other than BTRFS, this is relatively easy, as we
can use `filefrag`. Unfortunately, filefrag does not report
the real physical offset of files on BTRFS systems (this is
intended)[^swapfile_hibernation_btrfs]. Fortunately, a user called Osandov has recognized
this problem and published a program to find the physical
offset of a file on BTRFS
[here](https://raw.githubusercontent.com/osandov/osandov-linux/master/scripts/btrfs_map_physical.c).
Download the program and compile it:

```sh
wget https://raw.githubusercontent.com/osandov/osandov-linux/master/scripts/btrfs_map_physical.c
gcc btrfs_map_physical.c -o btrfs_map_physical
```

Now use the program to find the offset:

```sh
sudo ./btrfs_map_physical /swap/swapfile
```

Find the first physical offset provided and divide it by the
page size of your system. Usually this is 4096, but you can
also find it with:

```sh
getconf PAGESIZE
```

Add this to your NixOS hardware configuration:

```nixos
boot.resumeDevice = "/dev/disk/by-uuid/[swap_device_uuid]"
boot.kernelParams = [ "resume_offset=[offset]" ];
```

And finally, rebuild and switch:
`sudo nixos-rebuild switch`

If you have any issues with your device not shutting off
after hibernating, add this to your NixOS config:
```nix
systemd.sleep.extraConfig = ''
    [Sleep]
    HibernateMode=shutdown
'';
```

[^swapfile_on_raid]: https://askubuntu.com/questions/1206157/can-i-have-a-swapfile-on-btrfs
[^arch_wiki_btrfs]: https://wiki.archlinux.org/title/btrfs#Swap_file
[^nixos_setup_swapfile]: https://discourse.nixos.org/t/how-do-i-set-up-a-swap-file/8323/7
[^swapfile_hibernation_btrfs]: https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation_into_swap_file_on_Btrfs
