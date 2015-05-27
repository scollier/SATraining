##Configure Kubernetes##

The kubernetes package provides several services

* kube-apiserver
* kube-scheduler
* kube-controller-manager
* kubelet, kube-proxy

These services are managed by systemd and the configuration resides in a central location, `/etc/kubernetes`. We will break the services up between the hosts.  The first host, *master*, will be the kubernetes master.  This host will run kube-apiserver, kube-controller-manager, and kube-scheduler. In addition, the master will also run _etcd_. The remaining hosts, the *minions* or *nodes*, will run kubelet, proxy, cadvisor and docker.

###Prepare the hosts

* Backup the kubernetes configuration files on each system (master and nodes) before continuing.

```bash
for i in $(ls /etc/kubernetes/*); do cp $i{,.orig}; echo "Making a backup of $i"; done
```

**NOTE:** It is important that you replace any exsiting content in the config files below with the new content provided.  You will find that the existing content in the config files may not match what is provided below.  This is expected.

* Edit `/etc/kubernetes/config` to be the same on **all hosts**. For OpenStack VMs we will be using the *private IP address* of the master host.  Make sure to substitute out the MASTER_PRIV_IP_ADDR placeholder below.

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

* Edit `/etc/kubernetes/apiserver` to appear as such.  Make sure to substitute out the MASTER_PRIV_IP_ADDR placeholder below.  The portal_net IP addresses need to be an IP address range not used anywhere else.  They do not need to be routed.  They do not need to be assigned to anything.  It just must be an unused block of addresses.  Kubernetes will assign "services" one of these addresses.  But traffic to (or from) these addresses will NEVER leave a node.  It's actually the proxy on the local node that reponds to these addresses.  This must be a different unused range than that assigned to flannel.  Flannel specifies the addresses used by pods.  portal_net specifies the addresses used by services.  But in both cases, no infrastructure changes are needed.  Just pict an unused block of addresses.

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

* Edit `/etc/kubernetes/controller-manager` to appear as such.  Substitute your minion IPs here in place of the MINION_PRIV_IP_{1,2} placeholder.

```
# Comma separated list of minions
KUBELET_ADDRESSES="--machines=MINION_PRIV_IP_1,MINION_PRIV_IP_2"
```

* Start the appropriate services on master:

```bash
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do 
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES 
done
```

####Configure the kubernetes services on the minions

**NOTE:** Make these changes on each minion.

***We need to configure and start the kubelet and proxy***

**UGLY** Due to a bug in kubernetes we must configure an empty JSON authorization file on each minion.
* Create the JSON file by running the following on all minions

```bash
echo "{}" > /var/lib/kubelet/auth
```

* Edit `/etc/kubernetes/kubelet` to appear as below.  Make sure you substitute kublet or minion IP addresses appropriately. You have to make two changes below.

```
# The address for the info server to serve on
KUBELET_ADDRESS="--address=0.0.0.0"

# this MUST match what you used in KUBELET_ADDRESSES on the controller manager
# unless you used what hostname -f shows in KUBELET_ADDRESSES.
KUBELET_HOSTNAME="--hostname_override=LOCAL_MINION_ETH0_ADDRESS"

KUBELET_API_SERVER="--api_servers=http://MASTER_PRIV_IP_ADDR:8080"

# Add your own!
KUBELET_ARGS="--auth_path=/var/lib/kubelet/auth"
```

* edit `/etc/kubernetes/proxy` to appear as below.

```
# How the proxy find the apiserver
KUBE_PROXY_ARGS="--master=http://MASTER_PRIV_IP_ADDR:8080"
```

* Start the appropriate services on the minions.

```bash
for SERVICES in kube-proxy kubelet docker; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```

*You should be finished!*

* Check to make sure the cluster can see the minions from the master.

```
$ kubectl get nodes
NAME           LABELS        STATUS
192.168.121.147   Schedulable   <none>    Ready
192.168.121.101   Schedulable   <none>    Ready
```

**The cluster should be running! Launch a test pod.**

##Deploy an application##


* Create a file on master named `apache.json` that looks as such:

```json
{
    "apiVersion": "v1beta1",
    "desiredState": {
        "manifest": {
            "containers": [
                {
                    "image": "fedora/apache",
                    "name": "my-fedora-apache",
                    "ports": [
                        {
                            "containerPort": 80,
                            "protocol": "TCP"
                        }
                    ]
                }
            ],
            "id": "apache",
            "restartPolicy": {
                "always": {}
            },
            "version": "v1beta1",
            "volumes": null
        }
    },
    "id": "apache",
    "kind": "Pod",
    "labels": {
        "name": "apache"
    },
    "namespace": "default"
}
```

This JSON file is describing the attributes of the application environment. For example, it is giving it a "kind", "id", "name", "ports", and "image". Since the fedora/apache images doesn't exist in our environment yet, it will be pulled down automatically as part of the deployment process.

For more information about which options can go in the schema, check out the docs on the [kubernetes github page](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/docs).

* Deploy the fedora/apache image via the `apache.json` file.

```
kubectl create -f apache.json
```


* This command exits immediately, returning the value of the label, `apache`. You can monitor progress of the operations with these commands:
On the master (master) -

```
journalctl -f -l -xn -u kube-apiserver -u kube-scheduler
```

* On the minion (minion) -

```
journalctl -f -l -xn -u kubelet -u kube-proxy -u docker
```


* After the pod is deployed, you can also list the pod.  I have a few pods running here.

```
# kubectl get pods
POD                 IP                  CONTAINER(S)        IMAGE(S)            HOST                LABELS              STATUS
apache              18.0.53.3           my-fedora-apache    fedora/apache       192.168.121.147/    name=apache         Running
mysql               18.0.73.2           mysql               mysql               192.168.121.101/    name=mysql          Running
redis-master        18.0.53.2           master              dockerfile/redis    192.168.121.147/    name=redis-master   Running
```

The state might be 'Pending'. This indicates that docker is still attempting to download and launch the container.

* You can get even more information about the pod like this.

```
kubectl get pods --output=json apache
```

* Finally, on the minion (minion), check that the pod is available and running.

```
docker images
REPOSITORY TAG IMAGE ID CREATED VIRTUAL SIZE
kubernetes/pause latest 6c4579af347b 7 weeks ago 239.8 kB
fedora/apache latest 6927a389deb6 3 months ago 450.6 MB

docker ps -l
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
05c69c00ea48 fedora/apache:latest "/run-apache.sh" 2 minutes ago Up 2 minutes k8s--master.3f918229--apache.etcd--8cd6efe6_-_3a95_-_11e4_-_b618_-_5254005318cb--9bb78458
```

## Create a service to make the pod discoverable ##

Now that the pod is known to be running we need a way to find it.  Pods in kubernetes may launch on any minion and get an IP addresses from flannel.  So finding them is obviously not easy.  You don't want people to have to look up what minion the web server is on before they can find your web page!  Kubernetes solves this with a "service"  By default kubernetes will create an internal IP address for the service (from the portal_net range) which pods can use to find the service.  But we want the web server to be available outside the cluster.  So we need to tell kubernetes how traffic will arrive into the cluster destined for this webserver.  To do so we define a list of "publicIPs".  These need to be actual IP addresses assigned to actual minions.  In configurations like AWS or OpenStack where machines have both a public IP assigned somewhere "in the cloud" and the private IP assigned to the node, you must use the private IP.  This IP must be assigned to a minion and be visable on the minion via "ip addr." This is a list, you may list multiple nodes public IP.

* Create a service on the master by creating a `service.json` file

**NOTE:** You must use an actual IP address for the `publicIPs` value or the service will not run correctly on the minions

```json
{
    "apiVersion": "v1beta1",
    "containerPort": 80,
    "id": "frontend",
    "kind": "Service",
    "labels": {
        "name": "frontend"
    },
    "port": 80,
    "publicIPs": [
        "MINION_PRIV_IP_1"
    ],
    "selector": {
        "name": "apache"
    }
}
```

* Load the JSON file into the kubenetes system

```bash
kubectl create -f service.json
```

* Check that the service is loaded on the master

```bash
# kubectl get services
NAME                LABELS                                    SELECTOR            IP                  PORT
kubernetes-ro       component=apiserver,provider=kubernetes   <none>              10.254.207.162      80
frontend            name=frontend                             name=apache         10.254.195.231      80
kubernetes          component=apiserver,provider=kubernetes   <none>              10.254.8.30         443
```

* Check out how it by looking at the following commands on any minion

```bash
iptables -nvL -t nat
journalctl -b -l -u kube-proxy
```

* Finally, test that the container is actually working.

```
curl http://MINION_PRIV_IP_1/
Apache
```

* Now really test it.  If you are using OS1 you should be able to hit the web server from you web browser by going to the PUBLIC IP associated with the minion(s) you chose in your service.

```
firefox http://MINION_PUBLIC_IP_1/
```

* To delete the container.

```
kubectl delete pod apache
```

## Create a replication controller to control the pod ##

This should have the exact same definition of the pod as above, only now it is being controlled by a replication controller.  So if you delete the pod, or if the node disappears, the pod will be restarted elsewhere in the cluster!

* Create an `rc.json` file to describe the replication controller

```json
{
    "apiVersion": "v1beta1",
    "desiredState": {
        "podTemplate": {
            "desiredState": {
                "manifest": {
                    "containers": [
                        {
                            "image": "fedora/apache",
                            "name": "my-fedora-apache",
                            "ports": [
                                {
                                    "containerPort": 80,
                                    "protocol": "TCP"
                                }
                            ]
                        }
                    ],
                    "id": "apache",
                    "restartPolicy": {
                        "always": {}
                    },
                    "version": "v1beta1",
                    "volumes": null
                }
            },
            "labels": {
                "name": "apache"
            }
        },
        "replicaSelector": {
            "name": "apache"
        },
        "replicas": 1
    },
    "id": "apache-controller",
    "kind": "ReplicationController",
    "labels": {
        "name": "apache"
    }
}
```

* Load the JSON file on the master

```bash
kubectl create -f rc.json
```
* Check that the replication controller has started

```bash
# kubectl get rc
CONTROLLER          CONTAINER(S)        IMAGE(S)            SELECTOR            REPLICAS
apache-controller   my-fedora-apache    fedora/apache       name=apache         1
```

* The replication controller should have spawned a pod on a minion.  (This make take a short while, so STATUS may be Unknown at first)

```bash
# kubectl get pods
POD                                    IP                  CONTAINER(S)        IMAGE(S)            HOST                LABELS              STATUS
52228aef-be99-11e4-91e5-52540052bd24   18.0.79.4           my-fedora-apache    fedora/apache       kube-minion1/       name=apache         Running
```

Feel free to resize the replication controller and run multiple copies of apache.  Note that the kubernetes `publicIP` balances between ALL of the replicas!

```bash
# kubectl resize --replicas=3 replicationController apache-controller
resized

# kubectl get rc
CONTROLLER          CONTAINER(S)        IMAGE(S)            SELECTOR            REPLICAS
apache-controller   my-fedora-apache    fedora/apache       name=apache         3

# kubectl get pods
POD                                    IP                  CONTAINER(S)        IMAGE(S)            HOST                LABELS              STATUS
ac23ccfa-be99-11e4-91e5-52540052bd24   18.0.98.3           my-fedora-apache    fedora/apache       kube-minion2/       name=apache         Running
52228aef-be99-11e4-91e5-52540052bd24   18.0.79.4           my-fedora-apache    fedora/apache       kube-minion1/       name=apache         Running
ac22a801-be99-11e4-91e5-52540052bd24   18.0.98.2           my-fedora-apache    fedora/apache       kube-minion2/       name=apache         Running
```

I suggest you resize to 0 before you delete the replication controller.  Deleting a `replicationController` will leave the pods running.

Of course this just scratches the surface. I recommend you head off to the kubernetes github page and follow the [guestbook example](https://github.com/GoogleCloudPlatform/kubernetes/tree/754a2a8305c812121c3845d8293efdd819b6a704/examples/guestbook-go). It is a bit more complicated but should expose you to more functionality.
