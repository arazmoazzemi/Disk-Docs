Install DRBD between two nodes ubuntu 22.04

```
nano /etc/hosts

172.16.100.2 drbd01
172.16.100.3 drbd02
```

```
apt install software-properties-common apt-transport-https ca-certificates -y
add-apt-repository ppa:linbit/linbit-drbd9-stack
ls -alh /etc/apt/sources.list.d/
apt update
apt install linux-headers-`uname -r`
apt install drbd-dkms drbd-utils -y
```

```
modprobe drbd
```


```
apt install ntp -y
timedatectl set-timezone Asia/Tehran
apt install ntpdate 
ntpdate pool.ntp.org
service ntp start
systemctl enable ntp
```

```
iptables -L

iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -F
```

```
firewall-cmd --add-port 7789/tcp --permanent
firewall-cmd --reload
```

```
cp /etc/drbd.d/global_common.conf /etc/drbd.d/global_common.conf.bak
```

```
nano /etc/drbd.d/global_common.conf 

global {
    usage-count no;
}

common {
    net {
        protocol C;
    }
}

```

### Or use below config:

```
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


### 2 nodes:
```
drbdadm create-md mydata
drbdadm up mydata
```

### Run at primary_node
```
sudo drbdadm primary mydata --force 

sudo drbdadm status mydata
drbdadm -- --overwrite-data-of-peer primary all

cat /proc/drbd


systemctl status drbd.service
```

### If faced with some errors:
```
# on_both_node

sudo drbdadm down mydata

# on_master_node

sudo drbdadm up mydata

```
```
# host2

drbdadm create-md mydata
drbdadm up mydata
```


### Re-sync commands:
```
drbdadm secondary all
drbdadm disconnect all

drbdadm status

drbdadm invalidate all

drbdadm status

drbdadm connect all

drbdadm status 
```


### troubleshot_Bandwith

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

```

```
/etc/init.d/drbd restart
```

### all_nodes

```
drbdadm adjust mydata
```

### Create file system

```
# Host1

sudo mkfs.xfs /dev/drbd0

# host1&host2

mkdir -p /root/replicated


# mount /dev/drbd0 /root/replicated/
# umount /dev/drbd0 /root/replicated/

# fstab
#/dev/drbd0 /root/replicated xfs defaults 0 0

Again_NOTE!_at_host_1==========>before_cluster_configuratiom_two_host=>secondaye

drbdadm secondary mydata

drbdadm status mydata

```





-------Setup Pacemaker HA cluster----------------------------------------------------------------------

https://www.canarytek.com/2017/09/06/DRBD_NFS_Cluster.html


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

-----------------Cluster_service_configuration-------------------------------------------------------------------------------


pcs cluster cib cluster_config

pcs -f cluster_config property set stonith-enabled=false
pcs -f cluster_config property set no-quorum-policy=ignore


pcs -f cluster_config resource defaults resource-stickiness=200


------------Resource&clone_drbd_volume----------------------------------------------------------------------------------------

pcs -f cluster_config resource create nfs01-vol ocf:linbit:drbd \
  drbd_resource=mydata \
  op monitor interval=30s

pcs -f cluster_config resource master nfs01-clone nfs01-vol \
  master-max=1 master-node-max=1 \
  clone-max=2 clone-node-max=1 \
  notify=true

---------------Cluster_Resource_for_filesystem------------------------------------------------------------------------------------

pcs -f cluster_config resource create nfs01_fs Filesystem \
  device="/dev/drbd0" \
  directory="/root/replicated" \
  fstype="xfs" \
  options=uquota,gquota

pcs -f cluster_config constraint colocation add nfs01_fs with nfs01-clone \
  INFINITY with-rsc-role=Master

pcs -f cluster_config constraint order promote nfs01-clone then start nfs01_fs


-----------------Floating_service_ip_used_for_NFS--------------------------------------------------------------------------------------------------------------------


pcs -f cluster_config resource create nfs_vip01 ocf:heartbeat:IPaddr2 \
 ip=192.168.31.60 cidr_netmask=24 \
 op monitor interval=30s

pcs -f cluster_config constraint colocation add nfs_vip01 with nfs01_fs INFINITY

pcs -f cluster_config constraint order nfs01_fs then nfs_vip01


-----------------Resource_for_nfs_service---------------------------------------------------------------------------------------------------------------

pcs -f cluster_config resource create nfs-service nfsserver nfs_shared_infodir=/root/replicated nfs_ip=192.168.31.60
pcs -f cluster_config constraint colocation add nfs-service with nfs_vip01 INFINITY
pcs -f cluster_config constraint order nfs_vip01 then nfs-service


---------------NFS_export_Resources----------------------------------------------------------------------------------------------

pcs -f cluster_config resource create nfs-export01 exportfs clientspec=192.168.31.0/24 options=rw,sync,no_root_squash,no_subtree_check directory=/root/replicated fsid=0
pcs -f cluster_config constraint colocation add nfs-export01 with nfs-service INFINITY
pcs -f cluster_config constraint order nfs-service then nfs-export01


---------Verify that defined resources and constraints are correct-----------------------------------------------------------

pcs -f cluster_config resource show

pcs -f cluster_config constraint


pcs cluster cib-push cluster_config


pcs cluster start --all
pcs cluster enable --all


#pcs resource cleanup



mount 192.168.31.60:/root/replicated/105




#client
rsize=8192,wsize=8192,acl,udp,nfsvers=3,rw

mount 192.168.31.60:/root/replicated/ /mnt/

nano /etc/fstab
192.168.31.60:/root/replicated /mnt/ nfs rw,hard,intr 0 0



-------------------------split_brain---------------------------------------------------------------

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















