# Archlinux + encrypted btrfs + snapshots + systemd bootloader with XBOOTLDR

Installation guide is based on QEMU/KVM x86_64 emulation with UEFI firmware, but it's applicable to real hardware.

## Manual install

---

### Boot from livecd 
Check [Installation guide](https://wiki.archlinux.org/title/Installation_guide) for details

### Prepare partitions

* Check connected disks
    ```
    root@archiso ~ # lsblk -f
    NAME  FSTYPE FSVER   LABEL       UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
    loop0 squash 4.0                                                            0   100% /run/archiso/airootfs
    sr0   iso966 Joliet  ARCH_202208 2022-08-05-11-09-34-00                     0   100% /run/archiso/bootmnt
    vda    
    ```
* Choose disk to install Arch (`vda` in current case) and make partitioning
    ```
    root@archiso ~ # fdisk /dev/vda
    
    Welcome to fdisk (util-linux 2.38).
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.
    
    Device does not contain a recognized partition table.
    Created a new DOS disklabel with disk identifier 0x8a53ea78.
    ```
* Create GPT (GUID Partition Table), which is required to support EFI
    ```
    Command (m for help): g
    Created a new GPT disklabel (GUID: 717C71CF-2183-5B41-ACEA-5D79C34CA76E).
    ```
* Add a new partition for `/efi` of `EFI System` type
    ```
    Command (m for help): n
    Partition number (1-128, default 1): 1
    First sector (2048-41943006, default 2048):
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-41943006, default 41940991): +100M
    
    Created a new partition 1 of type 'Linux filesystem' and of size 100 MiB.
    
    Command (m for help): t
    Selected partition 1
    Partition type or alias (type L to list all): 1
    Changed type of partition 'Linux filesystem' to 'EFI System'.
    ```
* Add a `Linux extended boot` for `/boot` 
    ```
    Command (m for help): n
    Partition number (2-128, default 2): 2
    First sector (206848-41943006, default 206848):
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (206848-41943006, default 41940991): +1G
    
    Created a new partition 2 of type 'Linux filesystem' and of size 1 GiB.
    
    Command (m for help): t
    Partition number (1,2, default 2): 2
    Partition type or alias (type L to list all): 136
    
    Changed type of partition 'Linux filesystem' to 'Linux extended boot'.
    ```
* Allocate the rest of space for Arch Linux installation
    ```
    Command (m for help): n
    Partition number (3-128, default 3): 3
    First sector (2304000-41943006, default 2304000):
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (2304000-41943006, default 41940991):
    
    Created a new partition 3 of type 'Linux filesystem' and of size 18.9 GiB.
    ```
* Write changes
    ```
    Command (m for help): w
    The partition table has been altered.
    Calling ioctl() to re-read partition table.
    Syncing disks.
    ```
* Make `FAT32` filesystems for `/efi` and `/boot`
    ```
    root@archiso ~ # mkfs.fat -F32 -n ESP /dev/vda1
    mkfs.fatmkfs.fat 4.2 (2021-01-31)
    ```
    ```
    root@archiso ~ # mkfs.fat -F32 -n BOOT /dev/vda1
    mkfs.fatmkfs.fat 4.2 (2021-01-31)
    ```
* Encrypt Linux partition with `luks`
    ```
    root@archiso ~ # cryptsetup luksFormat /dev/vda3
    WARNING!
    ========
    This will overwrite data on /dev/vda3 irrevocably.
    
    Are you sure? (Type 'yes' in capital letters): YES
    Enter passphrase for /dev/vda3:
    Verify passphrase:
    cryptsetup luksFormat /dev/vda3  18.83s user 0.74s system 127% cpu 15.381 total
    ```
* Open encrypted partition as `archcrypto` for further actions
    ```
    root@archiso ~ # cryptsetup open /dev/vda3 archcrypto
    passphrase for /dev/vda3:
    cryptsetup open /dev/vda3 archcrypto  6.46s user 0.09s system 170% cpu 3.840 total
    ```
* Format `archcrypto` with `btrfs`
    ```
    root@archiso ~ # mkfs.btrfs -L ARCH /dev/mapper/archcrypto
    mkfs.btrfsbtrfs-progs v5.18.1
    See http://btrfs.wiki.kernel.org for more information.
    
    NOTE: several default settings have changed in version 5.15, please make sure
          this does not affect your deployments:
          - DUP for metadata (-m dup)
          - enabled no-holes (-O no-holes)
          - enabled free-space-tree (-R free-space-tree)
    
    Label:              ARCH
    UUID:               51c6913e-103e-4690-9487-6adfd416bd77
    Node size:          16384
    Sector size:        4096
    Filesystem size:    18.88GiB
    Block group profiles:
      Data:             single            8.00MiB
      Metadata:         DUP             256.00MiB
      System:           DUP               8.00MiB
    SSD detected:       no
    Zoned device:       no
    Incompat features:  extref, skinny-metadata, no-holes
    Runtime features:   free-space-tree
    Checksum:           crc32c
    Number of devices:  1
    Devices:
       ID        SIZE  PATH
        1    18.88GiB  /dev/mapper/archcrypto
    ```
* Mount `btrfs` filesystem and create subvolumes for `/`, `/home` and `/snapshots`
    ```
    root@archiso ~ # mount /dev/mapper/archcrypto /mnt
    ```
    ```
    root@archiso ~ # btrfs subvolume create /mnt/@
    btrfsCreate subvolume '/mnt/@'
    ```
    ```
    root@archiso ~ # btrfs subvolume create /mnt/@home
    btrfsCreate subvolume '/mnt/@home'
    ```
    ```
    root@archiso ~ # btrfs subvolume create /mnt/@snapshots
    btrfsCreate subvolume '/mnt/@snapshots'
    ```
* Umount `btrfs` filesystem and mount each of subvolumes separately
    ```
    root@archiso ~ # mount -o subvol=@ /dev/mapper/archcrypto /mnt
    ```
    ```
    root@archiso ~ # mkdir -p /mnt/{.snapshots,boot,efi,home}
    ```
    ```
    root@archiso ~ # mount -o subvol=@home /dev/mapper/archcrypto /mnt/home
    ```
    ```
    root@archiso ~ # mount -o subvol=@snapshots /dev/mapper/archcrypto /mnt/.snapshots
    ```
    ```
    root@archiso ~ # mount /dev/vda1 /mnt/efi
    ```
    ```
    root@archiso ~ # mount /dev/vda2 /mnt/boot
    ```
* Install core packages
    ```
    pacstrap /mnt base linux linux-firmware vim networkmanager btrfs-progs
    ```
    Make a cup of coffee :)


* Generate `/etc/fstab`
    ```
    root@archiso ~ # genfstab -U /mnt >> /mnt/etc/fstab
    ```
    Check if everything is correct
    ```
    root@archiso ~ # cat /mnt/etc/fstab
    # Static information about the filesystems.
    # See fstab(5) for details.
    
    # <file system> <dir> <type> <options> <dump> <pass>
    # /dev/mapper/archcrypto LABEL=ARCH
    UUID=51c6913e-103e-4690-9487-6adfd416bd77       /               btrfs
            rw,relatime,space_cache=v2,subvolid=256,subvol=/@       0 0
    
    # /dev/vda1 LABEL=ESP
    UUID=2E9F-0C4E          /efi            vfat            rw,relatime,fm
    ask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,
    errors=remount-ro       0 2
    
    # /dev/mapper/archcrypto LABEL=ARCH
    UUID=51c6913e-103e-4690-9487-6adfd416bd77       /home           btrfs         rw,relatime,space_cache=v2,subvolid=257,subvol=/@home   0 0
    
    # /dev/vda2 LABEL=BOOT
    UUID=2EEC-CF10          /boot           vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro       0 2
    ```
    
* Chroot to `/mnt`
    ```
    root@archiso ~ # arch-chroot /mnt
    ```
* Setup localization
    ```
    [root@archiso /]# ln -sf /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
    ```
    ```
    [root@archiso /]# echo "en_US ISO-8859-1" >> /etc/locale.gen
    ```
    ```
    [root@archiso /]# locale-gen
    Generating locales...
    en_US.ISO-8859-1... done
    Generation complete.
    ```
* Add hostname
    ```
    [root@archiso /]# echo "cryptoarch" >> /etc/hostname
    ```
* Edit `/etc/mkinitcpio.conf` to support encryption by adding `encrypt` hook before `filesystems`
    ```
    HOOKS=(base udev autodetect modconf block encrypt filesystems keyboard fsck)   
    ```
  Then, generate `initramfs` image 
    ```
    [root@archiso /]# mkinitcpio -p linux
    ```
* Set root password
    ```
    [root@archiso /]# passwd
    New password:
    Retype new password:
    passwd: password updated successfully
    ```
* Install `systemd` bootloader
    ```
    [root@archiso /]# bootctl --esp-path=/efi --boot-path=/boot install
    ```
* Add bootloader configuration
    ```
    [root@archiso /]# echo "title   Arch Linux" >> /boot/loader/entries/arch.conf
    [root@archiso /]# echo "linux   /vmlinuz-linux" >> /boot/loader/entries/arch.conf
    [root@archiso /]# echo "initrd  /initramfs-linux.img" >> /boot/loader/entries/arch.conf
    ```
  Ensure that `/` from encrypted partition is loaded
    ```
    [root@archiso /]# echo "options cryptdevice=UUID=< /dev/vda3 UUID>:archcrypto root=UUID=< /dev/mapper/archcrypto UUID > rootflags=subvol=@ rw quiet" >> /boot/loader/entries/arch.conf
    ```
  Set `arch.conf` as default entry
    ```
    [root@archiso /]# echo "default arch.conf" >> /efi/loader/loader.conf
    [root@archiso /]# echo "timeout  4" >> /efi/loader/loader.conf
    [root@archiso /]# echo "console-mode max" >> /efi/loader/loader.conf
    [root@archiso /]# echo "editor   no" >> /efi/loader/loader.conf
    ```
* Make your first snapshot of `/home` partition 
    ```
    [root@archiso /]# btrfs subvolume snapshot /home /.snapsbots/home-$(date "+%Y%m%d%H%M%S")
    ```
* Reboot and enjoy :)


## Automatic install

TODO