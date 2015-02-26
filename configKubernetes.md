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

* Edit `/etc/kubernetes/apiserver` to appear as such.  Make sure to substitute out the MASTER_PRIV_IP_ADDR placeholder below.

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

* Edit `/etc/kubernetes/kubelet` to appear as below.  Make sure you substitute kublet or minion IP addresses appropriately. You have to make two changes below.

```
# The address for the info server to serve on
KUBELET_ADDRESS="--address=0.0.0.0"

# this MUST match what you used in KUBELET_ADDRESSES on the controller manager
# unless you used what hostname -f shows in KUBELET_ADDRESSES.
KUBELET_HOSTNAME="--hostname_override=LOCAL_MINION_ETH0_ADDRESS"

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

* On the minion (minion) -

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
