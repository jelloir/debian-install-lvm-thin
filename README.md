# Debian Install LVM Thin-Provision

## Description
Guide to installing Debian onto and thin-provisioned lvm pool.

### Kernel Modules
Extract the following modules from the latest linux-image referenced at https://packages.debian.org/buster/linux-image-amd64 or you get them from another running Debian system.

    dm-bio-prison.ko
    dm-persistent-data.ko
    dm-thin-pool.ko

They must be the from the same kernel version as the installer kernel version or they will not load.

### thin_check
Extract the following file from https://packages.debian.org/buster/thin-provisioning-tools

    thin_check

Place the files on media to copy them from during the install.

### lvm2thin
Create the following bash script and save it to your media with the name "lvm2thin"

```shell
#!/bin/sh

PREREQ="lvm2"

prereqs()
{
	echo "$PREREQ"
}

case $1 in
prereqs)
	prereqs
	exit 0
	;;
esac

. /usr/share/initramfs-tools/hook-functions

copy_exec /usr/sbin/thin_check
manual_add_modules dm_thin_pool
```
### Install
Boot the Debian installer and proceed up until you have reached **partition disks** and then switch to another console e.g. `ALT-F2`
mount the media with the files you've copied.  I'm using `/usb` and the files are in a folder called thin.  Then copy the files into the installer kernel modules path.

    mount /dev/sdb1 /usb
    cp /usb/thin/*.ko /lib/modules/4.9.0-3-amd64/

Then probe all modules

    depmod -a

Copy `thin_check` into `/usr/sbin/` and make executeable

    cp /usb/thin/thin_check /usr/sbin/
    chmod +x /usr/sbin/thin_check

Use fdisk to create 3 partitions (assumes UEFI so adjust for your needs).

    Device     Boot   Start      End  Sectors  Size Id Type
    /dev/sda1          2048  1050623  1048576  512M ef EFI (FAT-12/16/32)
    /dev/sda2       1050624  2099199  1048576  512M 83 Linux
    /dev/sda3       2099200 25165823 23066624   11G 8e Linux LVM

Then proceed to create our LVM structure.

pvcreate /dev/sda3
vgcreate vg0drg56 /dev/sda3

Create a contiguous swap partition first on LVM

lvcreate --name swap --size 1GiB --contiguous y vg0drg56

Then create the thin pool for our thin volumes with a decent pool meta data size - you never ever want to run out of this!!

lvcreate --thin-pool tpool0 --extents +100%FREE --poolmetadatasize 256M vg0drg56

Then create the remaining as LVM thin.

lvcreate --thin --name root --virtualsize 20GiB vg0drg56/tpool0
lvcreate --thin --name var_cache --virtualsize 4GiB vg0drg56/tpool0
lvcreate --thin --name var_log --virtualsize 4GiB vg0drg56/tpool0
lvcreate --thin --name home --virtualsize 100GiB vg0drg56/tpool0

Note that you can over provision the storage - cool hey!

Now that LVM is setup go back to the UI and format the partitions including the EFI partion and the 512M volume after it which should be mounted as /boot


LVM VG vg0drg56, LV home - 107.4 GB Linux device-mapper (thin)
>     #1           107.4 GB     f  ext4    /home
LVM VG vg0drg56, LV root - 21.5 GB Linux device-mapper (thin)
>     #1            21.5 GB     f  ext4    /
LVM VG vg0drg56, LV swap - 1.1 GB Linux device-mapper (linear)
>     #1             1.1 GB     f  swap    swap
LVM VG vg0drg56, LV tmp - 4.3 GB Linux device-mapper (thin)
>     #1             4.3 GB     f  ext4    /tmp
LVM VG vg0drg56, LV tpool0 - 10.2 GB Linux device-mapper (linear)
>     #1            10.2 GB
LVM VG vg0drg56, LV tpool0-tpool - 10.2 GB Linux device-mapper (thin-pool)
>     #1            10.2 GB
LVM VG vg0drg56, LV tpool0_tdata - 10.2 GB Linux device-mapper (linear)
>     #1            10.2 GB
LVM VG vg0drg56, LV tpool0_tmeta - 268.4 MB Linux device-mapper (linear)
>     #1           268.4 MB
LVM VG vg0drg56, LV var_cache - 4.3 GB Linux device-mapper (thin)
>     #1             4.3 GB     f  ext4    /var/cache
LVM VG vg0drg56, LV var_log - 4.3 GB Linux device-mapper (thin)
>     #1             4.3 GB     f  ext4    /var/log
SCSI1 (0,0,0) (sda) - 12.9 GB ATA VBOX HARDDISK
>     #1  primary  536.9 MB  B  f  ESP
>     #2  primary  536.9 MB     f  ext4    /boot
>     #3  primary   11.8 GB     K  lvm

Complete the install process.

After it has installed the base system you can copy lvm2thin into place so the required lvm thin components are included in initramfs 

cp /usb/thin/lvm2thin /target/etc/initramfs-tools/hooks/

Then chroot to the install to install thin-provisioning-tools and update the initramfs

chroot /target /bin/bash
apt-get install thin-provisioning-tools
update-initramfs -u

Exit the chroot and complete the install to boot to your thin provision setup yeah!

after you reboot have a look at your slender setup!

sudo lvs -a
  LV              VG       Attr       LSize   Pool   Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home            vg0drg56 Vwi-aotz-- 100.00g tpool0        2.07
  [lvol0_pmspare] vg0drg56 ewi------- 256.00m
  root            vg0drg56 Vwi-aotz--  20.00g tpool0        7.08
  swap            vg0drg56 -wc-ao----   1.00g
  tmp             vg0drg56 Vwi-aotz--   4.00g tpool0        3.20
  tpool0          vg0drg56 twi-aotz--   9.50g               43.56  0.84
  [tpool0_tdata]  vg0drg56 Twi-ao----   9.50g
  [tpool0_tmeta]  vg0drg56 ewi-ao---- 256.00m
  var_cache       vg0drg56 Vwi-aotz--   4.00g tpool0        9.46
  var_log         vg0drg56 Vwi-aotz--   4.00g tpool0        3.59
  
And now for the really cool stuff!

Install snapper

apt-get install snapper

Create configs for / and /home.  By default it will begin taking snapshots every hour on the hour.  If you don't want this happening 

sudo snapper --config root create-config --fstype='lvm(ext4)' /
sudo snapper --config home create-config --fstype='lvm(ext4)' /home

I don't need hourly snapshots of / so I disable it.

sudo snapper --config root set-config TIMELINE_CREATE="no"

Then I create a permanent snapshot of the system when I first installed with minimal config.

sudo snapper --config root create --description minimal

apt also has a hook that takes a snapshot when you install software, so for fun lets install kde and then rollback to our minimal install.






  
  
