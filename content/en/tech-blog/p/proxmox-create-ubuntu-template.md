---
title: Creating a Ubuntu template on Proxmox
date: 2024-06-04T08:19:34-03:00
draft: false
language: en
featured_image: ../assets/images/featured/featured-proxmox-logo.jpg 
summary:
  This is a quick tutorial on creating a Ubuntu template on Proxmox, using the command line shell instead of the
  interface. The template will contain the proxmox agent and cloud-init, alongside a ssh public key to access
  the server after the template is provisioned.
categories: Tech-Blog
tags:
- proxmox
- virtualization
- template
- vm
type: tech-blog
---

This is a quick tutorial on creating a Ubuntu template on `Proxmox`, using the command line shell instead of the
interface. The template will contain the proxmox agent and `cloud-init`, alongside a ssh public key to access
the server after the template is provisioned.

After installing the [Proxmox Virtual Environment](https://www.proxmox.com/en/proxmox-virtual-environment/overview),
you can access the server's SSH as any other server. This tutorial will use this SSH shell as root to run all commands.

- This tutorial used Proxmox Virtual Environment 8.1.

## Downloading the Ubuntu distribution

In this Ubuntu Cloud Images [release page](https://cloud-images.ubuntu.com/releases/) you can download the
appropriate version that you want to create the template. In this example, I'll use the version
`Ubuntu 24.04 LTS (Noble Numbat)`.

Download the image:

```bash
wget -c https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img
```

- I'm using the `amd64` architecture, but you can use any other you want/need. It's the same.

- I'm trusting this image in this tutorial for the sake of simplicity. You should check the download with the
  `SHA256SUMS` checksum file and the authenticity with GPG and file `SHA256SUMS.gpg`.

## Customizing the image

Now it's time to begin customizing the image. You can do this by booting up a VM using this image, making
any changes and then using the resulting image, but that's too complicated.

Instead, let's use the `virt-customize` utility to install a package. Install it on the PVE server first:

```bash
apt-get -y install guestfs-tools
```

Then install the guest tools for proper integration between the VM and the ProxMox hypervisor:

```bash
virt-customize \
  -a ubuntu-24.04-server-cloudimg-amd64.img \
  --install qemu-guest-agent
```

You can do a lot of other things with `virt-custozmize`, like changing the root password, running scripts to modify
the filesystem and do some configuration, copy files, etc. We don't need any of that because the cloud-image is
simple and clean for our use as a template.

## Creating the template

The template ID will be 2404 and we'll import the previous customized image to a VM and then transform this VM into
a template.

First, create a basic VM using the `qm` command:

```bash
qm create 2404 \
  --name ubuntu-2404-cloud-init --numa 0 --ostype l26 \
  --cpu cputype=host --cores 2 --sockets 1 \
  --memory 1048  \
  --net0 virtio,bridge=vmbr0
```

Then import the disk image we customized earlier into the storage. In this example, PVE created a storage called
`local-lvm` and we'll import to that storage. You may need to change this storage name depending on your
PVE configuration.

```bash
# import the disk into PVE storage local-lvm
qm importdisk 2404 ubuntu-24.04-server-cloudimg-amd64.img local-lvm

# attach the imported disk as device scsi0
qm set 2404 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-2204-disk-0

# attach a virtual disk with cloudinit data as device ide2
qm set 2404 --ide2 local-lvm:cloudinit

# make the scsi0 as the bootdisk
qm set 2404 --boot c --bootdisk scsi0

# set serial connection to be able to see and control the server terminal on proxmox
qm set 2404 --serial0 socket --vga serial0

# enable the agent that we installed on image customization
qm set 2404 --agent enabled=1
```

For the SSH public key for the default user (`ubuntu`), if you don't already have the SSH public key, create one:

```bash
ssh-keygen -t ed25519 -f ubuntu -N ''
```

In this case, the public key file created is `ubuntu.pub`, so we include it in the VM too:

```bash
qm set 2404 --sshkey ubuntu.pub
```

With all configuration and customization done, the last step is to convert this VM into a template. Make sure
you didn't miss anything and run:

```bash
qm template 2404
```

The template is created and can be used!

## Using the template and acessing the server

You can create VMs using this template through the WebUI or command line. This is an example of creating a VM
using the command line:

```bash
# clone the template (ID 1000) into a new VM named my-new-server
# the VM's storage will be local-lvm as the earlier examples.
qm clone 2404 1000 --name my-new-server --full --storage local-lvm

# resize the disk to 20G
qm resize 1000 scsi0 20G

# if you don't have DHCP in your network, set a fixed IP address
qm set 1000 --ipconfig0 ip=192.168.124.101/24,gw=192.168.124.1

# start the VM
qm start 1000
```

When the system boots up, you can access the VM using the SSH key, like this:

```bash
ssh -i ubuntu ubuntu@192.168.124.101
```

This will use the SSH key `ubuntu` to login with user `ubuntu`. That's it!

## Conclusion and references

With these steps, you now have a proper cloud image template, working like all those public clouds.

References:

- `man 1 virt-customize`
- `man 1 qm`
- [Proxmox VE API](https://pve.proxmox.com/wiki/Proxmox_VE_API)
