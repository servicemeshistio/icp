## Configure dnsmasq

In a Kubernetes environment, the micro services sometime need to access external URLs. If the domain name can be resolved through a domain name server, things will work fine. If there are local services that are not defined in the local domain name server, it will be problematic to resolve the names.  

There are two solutions to this problem:

* Pass host name entries as `hostAliases` when a pod deployment is defined. For example:
```
hostAliases:
  - ip: "192.168.142.110"
    hostnames:
    - "registry.servicemesh.local"
```

* Create a local dns server that will resolve the IP addresses of local hostnames

> Note: Docker copies the contents of `/etc/hosts` but Kubernetes does not allow that and it recommends the use of `hostAliases` a way to pass host names to the running pod.

Since in our case, we do not have an upstream dns server to use, we need to use and `dnsmasq` is a simple one to configure and use.

In our example: We will run a `dnsmasq` service on 1st node and configure it in such a way so that it provides name resolution to the local hosts defined in `/etc/hosts` and then pass control to upstream name server. We will also use this name server to pass control to Kubernetes name server for resolution of internal service names.

## dnsmasq config file

The `/etc/dnsmasq.conf` file

```
domain-needed
bogus-priv
no-resolv
server=/cluster.local/10.0.0.10
server=192.168.142.2
no-hosts
addn-hosts=/etc/dnsmasq.hosts
conf-dir=/etc/dnsmasq.d,.rpmnew,.rpmsave,.rpmorig
```

The `domain-needed` parameter is useful and it tells `dnsmasq` to never forward A or AAAA queries for plain names, without dots or domain parts, to the upstream nameservers. For example, it will not forward name `node01` for IP address resolution to the upstream name server.

The `bogus-priv` - All reverse lookups for private IP ranges (ie 192.168.x.x, etc) which are not found in `/etc/hosts` or configured file or the DHCP leases file are answered with "no such domain" rather than being forwarded upstream.

The `no-resolv` is a singnal to `dnsmasq` to not to use `/etc/resolv.conf` for a list of host name servers. In our case, we will use `server` directive in this configuration file to define the list of dns servers that we want to use.

The  `server=/cluster.local/10.0.0.10` is very useful in which the Kubernetes internal service name resolution is sent to Kubernetes DNS server - which is `kube-dns` or `coreDns` depending upon version of Kubernetes. For example: If we are using name `cockroach.servicemesh.svc.cluster.local` name from the local hosts, the IP address resolution will be sent to 10.0.0.10 since the name matches with `cluster.local`.

The `server=192.168.142.2` is used for the upstream name server when the name is resolved locally. In our case, we are using VMware Workstation and we are using `vmnet8` network adapter - which has the gateway address as `192.168.142.2`. From VMware level, the DNS request will be forwarded to its upstream dns server.

The `no-hosts` is a directive to `dnsmasq` to not read local `/etc/hosts` file.

The  `addn-hosts=/etc/dnsmasq.hosts` is our local file that is equivalent to `/etc/hosts` file. Sometime, the `/etc/hosts` files are overwritten by the provisioning mechanism so that is why it is better to separate this file.

After this config file is created, start the dns server based upon your Linux distribution.

For RHEL or CentOS

```
systemctl enable dnsmasq
systemctl start dnsmasq
```

## Other nodes

In a cluster environment suc as ours with just 3 nodes, it does not make sense to run `dnsmasq` in all VMs. Instead, we will be using `/etc/resolv.conf` on all other nodes to direct all dns related queries to the first VM where we have defined `dnsmasq`.

For example:

The `/etc/resolv.conf` from all other nodes.

```
nameserver 192.168.142.101
domain servicemesh.local
search servicemesh.local
```

The nameserver `192.168.142.101` is the first VM in our case - which is running the `dnsmasq` name server. The name resolution to local hosts, Kubernetes cluster service names and external names will be resolved by our `dnsmasq` server.
