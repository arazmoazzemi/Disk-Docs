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




