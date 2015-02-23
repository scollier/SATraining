Demo: March 3rd-4th 2015
Location: Westford

Pre-Requisites: Functioning libvirt backed Vagrant on Fedora


##**Agenda / High Level Overview:**

1. Deploy Atomic Hosts
2. Configure Flannel
3. Configure K8s
4. Deploy an Application
5. Use SPCs on the Atomic Hosts


#**Deployment**
There are many ways to deploy an Atomic host.  In this lab, we provide guidance for KVM.


##**Deploy Atomic Hosts on KVM**

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

* Confirm you can login to the hosts with:

Username: cloud-user
Password: atomic

Then: 

```
sudo -i
```

* Update all of the atomic hosts. The following commands will change you to the GA.staging tree, which should be what customers will see.

```
ostree remote add --set=gpg-verify=false GA.brew http://download.eng.bos.redhat.com/rel-eng/Atomic/7/trees/GA.brew/repo/
rpm-ostree rebase GA.brew:rhel-atomic-host/7/x86_64/standard
systemctl reboot 
```

* Check your version with atomic.

```
atomic host status
```




#**Configure Flannel**

* Open up 3 terminals and ssh into the Atomic hosts.

* Check the versions of software you have.  This should be the same on all nodes.

```
rpm -qa | egrep "etc|docker|flannel|kube"
```

On the master node (pick one):

* Look at networking before flannel configuration.
       
```
ip a
```

* Start etcd on the master node.

       
```
systemctl start etcd; systemctl status etcd
```


* Configure Flannel by creating a flannel-config.json in your current directory.  The contents should be:

       
```
{
"Network": "10.0.0.0/16",
"SubnetLen": 24,
"Backend": {
"Type": "vxlan",
"VNI": 1
     }
}

```

* Add the configuration to the etcd server.  Substitute out the IP address.

       
```
curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/config -XPUT --data-urlencode value@flannel-config.json
```


* Verify the key exists.

       
```
curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/config
```

* Backup the flannel configuration file.

       
```
cp /etc/sysconfig/flanneld{,.orig}
```

* Configure flannel, use your interface on your system.  Mine is eth0, yours might be ens3.

       
```
sed -i 's/#FLANNEL_OPTIONS=""/FLANNEL_OPTIONS="eth0"/g' /etc/sysconfig/flanneld
```


* The /etc/sysconfig/flanneld should look like this (sub your IP for the FLANNEL_ETCD key).

       
```
grep -v ^\# /etc/sysconfig/flanneld

FLANNEL_ETCD="http://192.168.121.105:4001"
FLANNEL_ETCD_KEY="/coreos.com/network"
FLANNEL_OPTIONS="eth0"
```


* Start up the flanneld service.

       
```
systemctl restart flanneld
systemctl status flanneld
```

* Check the interfaces on the host now. Notice there is now a flannel.1 interface.

       
```
ip a
```

Now that master is configured, lets configure the minions (minion{1,2}).

From the minions:

* Use curl to check firewall settings from the minion to the master.  We need to ensure connectivity to the etcd service.  You may want to set up your /etc/hosts file for name resolution here.  If there are any issues, just fall back to IP addresses for now.

       
```
curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/config
```

For some of the steps below, it might help to set up ssh keys on the master and copy those over to the minions, e.g. with ssh-copy-id.  You also might want to set hostnames on the minions and edit your /etc/hosts files on all nodes to reflect that.

From the master:

* Copy over flannel configuration to the minions, both of them.

       
```
scp /etc/sysconfig/flanneld x.x.x.x:/etc/sysconfig/.
```


* From master, restart services on both of the minions.
       
```
ssh root@x.x.x.x systemctl restart flanneld
ssh root@x.x.x.x systemctl enable flanneld
```

* From master, check the new interface on both of the minions.
       
```
ssh root@x.x.x.x ip a l flannel.1
```

From any node in the cluster, check the cluster members by issuing a query to etcd via curl.  You should see that three servers have consumed subnets.  You can associate those subnets to each server by the MAC address that is listed in the output.

       
```
curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/subnets | python -mjson.tool
```


* From all nodes, review the /run/flannel/subnet.env file.  This file was generated automatically by flannel.

       
```
cat /run/flannel/subnet.env
```

* Restart docker on the minions and make sure it is running properly.
       
```
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
systemctl status docker
```

* Check the network on the minion. If Docker fails to load, or the flannel IP is not set correctly, reboot the system. A functioning configuration should look like the following; notice the docker0 and flannel.1 interfaces.

       
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


##Configure Kubernetes##

The kubernetes package provides a few services: kube-apiserver, kube-scheduler, kube-controller-manager, kubelet, kube-proxy.  These services are managed by systemd and the configuration resides in a central location: /etc/kubernetes. We will break the services up between the hosts.  The first host, master, will be the kubernetes master.  This host will run the kube-apiserver, kube-controller-manager, and kube-scheduler.  In addition, the master will also run _etcd_.  The remaining hosts, the minions will run kubelet, proxy, cadvisor and docker.

**Prepare the hosts:**

* Edit /etc/kubernetes/config which will be the same on all hosts to contain:

```
# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd_servers=http://master:4001"

# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow_privileged=false"
```

**Configure the kubernetes services on the master.**

* Edit /etc/kubernetes/apiserver to appear as such:

```       
# The address on the local server to listen to.
KUBE_API_ADDRESS="--address=0.0.0.0"

# The port on the local server to listen on.
KUBE_API_PORT="--port=8080"

# How the replication controller and scheduler find the kube-apiserver
KUBE_MASTER="--master=http://master:8080"

# Port minions listen on
KUBELET_PORT="--kubelet_port=10250"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--portal_net=10.254.0.0/16"

# Add you own!
KUBE_API_ARGS=""
```

* Edit /etc/kubernetes/controller-manager to appear as such.  Substitute your minion IPs here.:

```
# Comma separated list of minions
KUBELET_ADDRESSES="--machines=MINION_IP_1,MINION_IP_2"
```

* Start the appropriate services on master:

```
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do 
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES 
done
```

**Configure the kubernetes services on the minions.  Make changes on each minion.**

***We need to configure the kubelet and start the kubelet and proxy***

* Edit /etc/kubernetes/kubelet to appear as below.  Make sure you substitute you kubelet / minion IP addresses appropriately.

```       
# The address for the info server to serve on
KUBELET_ADDRESS="--address=x.x.x.x"

# The port for the info server to serve on
KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname_override=x.x.x.x"

# Add your own!
KUBELET_ARGS=""
```       

* Start the appropriate services on minion (minion).

```
for SERVICES in kube-proxy kubelet docker; do 
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES 
done
```

*You should be finished!*

* Check to make sure the cluster can see the minion (on master)

```
# kubectl get minions
NAME                LABELS              STATUS
192.168.121.147     <none>              Ready
192.168.121.101     <none>              Ready
```

**The cluster should be running! Launch a test pod.**

##Deploy an application##


* Create a file on master called apache.json that looks as such:

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

For more information about which options can go in the schema, check out the docs on the kubernetes github page.

* Deploy the fedora/apache image via the apache.json file.

```
kubectl create -f apache.json
```


* You can monitor progress of the operations with these commands:
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

Of course this just scratches the surface. I recommend you head off to the kubernetes github page and follow the guestbook example. It is a bit more complicated but should expose you to more functionality.

## Use SPCs on the Atomic Hosts

The goal here is to explore some of the images that we will be distributing when Atomic GAs.  We are trying to keep the Atomic image as small as possible where it makes sense.  This means that anything else that gets added to the Atomic host will have to be inside a container.  The examples we will go over in this section are rsyslog, sadc and rhel-tools.  For this to work you need at least two functioning Atomic hosts.

###Using rsyslog

The rsyslog container runs in the background for the purposes of managing logs. We will cover two scenarios:

1. Quick smoke test to make sure logging is working on the localhost.
2. Remote logging.  We will send some logs over the network.


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



How do I use it (scenario 2: remote logging)?

Configure the client by pointing it to the rsyslog server in the /etc/rsyslog.conf

```
*.* @@192.168.121.249:514
```


* Configure the rsyslog server; ensure the following entries are in the /etc/rsyslog.conf.  Then restart rsyslog.


```
$ModLoad imklog # reads kernel messages (the same are read from journald)
$ModLoad imudp
$UDPServerRun 514
$ModLoad imtcp
$InputTCPServerRun 514
$template FILENAME,"/var/log/%fromhost-ip%/syslog.log"
*.* ?FILENAME
```

* Test the configuration.

* On the Atomic host open a terminal and issue: “logger remote test”

* On the rsyslog server, check in the /var/log/ directory. You should see a directory that has the IP address of the atomic server.  In that directory will be a syslog.log file.  Watch that file.

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
atomic uninstall docker-registry.usersys.redhat.com/sct_/rsyslog:latest
```


###Using sadc

The sadc container is our "system activity data collector", it is the daemon that runs in the background that provides the ongoing performance data that sar parses and presents to you.  This container is meant to run in the background only, it is not an interactive container like rhel-tools.




###Using rhel-tools

