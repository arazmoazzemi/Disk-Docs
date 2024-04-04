### Install and configure DRBD on twn nodes:

```
yum install nano -y
```

```
nano /etc/hosts


172.16.100.2 drbd01
172.16.100.3 drbd02
```

```
yum install ntp -y
timedatectl set-timezone Asia/Tehran
ntpdate pool.ntp.org
service ntpd start
systemctl enable ntpd.service
```

```
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
rpm -ivh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm

yum update -y
ll /etc/yum.repos.d/

yum info *drbd* | grep Name
#yum -y install drbd84-utils kmod-drbd84
yum -y install drbd90-utils kmod-drbd90
```

```
#Firewall_Configuration
#Disable_SELINUX

sestatus

nano /etc/selinux/config

#SELINUX=enforcing
SELINUX=disabled

sudo shutdown -r now

#systemctl disable firewalld

iptables -L

iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -F

yum install firewalld
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --add-port 7789/tcp --permanent
firewall-cmd --reload
```

```
modprobe drbd
lsmod | grep -i drbd
echo drbd > /etc/modules-load.d/drbd.conf
#lsmod | grep -i drbd
#find / -name drbd*
#insmod /usr/lib/modules/3.10.0-1127.el7.x86_64/extra/drbd84/drbd.ko
#insmod usr/lib/modules/3.10.0-1160.el7.x86_64/extra/drbd90/drbd.ko
#lsmod | grep -i drbd

systemctl start drbd
systemctl enable drbd

systemctl status drbd.service

```

```
cd /etc/drbd.d/

nano mydata.res

resource mydata {
 meta-disk internal;
 device /dev/drbd0;
 net {
  verify-alg sha256;
 }
 on drbd01 {   		
  node-id   0;				
  disk   /dev/sda;	
  address  172.16.100.2:7789;		
 }
 on drbd02 {
  node-id   1;
  disk   /dev/sda;
  address  172.16.100.3:7789;
 }
 
 connection-mesh {
  hosts drbd01 drbd02;
 }

```

```
mv /etc/drbd.d/global_common.conf /etc/drbd.d/global_common.conf.bak
```

```
nano /etc/drbd.d/global_common.conf

global {
     usage-count no;
}
common {
     handlers {
          fence-peer "/usr/lib/drbd/crm-fence-peer.sh";
          after-resync-target "/usr/lib/drbd/crm-unfence-peer.sh";
          split-brain "/usr/lib/drbd/notify-split-brain.sh root";
          pri-lost-after-sb "/usr/lib/drbd/notify-pri-lost-after-sb.sh; /usr/lib/drbd/notify-emergency-reboot.sh; echo b > /proc/sysrq-trigger ; reboot -f";
     }
     startup {
          wfc-timeout 0;
     }
     options {
     }
     disk {
          md-flushes yes;
          disk-flushes yes;
          c-plan-ahead 1;
          c-min-rate 100M;
          c-fill-target 20M;
          c-max-rate 4G;
     }
     net {
          after-sb-0pri discard-younger-primary;
          after-sb-1pri discard-secondary;
          after-sb-2pri call-pri-lost-after-sb;
          protocol     C;
          tcp-cork yes;
          max-buffers 20000;
          max-epoch-size 20000;
          sndbuf-size 0;
          rcvbuf-size 0;
     }
}

```

---
# Boath nodes:

```
drbdadm create-md mydata
drbdadm up mydata
```

#Primary_node
```
sudo drbdadm primary mydata --force 

sudo drbdadm status mydata
drbdadm -- --overwrite-data-of-peer primary all

cat /proc/drbd


systemctl status drbd.service
```


### Troubleshooting
```
#on_both_node
#sudo drbdadm down mydata
#on_master_node

sudo drbdadm up mydata
```
```
#host2
drbdadm create-md mydata
drbdadm up mydata
```

### Re-sync:
```
drbdadm secondary all
drbdadm disconnect all

drbdadm status


drbdadm invalidate all

drbdadm status

drbdadm connect all

drbdadm status 
```


### troubleshoot Bandwith
```
drbdadm -V

drbdadm disconnect all

# ON BOTH NODES
nano /var/lib/drbd.d/drbdmanage_global_common.conf 

# it must be content of /var/lib/drbd.d/drbdmanage_global_common.conf !!!!
common {
disk {
        on-io-error             detach;
        no-disk-flushes ;
        no-disk-barrier;
        c-plan-ahead 10;
        c-fill-target 24M;
        c-min-rate 10M;
        c-max-rate 100M;
}
net {
        # max-epoch-size          20000;
        max-buffers             36k;
        sndbuf-size            1024k ;
        rcvbuf-size            2048k;
}
}


/etc/init.d/drbd restart

```

```
#all_nodes
drbdadm adjust mydata
```

### Make a file system

```
#Host1
sudo mkfs.xfs /dev/drbd0
#host1&host2

mkdir -p /root/replicated

#mount /dev/drbd0 /root/replicated/
#umount /dev/drbd0 /root/replicated/

#fstab
#/dev/drbd0 /root/replicated xfs defaults 0 0

Again_NOTE!_at_host_1==========>before_cluster_configuratiom_two_host=>secondaye

drbdadm secondary mydata

drbdadm status mydata

```
---
---

### Setup Pacemaker HA cluster-

---
---
### [Help lnk](https://www.canarytek.com/2017/09/06/DRBD_NFS_Cluster.html)

```bash
yum install nfs-utils lvm2 mc corosync pcs pacemaker fence-agents-all  resource-agents psmisc pwgen policycoreutils-python -y

firewall-cmd --permanent --add-service=high-availability
firewall-cmd --add-service=high-availability
firewall-cmd --reload


systemctl stop nfs-lock && systemctl disable nfs-lock
systemctl enable rpcbind --now

firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=mountd
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --reload

systemctl enable corosync
systemctl enable pacemaker

passwd hacluster
systemctl start pcsd
systemctl enable pcsd
pcs cluster auth drbd01 drbd02 -u hacluster -p mo@123i
pcs cluster setup --start --name mycluster drbd01 drbd02


pcs cluster start --all

pcs status
```

### Cluster_service_configuration:

```
pcs cluster cib cluster_config
pcs -f cluster_config property set stonith-enabled=false
pcs -f cluster_config property set no-quorum-policy=ignore
pcs -f cluster_config resource defaults resource-stickiness=200
```

### Resource&clone_drbd_volume:

```
pcs -f cluster_config resource create nfs01-vol ocf:linbit:drbd \
  drbd_resource=mydata \
  op monitor interval=30s

pcs -f cluster_config resource master nfs01-clone nfs01-vol \
  master-max=1 master-node-max=1 \
  clone-max=2 clone-node-max=1 \
  notify=true
```

### Cluster_Resource_for_filesystem:

```
pcs -f cluster_config resource create nfs01_fs Filesystem \
  device="/dev/drbd0" \
  directory="/root/replicated" \
  fstype="xfs" \
  options=uquota,gquota

pcs -f cluster_config constraint colocation add nfs01_fs with nfs01-clone \
  INFINITY with-rsc-role=Master

pcs -f cluster_config constraint order promote nfs01-clone then start nfs01_fs
```

### Floating_service_ip_used_for_NFS

```
pcs -f cluster_config resource create nfs_vip01 ocf:heartbeat:IPaddr2 \
 ip=172.16.100.100 cidr_netmask=24 \
 op monitor interval=30s

pcs -f cluster_config constraint colocation add nfs_vip01 with nfs01_fs INFINITY
pcs -f cluster_config constraint order nfs01_fs then nfs_vip01
```

### Resource_for_nfs_service:

```
pcs -f cluster_config resource create nfs-service nfsserver nfs_shared_infodir=/root/replicated nfs_ip=172.16.100.100
pcs -f cluster_config constraint colocation add nfs-service with nfs_vip01 INFINITY
pcs -f cluster_config constraint order nfs_vip01 then nfs-service
```

### NFS_export_Resources:

```
pcs -f cluster_config resource create nfs-export01 exportfs clientspec=172.16.100.0/24 options=rw,sync,no_root_squash,no_subtree_check directory=/root/replicated fsid=0
pcs -f cluster_config constraint colocation add nfs-export01 with nfs-service INFINITY
pcs -f cluster_config constraint order nfs-service then nfs-export01
```

### Verify that defined resources and constraints are correct:

```bash
pcs -f cluster_config resource show
pcs -f cluster_config constraint
pcs cluster cib-push cluster_config
pcs cluster start --all
pcs cluster enable --all

#pcs resource cleanup

mount 172.16.100.100:/root/replicated/105

#client
rsize=8192,wsize=8192,acl,udp,nfsvers=3,rw

mount 192.168.31.60:/root/replicated/ /mnt/

nano /etc/fstab
172.16.100.100:/root/replicated /mnt/ nfs rw,hard,intr 0 0
```

### split_brain:

```
sudo journalctl | grep Split-Brain

drbdadm disconnect mydata
drbdadm secondary mydata
drbdadm -- --discard-my-data mydata connect mydata

drbdadm primary mydata
drbdadm connect mydata

systemctl restart drbd.service

drbdadm invalidate mydata
drbdadm down mydata
drbdadm create-md mydata
drbdadm adjust mydata

drbdadm adjust mydata

```













