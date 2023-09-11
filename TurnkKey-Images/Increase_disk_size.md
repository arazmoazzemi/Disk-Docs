# Increase Turnkey Disk Size images:

[Help](https://www.turnkeylinux.org/forum/support/thu-20200903-1526/screwed-disk-resizing-lv)

```bash
cfdisk -L /dev/sda

pvresize /dev/sda1

lvextend -l +100%FREE /dev/turnkey/root

resize2fs /dev/turnkey/root

df -h /


-----------ubuntu------------

lvm
lvm> lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
lvm> exit


resize2fs /dev/ubuntu-vg/ubuntu-lv

```
