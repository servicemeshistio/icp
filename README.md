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

### Download VMs

If you like to download the three base VM images to install IBM Cloud Private 3 node community edition cluster, you can download 7z of images from [here](#). If you download VMs, you can skip Prepare VM section as the following steps are already taken care in the VMs.

Continue with these steps, should you choose to use your own VM install.   

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
* Make sure that you can run `ping -c4 google.com` from each VM to make sure that Internet access is available.  
* Configure `passwordless` SSH access between all VMs. You can refer to this [link](https://github.com/vikramkhatri/sshsetup) for setting up SSH if you need automation.
* Configure [direct LVM](Scripts/directLVM) access for a disk to be used by Docker backend. Use 100 GB disk (vmdk) that you attached to the VM as an argument to the script. Run this on all nodes
* Install docker engine on all nodes
* Configure [dnsmasq](/Scripts/dnsmasq.md) in first VM for DNS routing if host names are not defined in the upstream DNS server. Optional - but useful
* Configure [ntpd](Scripts/ntpd.md) on 1st VM so that all VMs time is in sync. Since, we are going to use 3 replicas for CockroachDB, the time must be within 500ms for it work properly.   

## Build a 3 node IBM Cloud Private Cluster

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

The cloudctl tool is superset of cloud foundry tool to run command line tools to authenticate with IBM Cloud Private so that helm install can be done using mTLS. Their are many other capabilities of `cloudctl`.

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
