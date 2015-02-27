#**Configure Flannel**

* Check and explore the versions of software you have.  This should be the same on all nodes.

```
rpm -qa | egrep "etc|docker|flannel|kube"
rpm -ql docker
rpm -ql etcd
rpm -ql kubernetes
rpm -ql flannel
rpm -qi docker
rpm -qi etcd
rpm -qi kubernetes
rpm -qi flannel
rpm -qd docker
rpm -qd etcd
rpm -qd kubernetes
rpm -qd flannel
rpm -qc docker
rpm -qc etcd
rpm -qc kubernetes
rpm -qc flannel
```

Perform the following on the master node (pick one):

* Look at networking before flannel configuration.

```
ip a
```

* Start etcd.


```
systemctl start etcd; systemctl enable etcd
systemctl status etcd
```


* Configure Flannel by creating a `flannel-config.json` in your current directory.  The contents should be:


**NOTE:** Choose an IP range that is *NOT* part of the public IP address range.

```json
{
    "Network": "18.0.0.0/16",
    "SubnetLen": 24,
    "Backend": {
        "Type": "vxlan",
        "VNI": 1
     }
}
```

* Add the configuration to the etcd server. Use the public IP address of the master node. If this is an OpenStack VM you will need to look up the public IP address on the OpenStack Horizon dashboard.


```
curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/config -XPUT --data-urlencode value@flannel-config.json
```

Example of successful output:

```json
{"action":"set","node":{"key":"/coreos.com/network/config","value":"{\n    \"Network\": \"18.0.0.0/16\",\n    \"SubnetLen\": 24,\n    \"Backend\": {\n        \"Type\": \"vxlan\",\n        \"VNI\": 1\n     }\n}\n","modifiedIndex":3,"createdIndex":3}}-bash-4.2#
```

* Verify the key exists.  Use the IP Address of your etcd / master node.


```
curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/config
```

* Backup the flannel configuration file.


```
cp /etc/sysconfig/flanneld{,.orig}
```

* Configure flannel using the network interface of the system. This is commonly `eth0` but might be `ens3`. Use `ip a` to list network interfaces. This should not be necessary on most systems unless they have multiple network interfaces.  In which case you will want to use the interface capable of talking to other nodes in the cluster.

```
sed -i 's/#FLANNEL_OPTIONS=""/FLANNEL_OPTIONS="eth0"/g' /etc/sysconfig/flanneld
```


* Edit `/etc/sysconfig/flanneld` file with the public IP address of the master node (If you are using OS1 use Private Address below for x.x.x.x). You must make a change here.


```
# Flanneld configuration options

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD="http://x.x.x.x:4001"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_KEY="/coreos.com/network"

# Any additional options that you want to pass
FLANNEL_OPTIONS="eth0"
```


* Start up the flanneld service.


```
systemctl start flanneld; systemctl enable flanneld
systemctl status flanneld
```

* Check the interfaces on the host now. Notice there is now a flannel.1 interface.


```
ip a
```

* The docker and flannel network interfaces must match otherwise docker will fail to start. If Docker fails to load, or the flannel IP is not set correctly, reboot the system.  It is also possible to stop docker, delete the docker0 network interface, and then restart docker after flannel has started.  But rebooting is easier. Do not move forward until you can issue an _ip a_ and the _flannel_ and _docker0_ interface are on the same subnet.

Now that master is configured, lets configure the other nodes called "minions" (minion{1,2}).

**Perform the following on the other two atomic host minions:**

* Use curl to check firewall settings from each minion to the master.  We need to ensure connectivity to the etcd service.  You may want to set up your `/etc/hosts` file for name resolution here.  If there are any issues, just fall back to IP addresses for now. **NOTE:** For OpenStack nodes use the *private IP address* of the master.


```
curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/config
```

For some of the steps below, it might help to set up ssh keys on the master and copy those over to the minions, e.g. with ssh-copy-id.  You also might want to set hostnames on the minions and edit your `/etc/hosts` files on all nodes to reflect that.

From the master:

* Copy over flannel configuration to the minions, both of them. Use `scp` or copy the file contents manually. In the OS1 (OpenStack) environment, you can not scp files without moving some keys around.  It might just be quicker to copy and paste the contents.  In a KVM hosted environment, feel free to _scp_ files around.


```
scp /etc/sysconfig/flanneld x.x.x.x:/etc/sysconfig/.
```


* Restart flanneld on both of the minions.

```
systemctl restart flanneld
systemctl enable flanneld
```

* Check the new interface on both of the minions.

```
ip a l flannel.1
```

From any node in the cluster, check the cluster members by issuing a query to etcd via curl.  You should see that three servers have consumed subnets.  You can associate those subnets to each server by the MAC address that is listed in the output.


```
curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/subnets | python -mjson.tool
```


* From all nodes, review the `/run/flannel/subnet.env` file.  This file was generated automatically by flannel.


```
cat /run/flannel/subnet.env
```

* Check the network on the minion. Docker will fail to load if the docker and flannel network interfaces are not setup correctly. Reboot each minion. Again it is possible to fix this by hand, but rebooting is easier.  A functioning configuration should look like the following; notice the docker0 and flannel.1 interfaces.


```
ip a
1: lo:  mtu 65536 qdisc noqueue state UNKNOWN group default
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever

2: eth0:  mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
link/ether 52:54:00:15:9f:89 brd ff:ff:ff:ff:ff:ff
inet 192.168.121.166/24 brd 192.168.121.255 scope global dynamic eth0
valid_lft 3349sec preferred_lft 3349sec
inet6 fe80::5054:ff:fe15:9f89/64 scope link
valid_lft forever preferred_lft forever

3: flannel.1:  mtu 1450 qdisc noqueue state UNKNOWN group default
link/ether 82:73:b8:b2:2b:fe brd ff:ff:ff:ff:ff:ff
inet 18.0.81.0/16 scope global flannel.1
valid_lft forever preferred_lft forever
inet6 fe80::8073:b8ff:feb2:2bfe/64 scope link
valid_lft forever preferred_lft forever

4: docker0:  mtu 1500 qdisc noqueue state DOWN group default
link/ether 56:84:7a:fe:97:99 brd ff:ff:ff:ff:ff:ff
inet 18.0.81.1/24 scope global docker0
valid_lft forever preferred_lft forever
```

Do not move forward until all three nodes have the docker and flannel interfaces on the same subnet.

At this point the flannel cluster is set up and we can test it. We have etcd running on the master node and flannel / Docker running on minion{1,2} minions. Next steps are for testing cross-host container communication which will confirm that Docker and flannel are configured properly.

##Test the flannel configuration

From each minion, pull a Docker image for testing. In our case, we will use fedora:20.

* Issue the following on minion1.


```
docker run -it fedora:20 bash
```

* This will place you inside the container. Check the IP address.


```
ip a l eth0
5: eth0:  mtu 1450 qdisc noqueue state UP group default
link/ether 02:42:0a:00:51:02 brd ff:ff:ff:ff:ff:ff
inet 18.0.81.2/24 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::42:aff:fe00:5102/64 scope link
valid_lft forever preferred_lft forever
```


You can see here that the IP address is on the flannel network.

* Issue the following commands on minion2:


```
docker run -it fedora:20 bash

ip a l eth0
5: eth0:  mtu 1450 qdisc noqueue state UP group default
link/ether 02:42:0a:00:45:02 brd ff:ff:ff:ff:ff:ff
inet 18.0.69.2/24 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::42:aff:fe00:4502/64 scope link
valid_lft forever preferred_lft forever
```


* Now, from the container running on minion2, ping the container running on minion1:


```
ping 18.0.81.2
PING 18.0.81.2 (18.0.81.2) 56(84) bytes of data.
64 bytes from 18.0.81.2: icmp_seq=2 ttl=62 time=2.93 ms
64 bytes from 18.0.81.2: icmp_seq=3 ttl=62 time=0.376 ms
64 bytes from 18.0.81.2: icmp_seq=4 ttl=62 time=0.306 ms
```

* You should have received a reply. That is it. flannel is set up on the two minions and you have cross host communication. Etcd is set up on the master node. Next step is to overlay the cluster with kubernetes.

Do not move forward until you can ping from container to container on different hosts.

Exit the containers on each node when finished.

