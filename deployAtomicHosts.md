##**BEFORE YOU ARRIVE**
    In order to make best use of the lab time please review the deployment options and ensure either 

1. A working KVM environment or
2. Access to an OpenStack environment.

##**Agenda / High Level Overview:**

1. Deploy Atomic Hosts
2. Configure Flannel
3. Configure K8s
4. Deploy an Application
5. Use SPCs on the Atomic Hosts


#**Deployment**
There are many ways to deploy an Atomic host.  In this lab, we provide guidance for OpenStack or local KVM.

##**Deployment Option 1: Atomic Hosts on OpenStack**
You may use an OpenStack service.

1. Navigate to Instances
1. Click "Launch Instances"
1 Complete form
  1. Details tab
    * Instance name: arbitrary name. Note the UUID of the image will be appended to the instance name. You may want to use your name in the image so you can easily find it.
    * Flavor: *m1.medium*
    * Instance count: *3*
    * Instance Boot Source: *Boot from image*
      * Image name: *[atomic_image]*
  1. Access & Security tab
    * select you keypair that was uploaded during OpenStack account setup.
    * Security Groups: *Default*
1. Click "Launch"

Three VMs will be created. Once Power State is *Running* you may SSH into the VMs. Your SSH public key will be used.

* Note: Each instance requires a floating IP address in addition to the private OpenStack `172.x.x.x` address. Your OpenStack tenant may automatically assign a floating IP address. If not, you may need to assign it manually. If no floating IP addresses are available, create them.
  1. Navigate to Access & Security
  1. Click "Floating IPs" tab
  1. Click "Allocate IPs to project"
  1. Assign floating IP addresses to each VM instance
* SSH into the VMs with user `cloud-user` and the instance floating IP address. This address will probably be in the `10.3.xx.xx` range.

```
ssh cloud-user@10.3.xx.xxx
```


##**Deployment Option 2: Atomic Hosts on KVM**

* Grab and extract the Atomic and metadata images from our internal repo.  Use sudo and appropriate permissions.

```
wget [metadata ISO image]
cp atomic0-cidata.iso /var/lib/libvirt/images/.
wget [atomic QCOW2 image]
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
virt-install --import --name atomic-ga-1 --ram 1024 --vcpus 2 --disk path=/var/lib/libvirt/images/rhel-atomic-host-7-1.qcow2,format=qcow2,bus=virtio --disk path=/var/lib/libvirt/images/atomic0-cidata.iso,device=cdrom --network bridge=br0 --force

virt-install --import --name atomic-ga-2 --ram 1024 --vcpus 2 --disk path=/var/lib/libvirt/images/rhel-atomic-host-7-2.qcow2,format=qcow2,bus=virtio --disk path=/var/lib/libvirt/images/atomic0-cidata.iso,device=cdrom --network bridge=br0 --force

virt-install --import --name atomic-ga-3 --ram 1024 --vcpus 2 --disk path=/var/lib/libvirt/images/rhel-atomic-host-7-3.qcow2,format=qcow2,bus=virtio --disk path=/var/lib/libvirt/images/atomic0-cidata.iso,device=cdrom --network bridge=br0 --force
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
subscription-manager register --serverurl=[stage] --baseurl=[stage] --username=[account_user] --password=[account_pass] --auto-attach
atomic host upgrade
```
This will subscribe the system to the stage environment and upgrade the system to the latest tree in the stage environment.

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

* Take note of the automatic storage configuration for Docker by looking at the logical volumes. An Atomic host comes optimized out of the box to take advantage of Device Mapper direct-lvm storage instead of straight device mapper.

```
# lvs
  LV          VG       Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  docker-data atomicos -wi-ao---- 16.72g                                                    
  docker-meta atomicos -wi-ao---- 32.00m                                                    
  root        atomicos -wi-ao----  2.94g                                                    
```

* Now see how that cooresponds to the Docker data and meta storage.

```
# docker info
Containers: 0
Images: 0
Storage Driver: devicemapper
 Pool Name: docker-253:0-12590043-pool
 Pool Blocksize: 65.54 kB
 Backing Filesystem: <unknown>
 Data file: /dev/atomicos/docker-data
 Metadata file: /dev/atomicos/docker-meta
 Data Space Used: 11.8 MB
 Data Space Total: 17.95 GB
 Metadata Space Used: 106.5 kB
 Metadata Space Total: 33.55 MB
 Udev Sync Supported: true
 Library Version: 1.02.93-RHEL7 (2015-01-28)
Execution Driver: native-0.2
Kernel Version: 3.10.0-229.el7.x86_64
Operating System: Employee SKU
CPUs: 2
Total Memory: 3.86 GiB
Name: scollier-atomic-ga-kube-test-acaea32f-667a-4a54-aea3-41d1ac573c1
ID: CNPB:PLKF:34V3:4ESX:Y3KG:XCUV:RYSQ:ZMHN:TFXF:2ENH:AR3V:MO5Q
```

* Explore the environment.  What can you do?  What can't you do?  You may see a lot of "Command not Found" messages...  We'll explain how to get around that with the rhel-tools container in a later lab.  Type the following commands.  

```
man tcpdump

git

tcpdump

sosreport
```
Why wouldn't we include these commands in the Atomic image?

This concludes the deploying Atomic lab.
