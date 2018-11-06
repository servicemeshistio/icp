# Installation of IBM Cloud Private

The Community Edition of IBM Cloud Private is free and is a great resource to learn.

Let's walk through the steps to install a 3 node IBM Cloud Private Cluster

### Assumptions

* You know how to install a Linux distribution in your VM environment
* You know how to install `docker`
* You have a docker hub account to download IBM Cloud Private images

### What you need?

* A decent Windows 7 / 10 laptop with minimum 32 GB RAM and Intel Core i7 processor and a preferable minimum 512 GB SSD
* VMWare Workstation 14.1.2 - Even though it is a licensed software, it is worth buying it. I am not a fan of VirtualBox. The VMware is a rock solid software.
* You can use `free` VMware player if you can get your hands on an already available VM that you can use. You need VMware Workstation should you need to build a VM from installation iso file.
* Build your 3 VMs. My personal choice is to use `CentOS` but you can use any Linux distribution of your choice.
* Use `vmnet8` subnet address `192.168.142.0` with NAT.
* Attach 100 GB thin provisioned disk (vmdk) on each node for holding Docker images and containers
* Attach 20 GB thin provisioned disk (vmdk) on each node for persistent storage used by the IBM Cloud Private

Build VMs and when you have all 3 VMs up and running, prepare your environment.

## Prepare your VMs

* Make sure that you can run `ping -c4 google.com` from each VM to make sure that Internet access is available.  
* Configure `passwordless` SSH access between all VMs. You can refer to this [link](https://github.com/vikramkhatri/sshsetup) for setting up SSH if you need automation.
* Configure [direct LVM](Scripts/directLVM) access for a disk to be used by Docker backend. Use 100 GB disk (vmdk) that you attached to the VM as an argument to the script. Run this on all nodes
* Install docker engine on all nodes
* Configure [dnsmasq](/Scripts/dnsmasq.md) in first VM for DNS routing if host names are not defined in the upstream DNS server. Optional - but useful
* Configure [ntpd](Scripts/ntpd.md) on 1st VM so that all VMs time is in sync. Since, we are going to use 3 replicas for CockroachDB, the time must be within 500ms for it work properly.   

## Download ICP Installer

On your 1st VM, login as root and login to your docker account to download the installation docker image.

```
docker pull ibmcom/icp-inception:3.1.0
```

### Prepare Hosts file

### Prepare config.yaml file
