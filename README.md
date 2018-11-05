# Installation of IBM Cloud Private - Community Edition

## Prepare your 3 VM environment

It is assumed that you have a 3 VM setup on your Windows laptop. You need minimum 32 GB of RAM to build a 3 node IBM Cloud Private - Community Edition.

However - it is possible to work with 16GB but it will be very slow.

Suggestions:

1. Use VMware Workstation - Even though it is a licensed software, it is worth buying it. I am not a fan of VirtualBox. The VMware is a rock solid software.

2. You can use `free` VMware player if you can get your hands on an already available VM that you can use. You need VMware Workstation should you need to build a VM from installation iso file.

3. My personal choice is to use `CentOS` but you can use any Linux distribution of your choice.

4. After you have 3 VMs up and running, you need to do the following:

  * Configure passwordless SSH access between all VMs. You can refer to this [link](https://github.com/vikramkhatri/sshsetup) for setting up SSH if you need automation.
  * Configure [direct LVM](Scripts/directLVM) access for a disk to be used by Docker
  * Configure `dnsmasq` in first VM for DNS routing if host names are not defined in the upstream DNS server. Optional - but useful
  * Configure `ntpd` on ist VM so that all VMs time is in sync. Since, we are going to use CockroachDB 3 replicas, the time must be within 500ms for it work properly.    

## Download ICP Installer

```
docker pull ibmcom/icp-inception:3.1.0
```

### Prepare Hosts file

### Prepare config.yaml file
