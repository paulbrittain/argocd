# How to create a ubuntu server template

## Create the VM
name: ubuntu-2204-template
vm id: high, such as 900. This way the templates are alphabetically sorted to the bottom of the list.
next
Do not use any media
next
Enable Qemu agent
next
delete the existing disk
next
1 core
next
1024 memory
create

```shell
$ wget https://cloud-images.ubuntu.com/minimal/releases/jammy/release/ubuntu-22.04-minimal-cloudimg-amd64.img
$ ls

$ qm set <vmId> --serial0 socket --vga serial0
$ mv ubuntu-22.04-minimal-cloudimg-amd64.img ubuntu-22.04.qcow2
$ qemu-img resize ubuntu-22.04.qcow2 32G
$ qm importdisk <vmId> ubuntu-22.04.qcow2

$ qm importdisk 900 ubuntu-22.04.qcow2 local-lvm
```

go back to proxmox ui
add the disk to the vm via Hardware, bottom screen, unused disk. If you have an ssd, then tick discard and ssd emulation.
go to options
edit Boot order, tick the unticked disk and place it in 2nd place after cdrom (which we keep first so that we can boot from disk/usb if needed)
edit Start at boot, turn it on, hit OK

right click the vm and choose Convert to Template


go to Hardware and Add the cloud init storage
Go to Cloud Init
set the ssh public key
set user and password
set ipv4 to DHCP
click regenerate image

## install and start qemu-guest-agent.
```shell
$ sudo apt install qemu-guest-agent
$ sudo systemctl enable qemu-guest-agent
$ sudo systemctl start qemu-guest-agent
```

## note the VM IPs
control ip: 192.168.178.89
node1 ip: 192.168.178.90

## ssh into the vms
```shell
$ ssh <username>@{IP}
```

## update os
```shell
$ sudo apt update && sudo apt dist-upgrade
```

## install nano, if needed
```shell
$ sudo apt install nano
```

## create a static ip for each node, be sure to have the right ethernet subnet thingy, eth0 in this case. And use the IP for the node.
```shell
$ sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
    version: 2
    ethernets:
        eth0:
            addresses: [192.168.178.89/24]
            nameservers:
                addresses: [192.168.178.1]
            routes:
                - to: default
                  via: 192.168.178.1
```

## test the new config out, hit enter to apply
```shell
$ sudo netplan try
```

## now all the tweaks
```shell
$ sudo apt install containerd
$ systemctl status containerd
```

## generate the default containerd config and store it in /etc/
```shell
$ sudo mkdir /etc/containerd
$ containerd config default | sudo tee /etc/containerd/config.toml
```

## make our change
```shell
$ sudo nano /etc/containerd/config.toml
```

CTRL+w for search...
"runc.options" + enter
Set SystemdCgroup to true, save file.

## disable swap, if its not in the fstab then do nothing
```shell
$ free -m
```

if you have swap values >0 then edit fstab

```shell
$ sudo nano /etc/fstab
```

## enable ip forwarding
```shell
$ sudo nano /etc/sysctl.conf
```

uncomment "net.ipv4.ip_forward=1"

```shell
$ sudo nano /etc/modules-load.d/k8s.conf
```

add: br_netfilter

# Now all the k8s-specific setup

## install gpg key and the repository
```shell
$ curl -fsSL "https://packages.cloud.google.com/apt/doc/apt-key.gpg" | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/kubernetes-archive-keyring.gpg
$ echo 'deb https://packages.cloud.google.com/apt kubernetes-xenial main' | sudo tee /etc/apt/sources.list.d/kubernetes.list
$ sudo apt update -y
```

## install packages for k8s
```shell
$ sudo apt install kubeadm kubectl kubelet
```

## create a node template
```shell
$ sudo cloud-init clean
$ sudo rm -rf /var/lib/cloud/instances
$ sudo truncate -s 0 /etc/machine-id
$ sudo rm /var/lib/dbus/machine-id
$ sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
$ ls -l /var/lib/dbus/machine-id
$ cat /etc/machine-id
$ sudo poweroff
```

# Using the template to create k8s nodes

## remember to adjust the CPU cores and ram for the VMs when in powered off state
4 cpu, 4gb for controller
2 cpu, 2gb for workers

# Connecting the nodes together to form the cluster

## reconnect to the nodes, then in the control initialize the cluster

```shell
$ sudo kubeadm init --control-plane-endpoint=192.168.178.89 --node-name control --pod-network-cidr=10.244.0.0/16

$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## check what pods are running, note they should mostly be Running, but coredns pods are Pending
```shell
$ kubectl get pods --all-namespaces
```

## this is because we need to install a networking overlay
```shell
$ kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
$ kubectl get pods --all-namespaces
```

## observe that all pods are Running now

## joining worker nodes
```shell
$ sudo... paste the join command
```

## note the error, too much time since cluster creation. So on the controller node, execute this command and use the response command to run on the worker nodes to join the cluster
```shell
$ kubeadm token create --print-join-command
```
