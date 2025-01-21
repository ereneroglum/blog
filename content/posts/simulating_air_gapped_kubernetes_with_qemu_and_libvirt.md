---
title: 'Simulating_air_gapped_kubernetes_with_qemu_and_libvirt'
date: 2025-01-21T10:55:36+03:00
tags: [ "kubernetes", "k3s", "libvirt", "qemu" ]
---

We will be simulating an air gapped Kubernetes install with QEMU and Libvirt.

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
sudo curl -L -O --output-dir /var/lib/libvirt/images http://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
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
  <forward mode="nat"/>
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

Allow all users to access `virbr1`:

```
echo "allow virbr1" | sudo tee /etc/qemu/bridge.conf
```

Now create VM images:

```bash
qemu-img create -b /var/lib/libvirt/images/noble-server-cloudimg-amd64.img -f qcow2 -F qcow2 harbor/harbor.img 20G
qemu-img create -b /var/lib/libvirt/images/noble-server-cloudimg-amd64.img -f qcow2 -F qcow2 control-panel/control-panel.img 20G
qemu-img create -b /var/lib/libvirt/images/noble-server-cloudimg-amd64.img -f qcow2 -F qcow2 worker-1/worker-1.img 20G
qemu-img create -b /var/lib/libvirt/images/noble-server-cloudimg-amd64.img -f qcow2 -F qcow2 worker-2/worker-2.img 20G
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
their respective directories.

Now use `virt-install` to install the VM's:

```bash
virt-install --name=harbor \
             --ram=4096 \
             --vcpus=2 \
             --import --disk path=harbor/harbor.img,format=qcow2 \
             --network bridge=virbr1,model=virtio \
             --cloud-init user-data=harbor/user-data,meta-data=harbor/meta-data \
             --os-variant=ubuntu24.04 \
             --graphics vnc,listen=0.0.0.0 --noautoconsole
```

Other VM's can be started similarly. Use

```bash
sudo virsh net-dhcp-leases --network kubernetes
```

to check your VM's ip's. You should be able to login to them with

```bash
ssh kubernetes-user@VMIP
```

Now we can proceed to setup kubernetes. I will use [K3S](https://docs.k3s.io)
for simplicity. First, ssh into control-panel and install k3s to control with:

```bash
curl -sfL https://get.k3s.io | sh -
```

Save the node token from `sudo cat /var/lib/rancher/k3s/server/node-token`.

Then ssh into worker-1 and worker-2 and use 

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://CONTROL-PANEL-IP:6443 K3S_TOKEN=NODE-TOKEN sh -
```

to join them into the kubernetes cluster. Now we will install Harbor to our
harbor VM. SSH into harbor and use the [official instructions](https://docs.docker.com/engine/install/ubuntu/)
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

In every node of your k3s installation, create a `/etc/rancher/k3s/registries.yaml`
file as described in [K3S Documentation](https://docs.k3s.io/installation/private-registry)
with following contents:

```bash
mirrors:
  docker.io:
    endpoint:
      - http://HARBOR-IP
```

Restart k3s in your nodes. Now we can use ```ip route del default``` to simulate
an air gapped setup. You should be able to use kubernetes with Harbor
container image proxy.
