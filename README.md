# Installation of IBM Cloud Private

The Community Edition of IBM Cloud Private is free and is a great resource to learn.

Let's walk through the steps to install a 3 node IBM Cloud Private Cluster

## Assumptions

* You know how to install a Linux distribution in your VM environment
* You know how to install `docker`
* You have a docker hub account to download IBM Cloud Private images

## Prerequisites

* A decent Windows 7 / 10 laptop with minimum 32 GB RAM and Intel Core i7 processor and a preferable minimum 512 GB SSD
* VMWare Workstation 14.1.2/15 - Even though it is a licensed software, it is worth buying it. I am not a fan of VirtualBox. The VMware is a rock solid software. I have run 8 VMs simultaneously on my Windows 7 laptop with double CPU provisioning. I rarely see crash of VMs now.
* You can use `free` VMware player but unfortunately you can run only one VM per host.
* You could use `virtualbox` but I do not have good feelings about it after my use of it.
* Build your 3 VMs. My personal choice is to use `CentOS` but you can use any Linux distribution of your choice.
* In VMware Workstation, `Edit` ⇨ `Virtual Network Editor` to set the `vmnet8` to subnet address `192.168.142.0` in NAT mode.
* Attach 100 GB thin provisioned disk (vmdk) on each node for storing Docker images and containers
* Attach 20 GB thin provisioned disk (vmdk) on each node for persistent storages configuration.

Build VMs and when you have all 3 VMs up and running, prepare your environment.

## Resources for VMs and Types

The CPU and memory recommendations for VMs to build a 3 node cluster is:

|VM |Type|IP Address|CPU|Memory|
|---------------------|-----------|-----------|-----------|----------|
|node01|Master,Proxy,Worker,Management|192.168.142.101|8|12 GB|
|node02|Worker|192.168.142.102|4|8 GB|
|node03|Worker|192.168.142.103|4|8 GB|

The installation requires minimum 8 cores for the Master VM. So, with a Intel Core i7 having only 4 cores, how do we get the extra cores.

It is OK to provision more cores than the machine has. The VMware is pretty good in dynamic CPU scheduling and this is one of the reason that I like VMware more than the VirtualBox.

If using Intel Core i7 processor in a laptop, it gives 4 cores and 2 threads per core. It is OK to allocate 8 CPU to first VM and 4 each to second and third VM. This is just the double booking of the CPU cores.

## Prepare your VMs

You can either build your own VMs using your choice of Linux distribution and your choice of virtualization platform.

If you are learning and have no prior experience, you can then download the base VMs that we built for this purpose.

### Download VMs

If you like to download three base VM images to install IBM Cloud Private community edition cluster, you can download 7z of images from [here](#). If you download VMs, you can skip this `Prepare your VMs` section as theese following steps are already taken care in the VMs.

Continue with these steps, should you choose to build your own VMs.   

#### Set up passwordless access

Configure `passwordless` SSH access between all VMs. You can refer to this [link](https://github.com/vikramkhatri/sshsetup) for setting up SSH if you need automation.

#### Disable Local firewall on each VM
```
systemctl disable firewalld
systemctl stop firewalld
```
#### Disable `selinux` on each VM

```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

Reboot is required after changing the above parameter.

#### Add the following parameters to `/etc/sysctl.conf`

```
net.ipv4.tcp_keepalive_time = 10
net.ipv4.tcp_keepalive_probes = 10
net.ipv4.tcp_keepalive_intvl = 10

net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv4.tcp_mem = 182757 243679 365514
net.core.netdev_max_backlog = 182757
net.ipv4.conf.eth1.proxy_arp = 1
fs.inotify.max_queued_events = 1048576
fs.inotify.max_user_instances = 1048576
fs.inotify.max_user_watches = 1048576
vm.max_map_count = 262144
kernel.dmesg_restrict = 0
net.bridge.bridge-nf-call-iptables = 1
```

### Configure dnsmasq on First VM

If you are building your own VMs, [dnsmasq](/docs/dnsmasq.md) in first VM for DNS routing if host names are not defined in the upstream DNS server. Optional - but useful.

If you are using downloaded VMs, the `dnsmasq` is already configured.

### Configure ntp

If you are building your own VMs, configure [ntpd](docs/ntpd.md) on 1st VM so that all VMs time is in sync. Since, we are going to use 3 replicas for CockroachDB, the time must be within 500ms for it work properly.

If you are using downloaded VMs, it is already configured.

### Install docker engine on all nodes

If you are using your own VM, install Docker engine in each VM. Consult docker documentation. If you are using downloaded VMs, docker version `18.03.1-ce` is already installed in the VMs.

### Internet Access

Make sure that you can run `ping -c4 google.com` from each VM to make sure that Internet access is available.  

If ping does not succeed, it is quite possible that the network address of VMnet8 adapter needs to be fixed. Refer to [this](docs/vmnet.md) link for instructions to fix vmnet8 subnet address in VMware.

### Configure docker backend

Configure [direct LVM](Scripts/directLVM) access for a disk to be used by Docker backend.

If you are building your own VM, attach a 100 GB thin provisioned disk to all VMs and run above script on all VMs. Use `lsblk` to find out name of the disk that you will use as a Docker backend.

The script requires input parameter as the name of the disk to be used as a Docker backend.

If you downloaded the VMs, the disk to be used for Docker backend is of size 100 GB and the script `directLVM` is in `/root/bin/scripts` folder.

```
lsblk
```

For example: if the disk is `/dev/sdc`, run the command:

```
cd /root/bin/scripts
./directLVM /dev/sdc
```

SSH to `node02` and `node03` and repeat above commands to configure Docker backend in remaining two VMs.

After configuring it, run the following command to make sure that it is created successfully.

```
ssh node01 docker info
ssh node02 docker info
ssh node03 docker info
```

## Install Three Node IBM Cloud Private Cluster

On your 1st VM, login as root and login to your docker account to download the installation docker image.

Login to Dockerhub
```
docker login -u <username> -p <password>
```

Download ICP 3.1 installer

```
docker pull ibmcom/icp-inception:3.1.0
```

The docker pull will begin to download the ICP installer.

```
3.1.0: Pulling from ibmcom/icp-inception
a073c86ecf9e: Pull complete
cbc604ae1944: Downloading [============>                                      ]  18.85MB/74.78MB
650be687acc7: Download complete
a687004334e9: Downloading [=>                                                 ]  9.157MB/380.7MB
7664ea9d19f3: Downloading [========>                                          ]  7.851MB/45.49MB
97c4037e775d: Pulling fs layer
5fb11bea9f19: Waiting
19aea527057b: Waiting
060ae3284742: Waiting
8db2b6e28cb8: Waiting
92a37b9fe585: Waiting
dfb007dce8a8: Waiting
0573abd4b204: Waiting
875ebddb61da: Waiting
a1a7fd214344: Waiting
391862def9ea: Waiting
66ae4853d35e: Waiting
bf0833453156: Waiting
6d3268394639: Waiting
```

### Prepare Hosts file

Extract sample hosts and config.yaml file from the installer and copy private ssh key to the ICP home directory

```
mkdir -p /opt/ibm/icp3.1.0
cd /opt/ibm/icp3.1.0

docker run --rm -v $(pwd):/data -e LICENSE=accept \
   ibmcom/icp-inception:3.1.0 \
   cp -r cluster /data

echo ========================================================
echo Copy private ssh key from /root/.ssh
echo ========================================================
cp -f ~/.ssh/id_rsa ./cluster/ssh_key
```
Define the topology of the ICP cluster by editing and making changes to the hosts file.

```
cd /opt/ibm/icp3.1.0/cluster
```

Edit file `hosts` to define the topology as per below:

```
[master]
192.168.142.101

[worker]
192.168.142.101
192.168.142.102
192.168.142.103

[proxy]
192.168.142.101
```

We will use first VM as a master node and all 3 VMs as worker nodes. The proxy node is also same as the master node.

### Prepare `config.yaml` file

The `config.yaml` file is the one that controls the installation of the IBM Cloud Private. We can decide the features and functions that we want to include in the install.

The detailed list of configuration parameters are available at this [link](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/installing/config_yaml.html).

We will use the following `config.yaml` file. Copy the following contents to your config.yaml file.

```
wait_for_timeout: 600
skip_pre_check: true
firewall_enabled: false
auditlog_enabled: false
federation_enabled: false
secure_connection_enabled: false
network_type: calico
network_cidr: 10.1.0.0/16
service_cluster_ip_range: 10.0.0.1/24
kubelet_extra_args: ["--fail-swap-on=false"]
cluster_name: servicemesh
default_admin_user: admin
default_admin_password: admin
install_docker: false
private_registry_enabled: false
image_repo: ibmcom
offline_pkg_copy_path: /temp
management_services:
 istio: disabled
 vulnerability-advisor: disabled
 storage-glusterfs: disabled
 storage-minio: disabled
 image-security-enforcement: disabled
 metering: disabled
 monitoring: disabled
 service-catalog: disabled
 custom-metrics-adapter: disabled
```

### Install IBM Cloud Private

```
cd /opt/ibm/icp3.1.0/cluster
docker run --rm \
   --net=host -t \
   --name=inception \
   -e LICENSE=accept \
   -e ANSIBLE_REMOTE_TEMP=/temp \
   -v $(pwd):/installer/cluster \
   ibmcom/icp-inception:3.1.0 \
   install -vv
```

The Ansible install will begin and it may take from 45 to 1 hour to build the IBM Cloud Private Cluster. The installer will download required docker images from Docker hub for different components. If installation fails, look through the message and in most case, this might be due to the slow Internet connection, missing dependencies on OS for some packages, insufficient CPU and/or memory. You can restart the installation using same command. To start clean again, you can uninstall incomplete previous install and then try again.

How to uninstall in case you need to restart it again.
```
cd /opt/ibm/icp3.1.0/cluster
docker run --rm \
   --net=host -t \
   -e LICENSE=accept \
   -v $(pwd):/installer/cluster \
   ibmcom/icp-inception:3.1.0 \
   uninstall -vv

```

After Install is done successfully, you will see the URL of the Web Console.

The last few lines of the install output may look like as below:

```
PLAY RECAP ******************************************************************************************************************************************************
192.168.142.101            : ok=140  changed=79   unreachable=0    failed=0   
192.168.142.102            : ok=87   changed=46   unreachable=0    failed=0   
192.168.142.103            : ok=82   changed=40   unreachable=0    failed=0   
localhost                  : ok=250  changed=151  unreachable=0    failed=0   


POST DEPLOY MESSAGE *********************************************************************************************************************************************

The Dashboard URL: https://192.168.142.101:8443, default username/password is admin/admin

Playbook run took 0 days, 0 hours, 39 minutes, 29 seconds
```

### Install kubectl

Install kubectl on the guest VM to run the command line tool.

```
cd /opt/ibm/icp3.1.0/cluster
docker run --rm -v $(pwd):/data -e LICENSE=accept \
   ibmcom/icp-inception:3.1.0 \
   cp -r /usr/local/bin/kubectl /data

chmod +x kubectl
cp kubectl /bin

```

### Install cloudctl and helm client

The `cloudctl` tool is superset of cloud foundry tool to run command line tools to authenticate with IBM Cloud Private so that helm install can be done using `mTLS`. Their are many other capabilities of `cloudctl`.

```
MASTERNODE=192.168.142.101
curl -kLo /tmp/helm-linux-amd64-v2.9.1.tar.gz https://$MASTERNODE:8443/api/cli/helm-linux-amd64.tar.gz
curl -kLo /tmp/cloudctl-linux-amd64-3.1.0-715 https://$MASTERNODE:8443/api/cli/cloudctl-linux-amd64
mv /tmp/cloudctl-linux-amd64-3.1.0-715 /bin/cloudctl
chmod +x /bin/cloudctl
tar xvfz /tmp/helm-linux-amd64-v2.9.1.tar.gz -C /tmp linux-amd64/helm
mv /tmp/linux-amd64/helm /bin
chmod +x /bin/helm
```

### Authenticate using cloudctl

```
MASTERNODE=102.168.142.101
CLUSTERNAME=servicemesh
cloudctl login -n kube-system -u admin -p admin -a https://$MASTERNODE:8443 --skip-ssl-validation -c "id-$CLUSTERNAME-account"
```

After the above command is run, it will set the context for the `kubectl` command to authenticate to the IBM Cloud Private cluster. It will create `~/.kube` and create the context.

The sample output from the above command will be:

```
Authenticating...
OK

Targeted account servicemesh Account (id-servicemesh-account)

Targeted namespace kube-system

Configuring kubectl ...
Property "clusters.servicemesh" unset.
Property "users.servicemesh-user" unset.
Property "contexts.servicemesh-context" unset.
Cluster "servicemesh" set.
User "servicemesh-user" set.
Context "servicemesh-context" created.
Switched to context "servicemesh-context".
OK

Configuring helm: /root/.helm
OK
```  

After above, you can run `kubectl` commands.

For example:
```
# kubectl -n kube-system get nodes -o wide
NAME              STATUS    ROLES                                 AGE       VERSION       INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
192.168.142.101   Ready     etcd,management,master,proxy,worker   1h        v1.11.1+icp   192.168.142.101   <none>        CentOS Linux 7 (Core)   3.10.0-693.11.6.el7.x86_64   docker://18.3.1
192.168.142.102   Ready     worker                                1h        v1.11.1+icp   192.168.142.102   <none>        CentOS Linux 7 (Core)   3.10.0-693.11.6.el7.x86_64   docker://18.3.1
192.168.142.103   Ready     worker                                1h        v1.11.1+icp   192.168.142.103   <none>        CentOS Linux 7 (Core)   3.10.0-693.11.6.el7.x86_64   docker://18.3.1
```

> Remember: After you login to the ICP cluster using cloudctl login command, an authentication token is generated which is valid only for 12 hours. After expiry of the token, you will need to run the `cloudctl login` command.

### Non-expiring Login

If we authenticate using certificates, we can eliminate 12 hour limit. We can do this by using a different home for kubectl config.

Here is an example:

```
ICP_HOMEDIR=/opt/ibm/icp3.1.0
MYKUBE=$HOME/.mykube
export KUBECONFIG=$MYKUBE/config
mkdir -p $MYKUBE/conf
sudo cp $ICP_HOMEDIR/cluster/cfc-certs/kubecfg.key $MYKUBE/conf
sudo cp $ICP_HOMEDIR/cluster/cfc-certs/kubecfg.crt $MYKUBE/conf
sudo chown -R $(echo $(id -u)).$(echo $(id -g)) $MYKUBE/conf
cat << EOF > $MYKUBE/config
apiVersion: v1
kind: Config
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://$MASTERNODE:8001
  name: servicemesh
contexts:
- context:
    cluster: servicemesh
    namespace: default
    user: $DEFAULTUSERNAME
  name: servicemesh
current-context: servicemesh
users:
- name: $DEFAULTUSERNAME
  user:
    client-certificate: $MYKUBE/conf/kubecfg.crt
    client-key: $MYKUBE/conf/kubecfg.key
EOF
```

After we create a different home dir for `kubeconfig`, we can set the context to this.

```
kubectl config use-context servicemesh
```

Now, we can run `kubectl` command which will not expire as we are using certificate based authentication, which was the case `cloudctl login` but that comes with a bearer token.

This method is good if we work directly from the server.

### Launch IBM Cloud Private Web UI

Open URL https://192.168.142.101:8443 from a browser. My personal choice is Google Chrome.

Use user id `admin` and password `admin`.

This will show the web UI of the IBM Cloud Private. Go through this to familiarize yourself.

## Configure Gluster

`GlusterFS` (Gluster File System) is an open source distributed scale out file system to build large storage clusters.

`Heketi` - It is a Gluster Volume Manager that provides a REST API to create / manage Gluster volumes. Heketi makes it easy for Kubernetes to provision volumes. You can use Gluster/Heketi with IBM Cloud Private. It can be installed at the time of initial installation or after the install. With the use of Heketi, it is possible to use dynamic volume provisioning.

Even though, IBM Cloud Private has support for GlusterFS and Heketi, we will be using Gluster manually to provision LVs for persistent storages. From learning standpoint, this will be a good exercise.

### Install glusterFS server

> The Gluster 4.1 is already installed on all VMs if you downloaded them.

Repeat this on all VMs.

These instructions are for building your own Gluster on all VMs.

```
yum install centos-release-gluster
```

The above will install an epel repo in `/etc/yum.repos.d`.

```
yum install glusterfs-server
```

After install, run `gluster --version` to check the version.

```
# gluster --version
glusterfs 4.1.4
Repository revision: git://git.gluster.org/glusterfs.git
Copyright (c) 2006-2016 Red Hat, Inc. <https://www.gluster.org/>
```

### Build GlusterFS Cluster

You need minimum 2 nodes to build a GlusterFS cluster (or domain). It is a likelihood that you may run into split-brain issues with 2 nodes. It is recommended to use minimum 3 nodes.

We will use all 3 nodes to build a GlusterFS domain.

Ideally - you should use separate VMs to host Gluster hosts and do not mix them with workker nodes. In our case, we will use same workker node to also host gluster servers.


#### Create GlusterFS domain

The script [/root/bin/scripts/01-glusterdomain](/Gluster/01-glusterdomain) creates a 3 node GlusterFS domain.

Run script to create 3 node gluster peer domain (From 1st node only)

```
cd /root/bin/scripts
./01-glusterdomain
```

Check the `gluster peer status`

```
gluster peer status
ssh node02 gluster peer status
ssh node03 gluster peer status
```

The sample output of the `gluster peer status` is:
```
# gluster peer status
Number of Peers: 2

Hostname: 192.168.142.102
Uuid: d0852167-9fd8-4f84-afb0-65ab60e3b6c8
State: Peer in Cluster (Connected)

Hostname: 192.168.142.103
Uuid: 9c3b4057-bb44-4e2c-a9db-61d807da8e6a
State: Peer in Cluster (Connected)
```

#### Create Logical Volume

The script [/root/bin/scripts/02-glusterlvm](/Gluster/02-glusterlvm) creates logical volumes.

We added a 40GB thin provisioned disk in each VM. We will use this disk to create LVs on each VMs.


Check the disk name
```
ssh node01 lsblk | grep 40G
ssh node02 lsblk | grep 40G
ssh node03 lsblk | grep 40G
```

In our VM setup, this disk is `/dev/sdb`. It is better to always double check.

On node01, run
```
cd /root/bin/scripts
./02-glusterlvm /dev/sdb brick1
```

On node01, we will use /dev/sdb to create 2 LVMs, create file system, create entries in `/etc/fstab` and mount them.

We create 2 mount points:

```
# df -h | grep glusterfs
```
Output:
```
/dev/mapper/ServiceMesh-cockroachdb  4.0G   33M  4.0G   1% /mnt/glusterfs/cockroachdb
/dev/mapper/ServiceMesh-web          4.0G   33M  4.0G   1% /mnt/glusterfs/web
```

On node02, run

```
ssh node02
cd /root/bin/scripts
./02-glusterlvm /dev/sdb brick2
```

Check mount points

```
# df -h | grep glusterfs
```
Output:
```
/dev/mapper/ServiceMesh-cockroachdb  4.0G   33M  4.0G   1% /mnt/glusterfs/cockroachdb
/dev/mapper/ServiceMesh-web          4.0G   33M  4.0G   1% /mnt/glusterfs/web
```

Type `exit` to logout from node02

On node03, run
```
ssh node03
cd /root/bin/scripts
./02-glusterlvm /dev/sdb brick3
```

Check mount points

```
# df -h | grep glusterfs
```
Output:
```
/dev/mapper/ServiceMesh-cockroachdb  4.0G   33M  4.0G   1% /mnt/glusterfs/cockroachdb
/dev/mapper/ServiceMesh-web          4.0G   33M  4.0G   1% /mnt/glusterfs/web
```

Type `exit` to logout from node03

#### Create Gluster Volume

The script [/root/bin/scripts/03-glustervol](/Gluster/03-glustervol) will create gluster volume.

Run this from 1st VM only.
```
cd /root/bin/scripts
./03-glustervol
```

#### Start Gluster Volume

The script [/root/bin/scripts/04-glusterstartstop](/Gluster/04-glusterstartstop) will start gluster volumes.

Start Gluster volume - do this only from 1st node

```
cd /root/bin/scripts
./04-glusterstartstop start
```

Check Gluster volume status. As a sanity check, all tasks `Online` status should show `Y`

```
# gluster volume status cockroachdb
Status of volume: cockroachdb
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick 192.168.142.101:/mnt/glusterfs/cockro
achdb/brick1                                49152     0          Y       19740
Brick 192.168.142.102:/mnt/glusterfs/cockro
achdb/brick2                                49152     0          Y       15453
Brick 192.168.142.103:/mnt/glusterfs/cockro
achdb/brick3                                49152     0          Y       15662
Self-heal Daemon on localhost               N/A       N/A        Y       19993
Self-heal Daemon on 192.168.142.103         N/A       N/A        Y       15780
Self-heal Daemon on 192.168.142.102         N/A       N/A        Y       15528

Task Status of Volume cockroachdb
------------------------------------------------------------------------------
There are no active volume tasks

# gluster volume status web
Status of volume: web
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick 192.168.142.101:/mnt/glusterfs/web/br
ick1                                        49153     0          Y       19969
Brick 192.168.142.102:/mnt/glusterfs/web/br
ick2                                        49153     0          Y       15504
Brick 192.168.142.103:/mnt/glusterfs/web/br
ick3                                        49153     0          Y       15724
Self-heal Daemon on localhost               N/A       N/A        Y       19993
Self-heal Daemon on 192.168.142.102         N/A       N/A        Y       15528
Self-heal Daemon on 192.168.142.103         N/A       N/A        Y       15780

Task Status of Volume web
------------------------------------------------------------------------------
There are no active volume tasks
```

## Synchronize Clocks

Before we install cockroachdb, we need to make sure that the clocks are in sync.

This is necessary to keep clocks in sync within 500 ms so that cockroachdb can function well in a cluster environment.

Run script [/root/bin/scripts/syncclocks](/Scripts/syncclocks) which does the following:

* Create `/bin/synclock` - which checks the time difference between VMs.
* Create a cronjob to run this sript every 15 minutes - though not necessary as NTP should do its job. Sync time in VMs is challenging due to time sync at Windows level, from VM to the guest and then using NTP between all nodes using ntpd.
* Create `/bin/hardsync` which will do a force time sync between VMs.

After running `syncclocks`, run `synclock`

```
# synclock
=================================================================================
Checking time diff between 192.168.142.101 with the node01 .... 0.000 milliseconds
Checking time diff between 192.168.142.102 with the node01 .... -2.141 milliseconds
Checking time diff between 192.168.142.103 with the node01 .... -1.983 milliseconds
=================================================================================
```
