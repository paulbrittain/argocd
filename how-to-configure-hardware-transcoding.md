# This is an admittedly quite poor and vague collection of pointers for configuring hardware transcoding on an LXC running in Proxmox.

https://forum.proxmox.com/threads/plex-hw-transcoding-lxc-and-jasper-lake-igpu-passthru.116163/

https://tteck.github.io/Proxmox/

## On the host

echo 'options i915 enable_guc=3' >> /etc/modprobe.d/i915.conf
echo 'options i915 enable_fbc=1' >> /etc/modprobe.d/i915.conf
update-initramfs -u

Add to the `/etc/pve/lxc/<num>.conf`

```
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.cgroup2.devices.allow: c 29:0 rwm
lxc.mount.entry: /dev/fb0 dev/fb0 none bind,optional,create=file
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/dri/renderD128 dev/renderD128 none bind,optional,create=file
```

## On the LXC

I added these to the /etc/apt/sources.list, not 100% sure if you need them (I'm writing this document backwards after hw passthrough has been achieved)

```
deb http://us.archive.ubuntu.com/ubuntu/ jammy main restricted multiverse
deb-src http://us.archive.ubuntu.com/ubuntu/ jammy main restricted multiverse
```

apt update
apt install pciutils -y
lspci -v

apt install i965-va-driver va-driver-all
apt install mesa-utils -y
apt install intel-media-va-driver-non-free -y

ls -l /dev/dri
chown plex:plex /dev/dri/renderD128