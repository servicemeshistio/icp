## ntpd server configuration

It is very critical that time is kept in sync between all VMs. The time difference cannot be more than 500 ms otherwise it will affect CockroachDB pods and they will refuce to run.

Even though System Administrators know how to set up ntp properly, it is appropriate to mention this here so that it is easier for you to configure it in your environment.

In our case, we are going to use `node01` as the time server and rests of the nodes will sync time with the first node. This approach is useful in an air gap environment.

### First VM - NTP Server

File `/etc/ntp.conf`

The `tinker` line is must if using VM environment.

The `server` address `127.127.1.0` is significant as it tells NTP that this is the local time server.

```
tinker panic 0

restrict default kod nomodify notrap no#peer noquery
restrict 127.0.0.1
restrict -6 default kod nomodify notrap no#peer noquery
restrict -6 ::1

server 127.127.1.0

driftfile /var/lib/ntp/drift
```

### Second VM - Use first VM as the time server

The configuration is similar to first node except that instead of `server`, we are using `peer` which points to the `192.168.142.101` as the time server.

```
tinker panic 0

restrict default kod nomodify notrap nopeer noquery
restrict 127.0.0.1
restrict -6 default kod nomodify notrap nopeer noquery
restrict -6 ::1

peer 192.168.142.101

driftfile /var/lib/ntp/drift

```

### Start NTP server

On each node, enable and start ntpd service

```
systemctl enable ntpd
systemctl start ntpd
```

### Monitoring

The ntpd will sync the time but it does on its own pace to allow gradual synchronization.

Sometime, it may be necessary to forcefully sync the time. In that case, you can take help from the script [synclock](/Scripts/synclock)

Usually, you can copy `synclock` to `/bin`.

A sample output is as shown in which if time diff is more than 50 ms, it will forcefully sync the time between VMs.

```
[root@node01 ~]# synclock 50
=================================================================================
Checking time diff between node02 with the node01 .... 0.049 milliseconds
Checking time diff between node03 with the node01 .... 0.010 milliseconds
=================================================================================
```
