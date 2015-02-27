##**BEFORE YOU ARRIVE**
    In order to make best use of the lab time please ensure you have:

1. A running Atomic Host

##**Agenda / High Level Overview:**

1. Inspect and understand LVM setup
2. Download images, write inside the container
3. Use bind mounts and notice that space goes to the host pool
4. More documentation

#**Inspect the system**

* Take note of the automatic storage configuration for Docker by
  looking at the logical volumes. An Atomic Host comes optimized out
  of the box to take advantage of LVM thinpool storage instead of
  the loopback used with Docker by default.

```
# lvs
  LV          VG       Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  docker-pool atomicos twi-a-tz-- 6.69g             0.16   0.20  
  root        atomicos -wi-ao----  2.94g                                                    
```

* Now see how that corresponds to the Docker image and container storage:

```
# docker info
Containers: 0
Images: 0
Storage Driver: devicemapper
 Pool Name: atomicos-docker--pool
 Pool Blocksize: 65.54 kB
 Backing Filesystem: <unknown>
 Data file:
 Metadata file:
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

#**Download images**
* Pull a Docker image and notice that the data goes into the pool:

```
# docker pull registry.access.redhat.com/rhel7
# docker info |grep 'Data Space Used'
 Data Space Used: 177 MB
```

* Create a new container, writing 50MB of data *inside* the container.  Note the container persists.

```
# docker run registry.access.redhat.com/rhel7 dd if=/dev/zero of=/var/tmp/data count=100000
100000+0 records in
100000+0 records out
51200000 bytes (51 MB) copied, 0.0789428 s, 649 MB/s
# docker ps -a 
CONTAINER ID        IMAGE                                COMMAND                CREATED             STATUS                      PORTS               NAMES
6d645085a215        registry.access.redhat.com/rhel7:0   "dd if=/dev/zero of=   5 seconds ago      Exited (0) 4 seconds ago                       prickly_stallman  
# docker info |grep 'Data Space Used'
  Data Space Used: 234.2 MB
```

Now, remove the stopped container and notice that the space is freed in Docker storage:
```
# docker rm prickly_stallman
# docker info |grep 'Data Space Used'
 Data Space Used: 200.3 MB
```

#**Use bind mounts**
* Create host directory, label it, use a bind mount to write 50MB of data *outside* the container

```
# mkdir -p /var/local/containerdata
# chcon -R -h -t svirt_sandbox_file_t /var/local/containerdata/
# docker run --rm registry.access.redhat.com/rhel7 -v /var/local/containerdata:/var/tmp dd if=/dev/zero of=/var/tmp/data count=100000
```

Notice that we used `--rm`, so the container is automatically deleted
after it runs.  However, the data exists on the host filesystem:

```
# ls -al /var/local/containerdata
```

And the space in `df -h` on the host will have increased.

#**More information**

In-progress Atomic storage guide:

http://jenkinscat.gsslab.pnq.redhat.com:8080/job/atomic-branch-yruseva-atomic-storage/lastSuccessfulBuild/artifact/pub_Atomic_Storage/index.html
