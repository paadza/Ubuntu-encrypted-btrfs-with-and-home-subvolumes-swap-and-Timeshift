# Ubuntu encrypted btrfs with @ and @home subvolumes, swap and Timeshift
- notes for my video [Ubuntu 25.04 (24.04) with encrypted btrfs and swap.. Timeshift snapshot.](https://www.youtube.com/watch?v=IRcJI9dQEmk&t=1183s)
- works from Ubuntu 24.04 LTS onwards, we do this installation using the newest 25.04

- the conversion is done by Torsten's script [btrfs-encrypt-root](https://gitlab.com/bronger/btrfs-encrypt-root/) - thank you!. I was using the latest version of the script, when shooting the video it was January's 2025 version.
- converts Ubuntu's just btrfs installation to encrypted btrfs installation with `@`and `@home` subvolume layout in live CD mode, IMMEDIATELLY after installation - no reboot!

## Ubuntu installation - 24.04 onwards, tested on 25.04
- boot Ubuntu 24.04 or later installation media, I booted 25.04
- select `Default selection`, A.K.A minimal...
- Disk setup..
  - select `Manual installation` - we need three partitions for the convert script - `/boot` `ext4`, `/boot/efi` `fat32` and `/` `btrfs`
    - click Free space
      - click `+`
        - Size
          - `1000 MB`
        - Used as
          - `ext4`
        - Mount point
          - `/boot`     
      - `/boot/efi` with FAT32 partition will be automatically created, `700 MB`
      - click `+` on the free space again
        - Size
          - `20 GB` - it will be extended later by the scripts - it wants to save time while encrypting
        - Used as
          - `btrfs`
        - Mount point
          - `/`   
      - click OK and click `Continue testing` on the `Ubuntu installed and ready to use` screen
      - `DO NOT REBOOT`, just quit the installer and run `terminal`

## converting btrfs to encrypted btrfs with the script btrfs-encrypt-root
- ..still without a reboot in the live installation environment
- go to gitlab and download the script from [btrfs-encrypt-root](https://gitlab.com/bronger/btrfs-encrypt-root/-/blob/master/btrfs-encrypt-root.sh?ref_type=heads)
- in terminal, make the script executable and run it. It will convert the non-encrypted btrfs partition to an encrypted LUKS volume and creates `@` and `@home` subvolumes AND moves the data from the btrfs filesystem to the newly created subvolumes then updates grub... great
- I am working with a VM, so for me, the drive is `/dev/vda`. You need to find yours using `lsblk`. You'll see for instance `nvme0n1` or `sda`..
```sh
sudo su
cd Downloads
chmod +x btrfs-encrypt-root.sh
lsblk
./btrfs-encrypt-root.sh vda3 vda1 vda2 --enlarge # the options order is the ROOT device (probably vda3), then /boot device (probably vda1) and the last one is EFI (probably vda2)
# we need to enter YES for confirmation AND encryption passphrase
```

## create more btrfs subvolumes - optional
- still in live CD installer, after the Ubuntu installation ...
```sh
lsblk
sudo cryptsetup open /dev/vda3 ubuntu # opens the LUKS volume to `ubuntu` mapper device - it's a name I gave it
lsblk # for me, it is vda3/ubuntu
sudo mount /dev/mapper/ubuntu /mnt
ls /mnt
sudo btrfs subvolume create /mnt/@log
sudo btrfs subvolume create /mnt/@cache
sudo btrfs subvolume create /mnt/@tmp
sudo btrfs subvolume create /mnt/@libvirt
sudo btrfs subvolume create /mnt/@swap
```
### move data to new subvolumes
```sh
sudo mv /mnt/@/var/cache/* /mnt/@cache/
sudo mv /mnt/@/var/log/* /mnt/@log/
```

## create a swap file on btrfs subvolume
```sh
sudo btrfs filesystem mkswapfile --size 4G /mnt/@swap/swapfile 
```

## edit /etc/fstab - optional 
- this is optional. Do it, if you created custom subvolumes or if you want to use some btrfs options like compression, noatime ....
`sudo nano /mnt/@/etc/fstab`
 - note: `compress` is applied to the whole filesystem  and only options in the first mounted subvolume will take effect..(https://btrfs.readthedocs.io/en/latest/Administration.html#btrfs-specific-mount-options)
```conf
# /etc/fstab
# created by the script
/dev/mapper/root /                      btrfs defaults,ssd,discard=async,noatime,space_cache=v2,compress=zstd:1,subvol=@ 0 0
/dev/mapper/root /home                  btrfs defaults,ssd,discard=async,noatime,space_cache=v2,compress=zstd:1,subvol=@home 0 0

# created by me
/dev/mapper/root /var/log               btrfs defaults,ssd,discard=async,noatime,space_cache=v2,compress=zstd:1,subvol=@log 0 0
/dev/mapper/root /var/cache             btrfs defaults,ssd,discard=async,noatime,space_cache=v2,compress=zstd:1,subvol=@cache 0 0
/dev/mapper/root /tmp                   btrfs defaults,ssd,discard=async,noatime,space_cache=v2,compress=zstd:1,subvol=@tmp 0 0
/dev/mapper/root /var/lib/libvirt/      btrfs defaults,ssd,discard=async,noatime,space_cache=v2,subvol=@libvirt 0 0
/dev/mapper/root /swap                  btrfs defaults,subvol=@swap 0 0
/swap/swapfile   none                   swap  defaults,ssd,discard=async,noatime 0 0
```

## unmout and close luks
```sh
sudo umount /mnt
sudo cryptsetup close ubuntu
```
## reboot to finish the installation

## OPTIONAL - disable copy-on-write for disk images if using libvirt virtualization 
- if you are planning to use KVM virtualization, you can prevent fragmentation of virtual disk images by turning off the copy-on-write feature 
```sh
sudo chattr +C /var/lib/libvirt  #/path/to/directory - will disable copy-on-write for that directory
```

---

# On a freshly installed Ubuntu system

## checking btrfs mounted options
- in your new Ubuntu system, you can check, if all those btrfs features you enabled in `/etc/fstab` are in use. You'll see probably more features than you configured, those are the default ones.
```sh
findmnt -t btrfs  # checking btrfs mounted options
```

## timeshift install
```sh
sudo apt install timeshift
```
- choose `btrfs` and the main disk for snapshot destination
- go to `Settings\Users` and select `Include @home subvolume in backups`
- if you want to see what is inside a snapshot, just select a snapshot and click `browse`. If you wanna do it in a terminal, run the `timeshift` program and do `ls /var/timeshift/xxxx/backup/`. 
