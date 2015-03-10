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

* Make 3 copy-on-write images, using the downloaded image as a "gold" master.

```bash
for i in $(seq 3); do qemu-img create -f qcow2 -o backing_file=rhel-atomic-host-7.qcow2 rhel-atomic-host-7-${i}.qcow2 ; done
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


* Update all of the atomic hosts. The following commands will subscribe you to receive updates and allow you to upgrade your Atomic host.  

**NOTE:** Depending on the version of Atomic that you initially installed, some of the sample output below may differ from what you see.


```
# atomic host status
  TIMESTAMP (UTC)         VERSION     ID             OSNAME               REFSPEC                                                 
* 2015-02-17 22:30:38     7.1.244     27baa6dee2     rhel-atomic-host     rhel-atomic-host:rhel-atomic-host/7/x86_64/standard     

# subscription-manager register --serverurl=[stage] --baseurl=[stage] --username=[account_user] --password=[account_pass] --auto-attach
```

**NOTE:** The below output is an example.  That is what a customer will see once there is a tree update.  What you will see in the lab is that there is "No upgrade Available", this is expected.

```
# atomic host upgrade
Updating from: rhel-atomic-host-ostree:rhel-atomic-host/7/x86_64/standard

53 metadata, 321 content objects fetched; 81938 KiB transferred in 71 seconds
Copying /etc changes: 26 modified, 4 removed, 57 added
Transaction complete; bootconfig swap: yes deployment count change: 1
Changed:
  libgudev1-208-99.atomic.0.el7.x86_64
  libsmbclient-4.1.12-21.el7_1.x86_64
  libwbclient-4.1.12-21.el7_1.x86_64
  python-six-1.3.0-4.el7.noarch
  redhat-release-atomic-host-7.1-20150219.0.atomic.el7.1.x86_64
  samba-common-4.1.12-21.el7_1.x86_64
  samba-libs-4.1.12-21.el7_1.x86_64
  shadow-utils-2:4.1.5.1-18.el7.x86_64
  subscription-manager-1.13.22-1.el7.x86_64
  subscription-manager-plugin-container-1.13.22-1.el7.x86_64
  subscription-manager-plugin-ostree-1.13.22-1.el7.x86_64
  systemd-208-99.atomic.0.el7.x86_64
  systemd-libs-208-99.atomic.0.el7.x86_64
  systemd-sysv-208-99.atomic.0.el7.x86_64
Upgrade prepared for next boot; run "systemctl reboot" to start a reboot
```

* Check the atomic tree version.  The output shows that a new deployment is at the top of the list and that will be the version which will be active after a reboot.

```
# atomic host status
  TIMESTAMP (UTC)         VERSION     ID             OSNAME               REFSPEC                                                        
  2015-02-19 20:26:26     7.1.0       5799825b36     rhel-atomic-host     rhel-atomic-host-ostree:rhel-atomic-host/7/x86_64/standard     
* 2015-02-17 22:30:38     7.1.244     27baa6dee2     rhel-atomic-host     rhel-atomic-host-ostree:rhel-atomic-host/7/x86_64/standard
```

Note the `*` identifies the active version.

* Reboot the VMs to switch to updated tree.

```
# systemctl reboot
```

* After the VMs have rebooted, SSH into each and enter sudo shell:

```
# sudo -i
```

* Check your version with atomic. The `*` pointer should now be on the new tree.

```
# atomic host status
  TIMESTAMP (UTC)         VERSION     ID             OSNAME               REFSPEC                                                        
* 2015-02-19 20:26:26     7.1.0       5799825b36     rhel-atomic-host     rhel-atomic-host-ostree:rhel-atomic-host/7/x86_64/standard     
  2015-02-17 22:30:38     7.1.244     27baa6dee2     rhel-atomic-host     rhel-atomic-host-ostree:rhel-atomic-host/7/x86_64/standard     

```

## Configure docker to use a private registry
Integrating a private registry is an important use case for customers. For this lab we add a private registry to pull and search images.

* Edit the `/etc/sysconfig/docker` file and restart docker. You will need the following lines in the file.

```
ADD_REGISTRY='--add-registry [PRIVATE_REGISTRY]'
```

**NOTE:** If the private registry is not configured with a CA-signed SSL certificate `docker pull ...` will fail with a message about an insecure registry. In that case add the following line to `/etc/sysconfig/docker`:

```
INSECURE_REGISTRY='--insecure-registry [PRIVATE_REGISTRY]'
```

* `/etc/sysconfig/docker` includes example ADD_REGISTRY and INSECURE_REGISTRY lines.  Uncomment them and append the [PRIVATE_REGISTRY] FQDN.  For example:

```
ADD_REGISTRY='--add-registry my.private.registry.fqdn'
INSECURE_REGISTRY='--insecure-registry my.private.registry.fqdn'
```

* Restart docker

```
systemctl restart docker
```

## Explore the environment

What can you do?  What can't you do?  You may see a lot of "Command not Found" messages...  We'll explain how to get around that with the rhel-tools container in a later lab.  Type the following commands.  

```
man tcpdump

git

tcpdump

sosreport
```
Why wouldn't we include these commands in the Atomic image?

# If you want to add something to Atomic Host, you must build a container

Lets try

```
atomic install rhel7/rhel-tools
```

This will install the rhel-tools container, which can be used as the Adminstrators shell.

Now lets try

```
atomic run rhel7/rhel-tools man tcpdump
atomic run rhel7/rhel-tools tcpdump
```

You can also go into the rhel-tools container and explore its contents.

```
atomic run rhel7/rhel-tools /bin/sh
```

You might even want to create a shell script like the following on the Atomic host, as a helper script:

```
cat /usr/local/bin/man
#!/bin/sh
atomic run rhel7/rhel-tools man $@

chmod +x /usr/local/bin/man
```

This script makes using man pages transparent to the user (even though man  pages are not installed on the Atomic host, only in the rhel-tools container).
It could also be done with a bash alias.

```
man tcpdump
```

rhel-tools is an Super Privileged Container, which will be covered in the next presentation and lab.

This concludes the deploying Atomic lab.

## [NEXT LAB](atomicDockerLVM.md)
