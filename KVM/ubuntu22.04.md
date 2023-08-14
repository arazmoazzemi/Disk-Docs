virt-install \
  --name ubuntu-desktop \
  --connect=qemu:///system \
  --ram 8192 \
  --disk path=/var/lib/libvirt/images/ubuntu-desktop.qcow2,size=100 \
  --vcpu 4 \
  --graphics vnc,listen=0.0.0.0 \
  --cdrom /var/lib/libvirt/isos/ubuntu-22.04.2-desktop-amd64.iso 
  --network bridge=virbr0,model=virtio \
  --boot hd
