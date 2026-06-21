---
title: 'Air Gapped Kubernetes Install with Kubekey and Harbor'
date: 2025-01-21T16:16:29+03:00
tags: [ "kubekey", "harbor", "kubernetes" ]
---

We will be simulating an Kubernetes install with QEMU and Libvirt and use 
Harbor to cache container images.

We will have 4 VM's for this job

1. harbor: This machine will hold a [Harbor](https://goharbor.io/) instance to
proxy our kubernetes image pull requests.
2. control-panel: This machine will hold control panel for our kubernetes
instance.
3. worker-1 and worker-2: This machines will be our Kubernetes workers.

First install libvirt, virt-install and qemu. The exact command might differ
from system to system. I'm using Arch (btw.) and this command worked fine for
me:

```bash
sudo pacman -S libvirt virt-install qemu-full
```

In order to use `libvirtd`, start libvirtd:

```bash
sudo systemctl start libvirtd
```

and add yourself to libvirt group:

```bash
sudo usermod -a -G libvirt (your username)
```

Now we can configure our virtual machines. First download your favourite 
distributions cloud image. I will be using Ubuntu Noble, you can select a 
different one if you want.

Download the cloud image to `/var/lib/libvirt/images`

```bash
sudo curl -L -O --output-dir /var/lib/libvirt/images http://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
```

Create a new directory with following structure:

```
kubernetes
├── control-panel
├── harbor
├── worker-1
└── worker-2
```

This directory will hold all of our machines and their configuration inside 
kubernetes directory create a `kubernetes.xml` to hold our network configuration.
This xml file should look like:

```xml
<network>
  <name>kubernetes</name>
  <bridge name="virbr1"/>
  <ip address="192.168.2.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.2.2" end="192.168.2.254"/>
    </dhcp>
  </ip>
</network>
```

Now use following to define and start the kubernetes network:

```bash
sudo virsh net-define kubernetes.xml
sudo virsh net-start kubernetes
```

Also start the default network:

```bash
sudo virsh net-start default
```

Allow all users to access `virbr1`:

```
echo "allow virbr1" | sudo tee -a /etc/qemu/bridge.conf
```

Now create VM images:

```bash
qemu-img create -b /var/lib/libvirt/images/jammy-server-cloudimg-amd64.img -f qcow2 -F qcow2 harbor/harbor.img 20G
qemu-img create -b /var/lib/libvirt/images/jammy-server-cloudimg-amd64.img -f qcow2 -F qcow2 control-panel/control-panel.img 20G
qemu-img create -b /var/lib/libvirt/images/jammy-server-cloudimg-amd64.img -f qcow2 -F qcow2 worker-1/worker-1.img 20G
qemu-img create -b /var/lib/libvirt/images/jammy-server-cloudimg-amd64.img -f qcow2 -F qcow2 worker-2/worker-2.img 20G
```

We will use cloud-init to configure initial configuration. Create a file named 
`harbor/meta-data` with following contents:

```yaml
instance-id: harbor
local-hostname: harbor
```

Create a file named `harbor/user-data` with following contents:

```yaml
#cloud-config
users:
  - name: kubernetes-user
    ssh_authorized_keys:
      - (Your public ssh key)
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    groups: sudo
    shell: /bin/bash
```

For `control-panel`, `worker-1` and `worker-2` replicate the configuration in
their respective directories. In the end you should have the following 
structure:

```
kubernetes/
├── control-panel
│   ├── control-panel.img
│   ├── meta-data
│   └── user-data
├── harbor
│   ├── harbor.img
│   ├── meta-data
│   └── user-data
├── kubernetes.xml
├── worker-1
│   ├── meta-data
│   ├── user-data
│   └── worker-1.img
└── worker-2
    ├── meta-data
    ├── user-data
    └── worker-2.img
```

Now use `virt-install` to install the VM's. Since harbor will need internet 
access we will also add the default network interface that comes with libvirt:

```bash
virt-install --name=harbor \
             --ram=2048 \
             --vcpus=2 \
             --import --disk path=harbor/harbor.img,format=qcow2 \
             --network bridge=virbr0,model=virtio \
             --network bridge=virbr1,model=virtio \
             --cloud-init user-data=harbor/user-data,meta-data=harbor/meta-data \
             --os-variant=ubuntu22.04 \
             --noautoconsole
```

For other VM's we don't need the default bridge. For example can use following
for control-panel:

```bash
virt-install --name=control-panel \
             --ram=2048 \
             --vcpus=2 \
             --import --disk path=control-panel/control-panel.img,format=qcow2 \
             --network bridge=virbr1,model=virtio \
             --cloud-init user-data=control-panel/user-data,meta-data=control-panel/meta-data \
             --os-variant=ubuntu22.04 \
             --noautoconsole
```

```bash
sudo virsh net-dhcp-leases --network default
sudo virsh net-dhcp-leases --network kubernetes
```

to check your VM's ip's. You should be able to login to them with

```bash
ssh kubernetes-user@VMIP
```

First login harbor and execute

```
sudo ip a
```

Note the mac address of the interface wihch does not have an IP. Create a 
`/etc/netplan/10-private.yaml` with following content:

```
network:
  version: 2
  ethernets:
    enp2s0:
      match:
        macaddress: "NOTED MAC ADDRESS"
      dhcp4: true
      dhcp6: true
      set-name: "enp2s0"
```

Then use `sudo netplan apply`. Second interface should get an IP address. Now 
we can install Harbor to our VM. Use the [official instructions](https://docs.docker.com/engine/install/ubuntu/)
to install Docker and Docker-compose. In my case it is:

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Now go to the [Harbor Releases Page](https://github.com/goharbor/harbor/releases)
and get the latest version.

```bash
curl -L -O https://github.com/goharbor/harbor/releases/download/v2.12.2/harbor-online-installer-v2.12.2.tgz
```

Unarchive the installer

```bash
tar xvzf harbor-online-installer-v2.12.2.tgz
```

Copy and edit the harbor configuration:

```bash
cd harbor
cp harbor.yml.tmpl harbor.yml
vim harbor.yml
```

Now deploy harbor with:

```
sudo ./install.sh
```

Create a proxy cache as described in [Harbor Documentation](https://goharbor.io/docs/2.1.0/administration/configure-proxy-cache/).

Now we will use Kubekey to deploy our kubernetes cluster. If you are going to 
use harbor to use Kubekey, make sure that you can ssh into other VM's from harbor.

Use following to install Kubekey:

```bash
curl -sfL https://get-kk.kubesphere.io | sh -
```

Then copy example [manifest.yaml](https://github.com/kubesphere/kubekey/blob/master/docs/manifest-example.md)
file and edit it to fit your setup as described in the [official documentation](https://kubesphere.io/docs/v3.4/installing-on-linux/introduction/air-gapped-installation/).

Then use

```bash
./kk artifact export -m manifest.yaml -o kubesphere.tar.gz
```

to generate the artifacts.

Now generate Kubekey config with following command:

```
./kk create config -f config.yml
```

edit `kubekey.yml` to include your control-panel, worker-1 and worker-2 nodes.

Don't forget to include harbor's ip to insecureRegisteries.

You can deploy your cluster:

```bash
./kk create cluster -f config.yaml -a kubesphere.tar.gz --with-packages
```
