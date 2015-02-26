Demo: March 3rd-4th 2015
Location: Westford

##**BEFORE YOU ARRIVE**
    In order to make best use of the lab time please review the deployment options and ensure either 

1. A working KVM environment or
2. Access to the internal OpenStack OS1 environment.

##**Agenda / High Level Overview:**

1. Deploy Atomic Hosts
2. Configure Flannel
3. Configure K8s
4. Deploy an Application
5. Use SPCs on the Atomic Hosts


#**Deployment**
There are many ways to deploy an Atomic host.  In this lab, we provide guidance for OpenStack or local KVM.

##**Deployment Option 1: Atomic Hosts on OpenStack**
You may use the the Red Hat internal, HSS-supported **OS1** OpenStack service.

1. PREREQUISITE: You must create an OS1 account and upload your SSH public key. Follow the ["Getting Started" instructions](https://mojo.redhat.com/docs/DOC-28082#jive_content_id_Getting_Started)
1. Navigate to [Instances](https://control.os1.phx2.redhat.com/dashboard/project/instances/)
1. Click "Launch Instances"
1 Complete form
  1. Details tab
    * Instance name: arbitrary name. Note the UUID of the image will be appended to the instance name. You may want to use your name in the image so you can easily find it.
    * Flavor: *m1.medium*
    * Instance count: *3*
    * Instance Boot Source: *Boot from image*
      * Image name: *rhel-atomic-cloud-7.1-6*
  1. Access & Security tab
    * select you keypair that was uploaded during OS1 account setup. See [instructions](https://mojo.redhat.com/docs/DOC-28082#jive_content_id_Getting_Started)
    * Security Groups: *Default*
1. Click "Launch"

Three VMs will be created. Once Power State is *Running* you may SSH into the VMs. Your SSH public key will be used.

* Note: Each instance requires a floating IP address in addition to the private OpenStack `172.x.x.x` address. Your OpenStack tenant may automatically assign a floating IP address. If not, you may need to assign it manually. If no floating IP addresses are available, create them.
  1. Navigate to [Access & Security](https://control.os1.phx2.redhat.com/dashboard/project/access_and_security/)
  1. Click "Floating IPs" tab
  1. Click "Allocate IPs to project"
  1. Assign floating IP addresses to each VM instance
* SSH into the VMs with user `cloud-user` and the instance floating IP address. This address will be in the `10.3.xx.xx` range.

```
ssh cloud-user@10.3.xx.xxx
```


##**Deployment Option 2: Atomic Hosts on KVM**

* Grab and extract the Atomic and metadata images from our internal repo.  Use sudo and appropriate permissions.

```
wget http://refarch.cloud.lab.eng.bos.redhat.com/pub/projects/atomic/atomic0-cidata.iso
cp atomic0-cidata.iso /var/lib/libvirt/images/.
wget http://download.eng.bos.redhat.com/rel-eng/Atomic/7/trees/GA.brew/images/20150217.0/cloud/rhel-atomic-host-7.qcow2.gz
cp rhel-atomic-host-7.qcow2.gz /var/lib/libvirt/images/.; cd /var/lib/libvirt/images
gunzip rhel-atomic-host-7.qcow2.gz
```

* Make 3 copies of the image.

```
cp rhel-atomic-host-7.qcow2 rhel-atomic-host-7-1.qcow2
cp rhel-atomic-host-7.qcow2 rhel-atomic-host-7-2.qcow2
cp rhel-atomic-host-7.qcow2 rhel-atomic-host-7-3.qcow2
```

* Use the following commands to install the images. Note: You will need to change the bridge to match your setup, or at least confirm it matches what you have.

```
virt-install --import --name atomic-ga-1 --ram 4096 --vcpus 2 --disk path=/var/lib/libvirt/images/rhel-atomic-host-7-1.qcow2,format=qcow2,bus=virtio --disk path=/var/lib/libvirt/images/atomic0-cidata.iso,device=cdrom --network bridge=br0 --force

virt-install --import --name atomic-ga-2 --ram 4096 --vcpus 2 --disk path=/var/lib/libvirt/images/rhel-atomic-host-7-2.qcow2,format=qcow2,bus=virtio --disk path=/var/lib/libvirt/images/atomic0-cidata.iso,device=cdrom --network bridge=br0 --force

virt-install --import --name atomic-ga-3 --ram 4096 --vcpus 2 --disk path=/var/lib/libvirt/images/rhel-atomic-host-7-3.qcow2,format=qcow2,bus=virtio --disk path=/var/lib/libvirt/images/atomic0-cidata.iso,device=cdrom --network bridge=br0 --force
```

##**Update VMs**

**NOTE:** We will be working on _all three (3)_ VMs. You will probably want to have three terminal windows open.

* Confirm you can login to the hosts:

    Username: cloud-user
    Password: atomic (KVM only)

* Enter sudo shell:

```
sudo -i
```


* Update all of the atomic hosts. The following commands will change you to the GA.staging tree, which should be what customers will see.

```
atomic host status
ostree remote add --set=gpg-verify=false GA.brew http://download.eng.bos.redhat.com/rel-eng/Atomic/7/trees/GA.brew/repo/
rpm-ostree rebase GA.brew:rhel-atomic-host/7/x86_64/standard
```
This will fetch a new tree and display any RPM changes. **Note:** Customers will not have to add a remote. They will issue command `atomic host upgrade` then reboot.

* Check the atomic tree version

```
atomic host status
```

Note the `*` identifies the active version.

* Reboot the VMs to switch to updated tree.

```
systemctl reboot
```

* After the VMs have rebooted, SSH into each and enter sudo shell:

```
sudo -i
```

* Check your version with atomic. The `*` pointer should now be on the new tree.

```
atomic host status
```


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
rpm -qi flannel
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


* Configure Flannel by creating a flannel-config.json in your current directory.  The contents should be:


**NOTE:** Choose an IP range that is *NOT* part of the public IP address range.

```
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

```
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


* Edit `/etc/sysconfig/flanneld` file with the public IP address of the master node. You must make a change here.


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
inet 10.0.81.0/16 scope global flannel.1
valid_lft forever preferred_lft forever
inet6 fe80::8073:b8ff:feb2:2bfe/64 scope link
valid_lft forever preferred_lft forever

4: docker0:  mtu 1500 qdisc noqueue state DOWN group default
link/ether 56:84:7a:fe:97:99 brd ff:ff:ff:ff:ff:ff
inet 10.0.81.1/24 scope global docker0
valid_lft forever preferred_lft forever
```

Do not move forward until all three nodes have the docker and flannel interfaces on the same subnet.  

At this point the flannel cluster is set up and we can test it. We have etcd running on the master node and flannel / Docker running on minion{1,2} minions. Next steps are for testing cross-host container communication which will confirm that Docker and flannel are configured properly.

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
inet 10.0.81.2/24 scope global eth0
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
inet 10.0.69.2/24 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::42:aff:fe00:4502/64 scope link
valid_lft forever preferred_lft forever
```


* Now, from the container running on minion2, ping the container running on minion1:


```
ping 10.0.81.2
PING 10.0.81.2 (10.0.81.2) 56(84) bytes of data.
64 bytes from 10.0.81.2: icmp_seq=2 ttl=62 time=2.93 ms
64 bytes from 10.0.81.2: icmp_seq=3 ttl=62 time=0.376 ms
64 bytes from 10.0.81.2: icmp_seq=4 ttl=62 time=0.306 ms
```

* You should have received a reply. That is it. flannel is set up on the two minions and you have cross host communication. Etcd is set up on the master node. Next step is to overlay the cluster with kubernetes.

Do not move forward until you can ping from container to container on different hosts.

Exit the containers on each node when finished.


##Configure Kubernetes##

The kubernetes package provides several services

* kube-apiserver
* kube-scheduler
* kube-controller-manager
* kubelet, kube-proxy

These services are managed by systemd and the configuration resides in a central location, `/etc/kubernetes`. We will break the services up between the hosts.  The first host, *master*, will be the kubernetes master.  This host will run kube-apiserver, kube-controller-manager, and kube-scheduler. In addition, the master will also run _etcd_. The remaining hosts, the *minions* or *nodes*, will run kubelet, proxy, cadvisor and docker.

###Prepare the hosts

* Backup the kubernetes configuration files on each system (master and nodes) before continuing.

```
for i in $(ls /etc/kubernetes/*); do cp $i{,.orig}; echo "Making a backup of $i"; done
```


* Edit `/etc/kubernetes/config` to be the same on **all hosts**. For OpenStack VMs we will be using the *private IP address* of the master host.

```
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow_privileged=false"

# How the replication controller and scheduler find the apiserver
KUBE_MASTER="--master=http://MASTER_PRIV_IP_ADDR:8080"
```

####Configure the kubernetes services on the master

* Edit `/etc/kubernetes/apiserver` to appear as such:

```       
# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd_servers=http://MASTER_PRIV_IP_ADDR:4001"

# The address on the local server to listen to.
KUBE_API_ADDRESS="--address=0.0.0.0"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--portal_net=10.254.0.0/16"

# Add you own!
KUBE_API_ARGS=""
```

* Edit `/etc/kubernetes/controller-manager` to appear as such.  Substitute your minion IPs here.:

```
# Comma separated list of minions
KUBELET_ADDRESSES="--machines=MINION_PRIV_IP_1,MINION_PRIV_IP_2"
```

* Start the appropriate services on master:

```
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do 
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES 
done
```

####Configure the kubernetes services on the minions

**NOTE:** Make these changes on each minion.

***We need to configure the kubelet and start the kubelet and proxy***

* Edit `/etc/kubernetes/kubelet` to appear as below.  Make sure you substitute kublet or minion IP addresses appropriately.

```
# The address for the info server to serve on
KUBELET_ADDRESS="--address=0.0.0.0"

# this MUST match what you used in KUBELET_ADDRESSES on the controller manager
# unless you used what hostname -f shows in KUBELET_ADDRESSES.
KUBELET_HOSTNAME="--hostname_override=x.x.x.x"

# how the kubelet finds the apiserver
KUBELET_API_SERVER="--api_servers=http://MASTER_PRIV_IP_ADDR:8080"

# Add your own!
KUBELET_ARGS=""
```

* edit `/etc/kubernetes/proxy` to appear as below.

```
# How the proxy find the apiserver
KUBE_PROXY_ARGS="--master=http://MASTER_PRIV_IP_ADDR:8080"
```

* Start the appropriate services on the minions.

```
for SERVICES in kube-proxy kubelet docker; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```

*You should be finished!*

* Check to make sure the cluster can see the minions from the master.

```
# kubectl get minions
NAME                LABELS              STATUS
192.168.121.147     <none>              Ready
192.168.121.101     <none>              Ready
```

**The cluster should be running! Launch a test pod.**

##Deploy an application##


* Create a file on master named `apache.json` that looks as such:

```
{
        "apiVersion": "v1beta1",
        "kind": "Pod",
        "id": "apache",
        "namespace": "default",
        "labels": {
          "name": "apache"
        },
        "desiredState": {
          "manifest": {
          "version": "v1beta1",
          "id": "apache",
          "volumes": null,
          "containers": [
            {
              "name": "master",
              "image": "fedora/apache",
              "ports": [
            {
              "containerPort": 80,
              "hostPort": 80,
              "protocol": "TCP"
            }
          ],
        }
      ],
      "restartPolicy": {
      "always": {}
      }
    },
  },
}
```

This json file is describing the attributes of the application environment. For example, it is giving it a "kind", "id", "name", "ports", and "image". Since the fedora/apache images doesn't exist in our environment yet, it will be pulled down automatically as part of the deployment process.

For more information about which options can go in the schema, check out the docs on the [kubernetes github page](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/docs).

* Deploy the fedora/apache image via the `apache.json` file.

```
kubectl create -f apache.json
```


* This command exits immediately, returning the value of the label, `apache`. You can monitor progress of the operations with these commands:
On the master (master) -

```
journalctl -f -l -xn -u kube-apiserver -u etcd -u kube-scheduler
```

* On the minion (fed-minion) -

```
journalctl -f -l -xn -u kubelet -u kube-proxy -u docker
```


* After the pod is deployed, you can also list the pod.  I have a few pods running here.

```
# kubectl get pods
POD                 IP                  CONTAINER(S)        IMAGE(S)            HOST                LABELS              STATUS
apache              10.0.53.3           master              fedora/apache       192.168.121.147/    name=apache         Running
mysql               10.0.73.2           mysql               mysql               192.168.121.101/    name=mysql          Running
redis-master        10.0.53.2           master              dockerfile/redis    192.168.121.147/    name=redis-master   Running
```

The state might be 'Pending'. This indicates that docker is still attempting to download and launch the container.

* You can get even more information about the pod like this.

```
kubectl get pods --output=json apache
```

* Finally, on the minion (minion), check that the service is available, running, and functioning.

```
docker images
REPOSITORY TAG IMAGE ID CREATED VIRTUAL SIZE
kubernetes/pause latest 6c4579af347b 7 weeks ago 239.8 kB
fedora/apache latest 6927a389deb6 3 months ago 450.6 MB

docker ps -l
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
05c69c00ea48 fedora/apache:latest "/run-apache.sh" 2 minutes ago Up 2 minutes k8s--master.3f918229--apache.etcd--8cd6efe6_-_3a95_-_11e4_-_b618_-_5254005318cb--9bb78458

curl http://localhost
Apache
```

* To delete the container.

```
# /usr/bin/kubectl --server=http://master:8080 delete pod apache
```

Of course this just scratches the surface. I recommend you head off to the kubernetes github page and follow the [guestbook example](https://github.com/GoogleCloudPlatform/kubernetes/tree/754a2a8305c812121c3845d8293efdd819b6a704/examples/guestbook-go). It is a bit more complicated but should expose you to more functionality.

## Use SPCs on the Atomic Hosts

The goal here is to explore some of the images that we will be distributing when Atomic GAs.  We are trying to keep the Atomic image as small as possible where it makes sense.  This means that anything else that gets added to the Atomic host will have to be inside a container.  The examples we will go over in this section are rsyslog, sadc and rhel-tools.  For this to work you need at least two functioning Atomic hosts.

###Using rsyslog

The rsyslog container runs in the background for the purposes of managing logs. We will cover two scenarios:

1. Quick smoke test to make sure logging is working on the localhost.
2. Remote logging.  We will send some logs over the network.

####Scenario 1: Quick Smoketest

* Check the environment before.  You may have a couple of residual images.  You should not have any rsyslog images. You can perform this on the master node.

```
docker images
REPOSITORY                                               TAG                 IMAGE ID            CREATED             VIRTUAL SIZE

docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

* Install the container.

```
atomic install --name rsyslog docker-registry.usersys.redhat.com/atomcga/rsyslog
docker run --rm --privileged -v /:/host -e HOST=/host -e IMAGE=docker-registry.usersys.redhat.com/atomcga/rsyslog -e NAME=rsyslog docker-registry.usersys.redhat.com/atomcga/rsyslog /bin/install.sh
Installing file at /host//etc/rsyslog.conf in place of existing empty directory
Installing file at /host//etc/rsyslog.conf
Installing file at /host//etc/sysconfig/rsyslog
```

* Run the container.


```
atomic run --name rsyslog docker-registry.usersys.redhat.com/atomcga/rsyslog

```

* Check the environment after the install.  You should now see the rsyslog image, but no container yet.

```
docker images

docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

* Run it         

```
atomic run --name rsyslog docker-registry.usersys.redhat.com/atomcga/rsyslog
Pulling repository docker-registry.usersys.redhat.com/atomcga/rsyslog
72fff92e8533: Download complete 
Status: Downloaded newer image for docker-registry.usersys.redhat.com/atomcga/rsyslog:latest
docker run -d --privileged --name rsyslog -v /etc/pki/rsyslog:/etc/pki/rsyslog -v /etc/rsyslog.conf:/etc/rsyslog.conf -v /etc/rsyslog.d:/etc/rsyslog.d -v /var/log:/var/log -v /var/lib/rsyslog:/var/lib/rsyslog -v /run/log:/run/log -v /etc/machine-id:/etc/machine-id -v /etc/localtime:/etc/localtime -v /etc/hostname:/etc/hostname -e IMAGE=docker-registry.usersys.redhat.com/atomcga/rsyslog -e NAME=rsyslog --restart=always docker-registry.usersys.redhat.com/atomcga/rsyslog /bin/rsyslog.sh
c86ba2e7e205cadfff1facc4ebb820e849c8a8cc57c22caabffbbe7c7d3d9f8d
```

* Check the environment.  Now you should see a running rsyslog container.

```
# docker ps
CONTAINER ID        IMAGE                                                      COMMAND             CREATED             STATUS              PORTS               NAMES
c86ba2e7e205        docker-registry.usersys.redhat.com/atomcga/rsyslog:7.1-2   "/bin/rsyslog.sh"   2 minutes ago       Up 41 seconds                           rsyslog       
```

* How do I use it (scenario 1: single host smoke test)?  In one terminal on the master node, watch the logs.

```
tail -f /var/log/messages
```

* In another terminal, still on the master, generate a log.

```
logger test
```

* Back in the first terminal, you should see an entry with “test”

```
Feb  9 16:31:36 localhost vagrant: test
```

####Scenario 2: Remote Logging

#THIS MIGHT NOT WORK FOR SA TRAINING.  THE PROBLEM IS THAT WE NEED TO ADD A --net=host TO THE LABEL.  SKIP THIS SECTION FOR NOW.  2/23/2015 - scollier


Stop the rsyslog container on the master node.  We are going to make a change to the `/etc/rsyslog.conf` file and we will need to re-read that.  Use following steps to stop the container. After the container is stopped you can change the file and restart the container.

```
docker ps
docker stop <container id>
docker ps -a
docker rm <container id>
```

On the master node, point it to the rsyslog server in the `/etc/rsyslog.conf`.  Substitute your IP address here. The entry below is at the bottom of the file.

```
*.* @@192.168.121.147:514
```

* Now switch to the rsyslog server (kubelet 1).  Configure the rsyslog server.  In this case, the rsyslog server will be minion / kublet 1 server. Install rsyslog on the kublet server.

```
atomic install --name rsyslog docker-registry.usersys.redhat.com/atomcga/rsyslog
```



* Ensure the following entries are in the `/etc/rsyslog.conf`.  Then restart rsyslog. First backup the file.

```
cp /etc/rsyslog.conf{,.old}


$ModLoad imklog # reads kernel messages (the same are read from journald)
$ModLoad imudp
$UDPServerRun 514
$ModLoad imtcp
$InputTCPServerRun 514
$template FILENAME,"/var/log/%fromhost-ip%/syslog.log"
*.* ?FILENAME
```

* Start the rsyslog server.


```
atomic run --name rsyslog docker-registry.usersys.redhat.com/atomcga/rsyslog
```


* Test the configuration.


* On the Atomic master host open a terminal, make sure rsyslog is started with atomic run, and issue command `logger remote test`

* On the rsyslog server, check in the `/var/log/` directory. You should see a directory that has the IP address of the atomic server.  In that directory will be a `syslog.log` file.  Watch that file.

```
tail -f /var/log/192.168.121.228/syslog.log
Feb 10 09:40:01 localhost CROND[6210]: (root) CMD (/usr/lib64/sa/sa1 1 1)
Feb 10 09:40:03 localhost vagrant: remote test
Feb 10 09:40:05 localhost vagrant: remote test
Feb 10 09:40:07 localhost vagrant: remote test
```

* How do I remove it?

Stop the container and remove the image

```
atomic uninstall docker-registry.usersys.redhat.com/atomcga/rsyslog:7.1-2
```

# END SKIP SECTION 2/23/2015 - scollier

###Using rhel-tools

* Install the rhel-tools container

```
atomic install registry.access.stage.redhat.com/rhel7/rhel-tools
Pulling repository registry.access.stage.redhat.com/rhel7/rhel-tools
9a8ad4567c27: Download complete 
Status: Downloaded newer image for registry.access.stage.redhat.com/rhel7/rhel-tools:latest
```

Run the rhel-tools container.  Notice how you are dropped to the prompt inside the container.

```
atomic run registry.access.stage.redhat.com/rhel7/rhel-tools
docker run -it --name rhel-tools --privileged --ipc=host --net=host --pid=host -e HOST=/host -e NAME=rhel-tools -e IMAGE=registry.access.stage.redhat.com/rhel7/rhel-tools -v /run:/run -v /var/log:/var/log -v /etc/localtime:/etc/localtime -v /:/host registry.access.stage.redhat.com/rhel7/rhel-tools
[root@atomic-00 /]#
```

* Explore the environment.  Check processes.

```
# ps aux
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root          1  0.0  0.1  61880  7720 ?        Ss   Feb20   0:03 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
root          2  0.0  0.0      0     0 ?        S    Feb20   0:00 [kthreadd]
root          3  0.0  0.0      0     0 ?        S    Feb20   0:15 [ksoftirqd/0]
root          5  0.0  0.0      0     0 ?        S<   Feb20   0:00 [kworker/0:0H]
root          7  0.0  0.0      0     0 ?        S    Feb20   0:00 [migration/0]
<snip>
```

* Check the envirionment variables.

```
# env
HOSTNAME=atomic-00.localdomain
HOST=/host
TERM=xterm
NAME=rhel-tools
```

* Run a sosreport.  Notice where it is saved to.  The sosreport tool has been modified to work in a container environment.

```
# sosreport 

sosreport (version 3.2)

This command will collect diagnostic and configuration information from
this Red Hat Atomic Host system.

<snip>

Your sosreport has been generated and saved in:
  /host/var/tmp/sosreport-scollier.12344321-20150225144723.tar.xz

The checksum is: 9de2decce230cd4b2b84ab4f41ec926e

Please send this file to your support representative.
```


* Clone a git repo, and save the repo to the host files system, not to the image filesystem.

```
# git clone https://github.com/GoogleCloudPlatform/kubernetes.git /host/tmp/kubernetes
Cloning into '/host/tmp/kubernetes'...
remote: Counting objects: 48730, done.
remote: Compressing objects: 100% (22/22), done.
remote: Total 48730 (delta 7), reused 0 (delta 0), pack-reused 48708
Receiving objects: 100% (48730/48730), 30.44 MiB | 9.63 MiB/s, done.
Resolving deltas: 100% (32104/32104), done.
```

Exit the container and look at the git repo and the sosreport output.  Hit CTRL D to exit the contianer, or type _exit_.

```
# ls {/tmp,/var/tmp/}
/tmp:
ks-script-K46kdd  ks-script-Si6KRr  kubernetes

/var/tmp/:
sosreport-scollier.12344321-20150225144723.tar.xz  sosreport-scollier.12344321-20150225144723.tar.xz.md5
```

###Using sadc

The sadc container is our "system activity data collector", it is the daemon that runs in the background that provides the ongoing performance data that sar parses and presents to you.  This container is meant to run in the background only, it is not an interactive container like rhel-tools.

* Do this on these steps on the master node only.  Install the sadc container.

```
# atomic install registry.access.stage.redhat.com/rhel7/sadc
Pulling repository registry.access.stage.redhat.com/rhel7/sadc
1a97a9cc4d1b: Download complete 
Status: Downloaded newer image for registry.access.stage.redhat.com/rhel7/sadc:latest
docker run --rm --privileged --name sadc -v /:/host -e HOST=/host -e IMAGE=registry.access.stage.redhat.com/rhel7/sadc -e NAME=name registry.access.stage.redhat.com/rhel7/sadc /usr/local/bin/sysstat-install.sh
Installing file at /host//etc/cron.d/sysstat
Installing file at /host//etc/sysconfig/sysstat
Installing file at /host//etc/sysconfig/sysstat.ioconf
Installing file at /host//usr/local/bin/sysstat.sh
```

* check the status of the files.

```
stat /etc/cron.d/sysstat /etc/sysconfig/sysstat /etc/sysconfig/sysstat.ioconf /usr/local/bin/sysstat.sh
  File: ‘/etc/cron.d/sysstat’
  Size: 339         Blocks: 8          IO Block: 4096   regular file
Device: fd00h/64768d    Inode: 12659901    Links: 1
Access: (0600/-rw-------)  Uid: (    0/    root)   Gid: (    0/    root)
Context: system_u:object_r:unlabeled_t:s0
Access: 2015-02-25 01:38:01.277161028 +0000

...<snip>...

Modify: 2015-02-18 09:30:40.000000000 +0000
Access: 2015-02-25 01:36:50.848936926 +0000
Modify: 2015-02-18 09:30:40.000000000 +0000
Change: 2015-02-25 01:37:39.262403129 +0000
 Birth: -

```

* Run the container. Ensure the container is running.

```
# atomic run registry.access.stage.redhat.com/rhel7/sadc
docker run -d --privileged --name sadc -v /etc/sysconfig/sysstat:/etc/sysconfig/sysstat -v /etc/sysconfig/sysstat.ioconf:/etc/sysconfig/sysstat.ioconf -v /var/log/sa:/var/log/sa -v /:/host -e HOST=/host -e IMAGE=registry.access.stage.redhat.com/rhel7/sadc -e NAME=sadc --net=host --restart=always registry.access.stage.redhat.com/rhel7/sadc /usr/local/bin/sysstat.sh
79bf6243c05a9c1a07c7f987ac02b66264ff87ba84cc4714a24a48b3d526ebbc

# docker ps -l
CONTAINER ID        IMAGE                                               COMMAND                CREATED             STATUS              PORTS               NAMES
79bf6243c05a        registry.access.stage.redhat.com/rhel7/sadc:7.1-3   "/usr/local/bin/syss"   33 seconds ago      Up 32 seconds                           sadc              
```

* Check the status of the files in /var/log/.

```
stat /var/log/sa/sa*
  File: ‘/var/log/sa/sa24’
  Size: 656         Blocks: 8          IO Block: 4096   regular file
Device: fd00h/64768d    Inode: 4229027     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Context: system_u:object_r:docker_log_t:s0
Access: 2015-02-25 01:40:07.042784999 +0000
Modify: 2015-02-25 01:40:07.042784999 +0000
Change: 2015-02-25 01:40:07.042784999 +0000
 Birth: -
```

* Run the RHEL Tools container.

```
atomic install registry.access.stage.redhat.com/rhel7/rhel-tools
```

* Once inside the RHEL tools container, run sar and check the output.

```
sar
```


##**Troubleshooting**

### Restarting services

Restart services in this order:

1. etcd
1. flanneld
1. docker

### Networking

Flannel configures an overlay network that docker uses. `ip a` must show docker and flannel on the same network.

Flannel has file `/usr/lib/systemd/system/docker.service.d/flannel.conf` which sources `/run/flannel/docker`, generated from the `flannel-config.json` file. etcd stores the flannel configuration for the Master. Flannel runs on each node host (minion) to setup a unique class-C container network.
