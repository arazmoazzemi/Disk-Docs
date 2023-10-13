# Convert OVA disk images to qcow2:

Copy source images from windows host to linux host:

```cmd
scp tar -xfv PNET_4.2.10.ova <user@password:/home/it>
```

Convert disk images on linux host:

```bash
tar -xfv PNET_4.2.10.ova
qemu-img convert -O qcow2 PNET_4.2.10-disk1.vmdk PNET_4.2.10-disk1.qcow2
```
