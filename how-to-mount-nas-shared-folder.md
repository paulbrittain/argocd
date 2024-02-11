# How to mount a NAS shared folder

```shell
nano /etc/apt/sources.list.d/pve-enterprise.list
    # Comment out the pve-enterprise repository
nano /etc/apt/sources.list.d/ceph.list
    # Comment out the pve-enterprise repository
    # add: deb http://download.proxmox.com/debian/ceph-quincy bookworm no-subscription
nano /etc/apt/sources.list
    # add:deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription


apt-get update && apt-get install nfs-common

mkdir -p /mnt/nas

mount -t nfs <volume ip>:/<volume path> /mnt/nas

nano /etc/fstab

<volume ip>:/<volume path> /mnt/nas nfs defaults 0 0

mount -a
```