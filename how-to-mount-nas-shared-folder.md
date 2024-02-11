# How to mount a NAS shared folder

```shell
# If you're doing this on the Promox host itself, do these first. If not, just ignore this block of commands.
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

# To revert the mounting changes

```shell
rmdir /mnt/nas
umount /mnt/nas
```

# A script for automating the above setup

```shell
#!/bin/bash

# Variables
NFS_SERVER="<volume ip>"
NFS_SHARE="/<volume path> "
MOUNT_POINT="/mnt/nas"

# Install nfs-common if not already installed
which nfs-common &>/dev/null || {
  sudo apt-get update
  sudo apt-get install -y nfs-common
}

# Create mount point
sudo mkdir -p "${MOUNT_POINT}"

# Mount NFS share
sudo mount -t nfs "${NFS_SERVER}:${NFS_SHARE}" "${MOUNT_POINT}"

# Add to /etc/fstab for persistence
grep -q "${NFS_SHARE}" /etc/fstab || {
  echo "${NFS_SERVER}:${NFS_SHARE} ${MOUNT_POINT} nfs defaults 0 0" | sudo tee -a /etc/fstab
}
```