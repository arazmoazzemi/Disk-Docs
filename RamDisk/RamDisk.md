Note! use can use big swap file with seperated partition

***https://www.kernel.org/doc/html/v5.7/filesystems/tmpfs.html***

# Make TempFS Ramdisk :

```
$ mkdir -p /tmp/ramdisk

$ mount -t tmpfs -o size=65536M tmpfs /tmp/ramdisk

# 65536


# test wite speed
$ sudo dd if=/dev/zero of=/tmp/ramdisk/zero bs=4k count=100000

# test read speed
$ sudo dd if=/tmp/ramdisk/zero of=/dev/null bs=4k count=100000

```

Make TempFS Ramsk, With fstab File :

```

$ sudo nano /etc/fstab
$ myramdisk  /tmp/ramdisk  tmpfs  defaults,size=32G,x-gvfs-show  0  0
$ sudo mount -a

```
========================================================================================

# Example For Using RAM Disk to Reduce SSD Wear Out

```

# Packages
package_archive   /var/cache/apt/archives   tmpfs   defaults,size=6G   0   0

# Logs
cd /var/log/
sudo du -h
systemd_journal   /var/log/journal       tmpfs      defaults,size=6G    0    0

# Nginx
nginx_logs     /var/log/nginx/      tmpfs     defaults,size=6G    0    0


mount -a

```







sudo mkdir -p /media/ramdisk
sudo mount -t tmpfs -o size=2048M tmpfs /media/ramdisk

sudo chmod 1777 /media/ramdisk



grep /media/ramdisk /etc/mtab | sudo tee -a /etc/fstab
tmpfs /media/ramdisk tmpfs rw,size=32768 0 0
# 65536


# Check the existing ramdisk size.
df /media/ramdisk
# change size=512M for a 512 megabyte ram drive.
sudo vi /etc/fstab
# Remount the ramdisk, you will lose any existing content.
sudo mount -a /media/ramdisk
# Verify the new ramdisk size.
df /media/ramdisk


--------------brd------(block ram disk)---------------------------------------------------------


# he RAM Disk is created when the "brd" module is loaded (brd=block ram disk)

  modprobe brd

# This has three parameters.  If no parameters are used you get the defaults.

  rd_nr : Maximum number of brd devices
  rd_size : Size of each RAM disk in kbytes.
  max_part : Maximum number of partitions per RAM disk

# For example if you want one 1GB RAM Disk that can have two partitions you would use:

  modprobe brd rd_size=1024000 max_part=2 rd_nr=1

# The RAM Disk will then be in /dev/ram*
# You can then partition it as needed up to the max number of partitions.
# Then put a file system on it and mount it and your ready to use your RAM Disk.

----------------------------------------------------------------------------------
modprobe brd

modprobe brd rd_size=10240000 

sudo mkfs /dev/ram0
# sudo mkfs.xfs /dev/ram0

mkdir /tmp/ramdisk
sudo mount /dev/ram0 /tmp/ramdisk

df -h

# umount /tmp/ramdisk
# modprobe -r brd 

# test wite speed
sudo dd if=/dev/zero of=/tmp/ramdisk/zero bs=4k count=100000

# test read speed
sudo dd if=/tmp/ramdisk/zero of=/dev/null bs=4k count=100000

sudo chown -R yourUserName:yourGroupName /tmp/ramdisk

------------------------vga installation-----------------------------
virt-install \
  --name ramdisk-test \
  --connect=qemu:///system \
  --ram 8192 \
  --disk path=/tmp/ramdisk/ramdisk-test.qcow2,size=5 \
  --vcpu 4 \
  --graphics vnc \
  --cdrom /var/lib/libvirt/isos/ubuntu-22.04.2-live-server-amd64.iso 
  --network bridge=virbr0,model=virtio \
  --boot hd

---------------ZRAM-----compressed  and decompressed ---------------------------------


# Another option for a RAM Disks is zram. When you place a file onto a zram RAM disk, the file gets quickly compressed during the transfer and it is transparently decompress during the retrieval. This can be helpful in circumstances where your system doesn't quite have the amount of RAM you desire for your RAM disk.

# Here's how to create a zram RAM disk:

# Make a folder to which you will mount your RAM disk:
sudo mkdir /tmp/ramdisk 

# Change the ownership of that folder, so your user will have full access to the RAM disk when we later mount it:
sudo chown -R yourUserName:yourGroupName /tmp/ramdisk

# Make the folder immutable so you don't ever accidentally fill up your OS partition with data intended for the RAM disk:
sudo chattr +i /tmp/ramdisk

# Load the zram module:
sudo modprobe zram

# Create a 1GB ram disk:
sudo zramctl --find --size 1G

# The command above will output the device path of the RAM disk you've created. It will most likely be /dev/zram0, and that's what we'll assume going forward.
# Format the RAM disk to EXT4:
sudo mke2fs -t ext4 -O ^has_journal -L "zram device" /dev/zram0

# Mount the RAM disk to the immutable mount point folder we created:
sudo mount /dev/zram0 /tmp/ramdisk

# Now you should be able to move files to and from the RAM disk located at /tmp/ramdisk/.
#If you're done playing with it, unmount it:
sudo umount /tmp/ramdisk/

# Lastly, lets destroy the RAM disk and free up any memory it was using:
sudo zramctl --reset /dev/zram0

# If you also want to remove the the folder /tmp/ramdisk, first make it mutable:
sudo chattr -i /tmp/ramdisk

# Now you can delete the folder:
rm -rf /tmp/ramdisk




















