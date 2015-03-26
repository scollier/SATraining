## Using Super Privileged Containers on Atomic Host

The goal here is to explore some of the images that we will be distributing for Atomic. We are trying to keep the Atomic image as small as possible where it makes sense. This means that anything else that gets added to the Atomic Host will have to be inside a container. The examples we will go over in this section are rhel-tools, rsyslog, and sadc. For this to work, you need at least two functioning Atomic Hosts (which will be referred to as node 1 and node 2).

###Using rhel-tools

As we saw in the previous labs, the rhel-tools container provides the core system administrator and core developer tools to execute tasks on Red Hat Enterprise Linux 7 Atomic Host. The tools container leverages the atomic command for installation, activation and management.

* Install the rhel-tools container. You can do this on whichever system you want to be node 1.

```
# atomic install [REGISTRY]/rhel7/rhel-tools
Pulling repository [REGISTRY]/rhel7/rhel-tools
9a8ad4567c27: Download complete 
Status: Downloaded newer image for [REGISTRY]/rhel7/rhel-tools:latest
```

Run the rhel-tools container. Notice how you are dropped to the prompt inside the container.

```
# atomic run [REGISTRY]/rhel7/rhel-tools
docker run -it --name rhel-tools --privileged --ipc=host --net=host --pid=host -e HOST=/host -e NAME=rhel-tools -e IMAGE=[REGISTRY]/rhel7/rhel-tools -v /run:/run -v /var/log:/var/log -v /etc/localtime:/etc/localtime -v /:/host [REGISTRY]/rhel7/rhel-tools
[root@atomic-00 /]#
```

* Remember those commands at the end of the Atomic deployment lab? The ones that did not work? Try them again.

```
man tcpdump

git

tcpdump

sosreport
```

* Explore the environment. Check processes:

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

* Check the environment variables:

```
# env
HOSTNAME=atomic-00.localdomain
HOST=/host
TERM=xterm
NAME=rhel-tools
```

* Run an sosreport. Notice where it is saved to, the sosreport tool has been modified to work in a container environment.

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

Notice that the host os is mounted into /host within the container.

```
chroot /host
```

You are back in the host.

```
^d
```

You are back in the container. You may also notice that /run is from the host
and you see the host's network and processes, but you are still in a container.

Let's play around a little more.

* Clone a git repo, and save the repo to the host filesystem, not to the image filesystem.

```
# git clone https://github.com/GoogleCloudPlatform/kubernetes.git /host/tmp/kubernetes
Cloning into '/host/tmp/kubernetes'...
remote: Counting objects: 48730, done.
remote: Compressing objects: 100% (22/22), done.
remote: Total 48730 (delta 7), reused 0 (delta 0), pack-reused 48708
Receiving objects: 100% (48730/48730), 30.44 MiB | 9.63 MiB/s, done.
Resolving deltas: 100% (32104/32104), done.
```

Exit the container and look at the git repo and the sosreport output. Hit CTRL-D to exit the container, or type _exit_.

```
# ls {/tmp,/var/tmp/}
/tmp:
ks-script-K46kdd  ks-script-Si6KRr  kubernetes

/var/tmp/:
sosreport-scollier.12344321-20150225144723.tar.xz  sosreport-scollier.12344321-20150225144723.tar.xz.md5
```

```
less /var/log/messages
```
Notice on a RHEL Atomic Host there is no syslog by default. You can look at the log messages using journald.

```
journalctl 
```

If you want to use rsyslog on your host, you need to install the rsyslog SPC container.

###Using rsyslog

The rsyslog container runs in the background for the purposes of managing logs. We will cover two scenarios:

1. Quick smoke test to make sure logging is working on the localhost.
2. Remote logging, where we will send some logs over the network to another system.

####Scenario 1: Quick Smoketest

* Check the environment before starting, you may have a few residual images. You should not have any rsyslog images. Perform this on node 1:

```
# docker images
REPOSITORY                                               TAG                 IMAGE ID            CREATED             VIRTUAL SIZE

# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

* Install the container:

```
# atomic install rhel7/rsyslog
Pulling repository [PRIVATE_REGISTRY]/rhel7/rsyslog
b5168acccb4c: Download complete 
Status: Downloaded newer image for [PRIVATE_REGISTRY]/rhel7/rsyslog:latest
docker run --rm --privileged -v /:/host -e HOST=/host -e IMAGE=[PRIVATE_REGISTRY]/rhel7/rsyslog -e NAME=rsyslog [PRIVATE_REGISTRY]/rhel7/rsyslog /bin/install.sh
Creating directory at /host//etc/pki/rsyslog
Installing file at /host//etc/rsyslog.conf
Installing file at /host//etc/sysconfig/rsyslog
```

* Check the environment after the install. You should now see the rsyslog image, but no container yet.

```
# docker images
REPOSITORY                                               TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
[PRIVATE_REGISTRY]/rhel7/rsyslog                         7.1-3               b5168acccb4c        2 weeks ago         183.7 MB
[PRIVATE_REGISTRY]/rhel7/rsyslog                         latest              b5168acccb4c        2 weeks ago         183.7 MB

# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

* Run the container:

```
# atomic run --name rsyslog rhel7/rsyslog
docker run -d --privileged --name rsyslog --net=host -v /etc/pki/rsyslog:/etc/pki/rsyslog -v /etc/rsyslog.conf:/etc/rsyslog.conf -v /etc/rsyslog.d:/etc/rsyslog.d -v /var/log:/var/log -v /var/lib/rsyslog:/var/lib/rsyslog -v /run/log:/run/log -v /etc/machine-id:/etc/machine-id -v /etc/localtime:/etc/localtime -e IMAGE=[PRIVATE_REGISTRY]/rhel7/rsyslog -e NAME=rsyslog --restart=always [PRIVATE_REGISTRY]/rhel7/rsyslog /bin/rsyslog.sh
21f8ce9ce852157f048614d9ee8d2c111079ce793f6b4ad80972a50f293977bf
```

* Check the environment, you should now see a running rsyslog container:

```
# docker ps -a
CONTAINER ID        IMAGE                            COMMAND             CREATED             STATUS              PORTS               NAMES
55ff84fcc332        [PRIVATE_REGISTRY]/rhel7/rsyslog:7.1-3   "/bin/rsyslog.sh"   2 minutes ago       Up 2 minutes                            rsyslog             
```

* How do I use it (scenario 1: single host smoke test)? In a terminal on node 1, watch the logs.

```
tail -f /var/log/messages
```

* In another terminal, still on node 1, generate a log.

```
logger test
```

* Back in the first terminal, you should see an entry with “test”

```
Feb  9 16:31:36 localhost vagrant: test
```

####Scenario 2: Remote Logging

Stop the rsyslog container on node 1. We are going to make a change to the `/etc/rsyslog.conf` file and we will need to re-read that. Use the following steps to stop the container. After the container is stopped, you can change the file and restart the container.

```
docker ps
docker stop <container id>
docker ps -a
docker rm <container id>
```

On node 1, edit the `/etc/rsyslog.conf` file and point it to the rsyslog server. This lab will use your second node, node 2, as the rsyslog server. Point the config file at the IP address of node 2, at the bottom of the file:

```
*.* @@x.x.x.x:514
```

* Start the rsyslog container on node 1.

```
# atomic run --name rsyslog rhel7/rsyslog
docker run -d --privileged --name rsyslog --net=host -v /etc/pki/rsyslog:/etc/pki/rsyslog -v /etc/rsyslog.conf:/etc/rsyslog.conf -v /etc/rsyslog.d:/etc/rsyslog.d -v /var/log:/var/log -v /var/lib/rsyslog:/var/lib/rsyslog -v /run/log:/run/log -v /etc/machine-id:/etc/machine-id -v /etc/localtime:/etc/localtime -e IMAGE=rhel7/rsyslog -e NAME=rsyslog --restart=always rhel7/rsyslog /bin/rsyslog.sh
869edb432c7d172dac0317ac24a3763aa19321461415aaab74e3ac48c58e5bb5

# docker ps -l
CONTAINER ID        IMAGE                                                  COMMAND             CREATED             STATUS              PORTS               NAMES
869edb432c7d        [PRIVATE_REGISTRY]/rhel7/rsyslog:7.1-3                         "/bin/rsyslog.sh"   30 seconds ago      Up 29 seconds                           rsyslog
```

* Now that the container is running, switch to the rsyslog server (node 2) and configure it. Install rsyslog on node 2:

```
# atomic install rhel7/rsyslog
```

* Ensure the following entries are in the `/etc/rsyslog.conf` and then restart rsyslog. Ensure you backup the file first:

```
# cp /etc/rsyslog.conf{,.old}
```

Then, append the `/etc/rsyslog.conf` file with these lines. While some simply need to be uncommented, you will need to manually add a few lines as well:

```
$ModLoad imklog # reads kernel messages (the same are read from journald)
$ModLoad imudp
$UDPServerRun 514
$ModLoad imtcp
$InputTCPServerRun 514
$template FILENAME,"/var/log/%fromhost-ip%/syslog.log"
*.* ?FILENAME
```

* Start the rsyslog server:

```
# atomic run --name rsyslog rhel7/rsyslog
```

###Test the configuration.

* On node 1 open a terminal, make sure rsyslog is started with `atomic` run, and issue the command `logger remote test`.

* On the rsyslog server (node 2), check in the `/var/log/` directory. You should see a directory that has the IP address of the Atomic server (node 1). That directory will have a `syslog.log` file, keep an eye on that file while you run a few more _logger remote test_ commands.

```
# tail -f /var/log/192.168.121.228/syslog.log
Feb 10 09:40:01 localhost CROND[6210]: (root) CMD (/usr/lib64/sa/sa1 1 1)
Feb 10 09:40:03 localhost vagrant: remote test
Feb 10 09:40:05 localhost vagrant: remote test
Feb 10 09:40:07 localhost vagrant: remote test
```

* How do I remove it?

Stop the container and remove the image

```
# atomic uninstall rhel7/rsyslog:7.1-3
```

### More on the Atomic command

* What is the Docker run command being passed to Atomic? Below, you can see that there are a couple of different labels. These are part of the Dockerfile that was used to construct this image. The RUN label shows all the parameters that need to be passed to Docker in order to successfully run this rsyslog image. As you can see, by embedding that into the container and calling it with the Atomic command, it is a lot easier on the user. Atomic abstracts away some of the potential complexity of the docker command.

```
# atomic info rhel7/rsyslog
RUN          : docker run -d --privileged --name NAME --net=host -v /etc/pki/rsyslog:/etc/pki/rsyslog -v /etc/rsyslog.conf:/etc/rsyslog.conf -v /etc/rsyslog.d:/etc/rsyslog.d -v /var/log:/var/log -v /var/lib/rsyslog:/var/lib/rsyslog -v /run/log:/run/log -v /etc/machine-id:/etc/machine-id -v /etc/localtime:/etc/localtime -e IMAGE=IMAGE -e NAME=NAME --restart=always IMAGE /bin/rsyslog.sh
Name         : rsyslog-docker
Build_Host   : rcm-img04.build.eng.bos.redhat.com
Version      : 7.1
Architecture : x86_64
INSTALL      : docker run --rm --privileged -v /:/host -e HOST=/host -e IMAGE=IMAGE -e NAME=NAME IMAGE /bin/install.sh
Release      : 3
Vendor       : Red Hat, Inc.
```

###Using sadc

The sadc container is our "system activity data collector", it is the daemon that runs in the background that provides the ongoing performance data that sar parses and presents to you. This container is meant to run in the background only, it is not an interactive container like rhel-tools.

* Do these steps on node 1 only. Install the sadc container:

```
# atomic install rhel7/sadc
Pulling repository [PRIVATE_REGISTRY]/rhel7/sadc
1a97a9cc4d1b: Download complete 
Status: Downloaded newer image for [PRIVATE_REGISTRY]/rhel7/sadc:latest
docker run --rm --privileged --name sadc -v /:/host -e HOST=/host -e IMAGE=[PRIVATE_REGISTRY]/rhel7/sadc -e NAME=name [PRIVATE_REGISTRY]/rhel7/sadc /usr/local/bin/sysstat-install.sh
Installing file at /host//etc/cron.d/sysstat
Installing file at /host//etc/sysconfig/sysstat
Installing file at /host//etc/sysconfig/sysstat.ioconf
Installing file at /host//usr/local/bin/sysstat.sh
```

* Check the status of the files:

```
# stat /etc/cron.d/sysstat /etc/sysconfig/sysstat /etc/sysconfig/sysstat.ioconf /usr/local/bin/sysstat.sh
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

* Run the container and ensure the container is running:

```
# atomic run rhel7/sadc
docker run -d --privileged --name sadc -v /etc/sysconfig/sysstat:/etc/sysconfig/sysstat -v /etc/sysconfig/sysstat.ioconf:/etc/sysconfig/sysstat.ioconf -v /var/log/sa:/var/log/sa -v /:/host -e HOST=/host -e IMAGE=[PRIVATE_REGISTRY]/rhel7/sadc -e NAME=sadc --net=host --restart=always [PRIVATE_REGISTRY]/rhel7/sadc /usr/local/bin/sysstat.sh
79bf6243c05a9c1a07c7f987ac02b66264ff87ba84cc4714a24a48b3d526ebbc

# docker ps -l
CONTAINER ID        IMAGE                          COMMAND                CREATED             STATUS              PORTS               NAMES
79bf6243c05a        [PRIVATE_REGISTRY]/rhel7/sadc:7.1-3    "/usr/local/bin/syss"   33 seconds ago      Up 32 seconds                           sadc              
```

* Check the status of the files in /var/log/:

```
# stat /var/log/sa/sa*
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

* Run the RHEL Tools container:

```
# atomic run rhel7/rhel-tools
```

* Once inside the RHEL Tools container, run sar and check the output:

```
# sar
Linux 3.10.0-229.el7.x86_64 (atomic-00.localdomain) 	02/27/2015 	_x86_64_	(2 CPU)

08:22:03 PM       LINUX RESTART
```


###Building your own SPC - Example 1


You can build your own SPC using the Dockerfile and the LABEL options. This lab can be done on node 1.

Create a Dockerfile (name it Dockerfile) that looks like:

```dockerfile
FROM    rhel7
MAINTAINER  Your Name
ENV container docker

LABEL INSTALL="/bin/echo This is the install command"
LABEL UNINSTALL="/bin/echo This is the uninstall command"
LABEL RUN="/bin/echo This is the run command"
```

Now build the image:

```
# docker build -t test .
```

Once built, you can test your image:

```
# docker inspect test
# atomic install test
# atomic run test
# atomic uninstall test
```

###Building your own SPC - Example 2

This example will be a bit more complicated. We will introduce _systemd_ and more complex _labels_. 

Set up your directory structure on node 1:

```
# mkdir -vp /root/root/usr/bin; mkdir -vp /root/root/etc/systemd/system/; cd /root/.
```

You can backup or destroy the other Dockerfile. Construct a new Dockerfile (using a private registry) that looks like:

```dockerfile
FROM    [PRIVATE_REGISTRY]/rhel7
MAINTAINER  Your Name
ENV container docker
RUN yum --disablerepo=\* --enablerepo=rhel-7-server-rpms install -y yum-utils
RUN yum-config-manager --disable \*
RUN yum-config-manager --enable rhel-7-server-rpms
RUN yum -y update; yum -y install httpd; yum clean all; systemctl enable httpd

LABEL Version=1.0
LABEL Vendor="Red Hat" License=GPLv3
LABEL INSTALL="docker run --rm --privileged -v /:/host -e HOST=/host -e LOGDIR=${LOGDIR} -e CONFDIR=${CONFDIR} -e DATADIR=${DATADIR} -e IMAGE=IMAGE -e NAME=NAME IMAGE /usr/bin/install.sh"
LABEL UNINSTALL="docker run --rm --privileged -v /:/host -e HOST=/host -e IMAGE=IMAGE -e NAME=NAME IMAGE /usr/bin/uninstall.sh"

LABEL RUN="docker run -dt -p 80 -v /sys/fs/cgroup:/sys/fs/cgroup httpd"

ADD root /

EXPOSE 80

RUN echo "Apache is Working" > /var/www/html/index.html

CMD [ "/sbin/init" ]
```

You also need to create a directory tree under `root` with three files:

* `root/usr/bin/install.sh`
* `root/usr/bin/uninstall.sh`
* `root/etc/systemd/system/httpd_template.service`

Two executable scripts for install and uninstall of the systemd unit file template.


The first file is used by the INSTALL label in the Dockerfile:

```
# vi root/usr/bin/install.sh
#!/bin/sh
# Make Data Dirs
mkdir -p ${HOST}/${CONFDIR} ${HOST}/${LOGDIR}/httpd ${HOST}/${DATADIR}

# Copy Config
cp -pR /etc/httpd ${HOST}/${CONFDIR}

# Create Container
chroot ${HOST} /usr/bin/docker create -v /var/log/${NAME}/httpd:/var/log/httpd:Z -v /var/lib/${NAME}:/var/lib/httpd:Z --name ${NAME} ${IMAGE}

# Install systemd unit file for running container
sed -e "s/TEMPLATE/${NAME}/g" /etc/systemd/system/httpd_template.service > ${HOST}/etc/systemd/system/httpd_${NAME}.service

# Enabled systemd unit file
chroot ${HOST} /usr/bin/systemctl enable httpd_${NAME}.service
```

The second file is used by the UNINSTALL label in the Dockerfile:

```
# vi root/usr/bin/uninstall.sh
#!/bin/sh
chroot ${HOST} /usr/bin/systemctl disable /etc/systemd/system/httpd_${NAME}.service
rm -f ${HOST}/etc/systemd/system/httpd_${NAME}.service
```

Make sure you make them executable:

```
# chmod -v +x root/usr/bin/*.sh
```

This unit file is an example of how you might want to run a containerized service.
Instead of using atomic run, I built the docker commands into the unit file. You could
also use Kubernetes as a mechanism for starting the service.

```
# cat root/etc/systemd/system/httpd_template.service
[Unit]
Description=The Apache HTTP Server for TEMPLATE
After=docker.service

[Service]
ExecStart=/usr/bin/docker start TEMPLATE
ExecStop=/usr/bin/docker stop TEMPLATE
ExecReload=/usr/bin/docker exec -t TEMPLATE /usr/sbin/httpd $OPTIONS -k graceful

[Install]
WantedBy=multi-user.target
```

With those files created, build the container:

```
# docker build -t httpd .
```

You can install multiple apache services with different names and different config data:

```
# atomic install -n test1 httpd
# atomic install -n test2 httpd
```

The Atomic command will create a systemd unit file for each container as well
as Log dir under /var/log/CONTAINERNAME, DATADIR under /var/lib/CONTAINERNAME
and CONFDIR under /etc/CONTAINERNAME which can be used to configure your
services.

Now you need to run the container:

```
# atomic run httpd 
docker run -dt -p 80 -v /sys/fs/cgroup:/sys/fs/cgroup httpd
1847d02d4a68994e048297dd6e65e093cfe4bc9808479201977595f23251dda1
```

Locate the port that the httpd container is listening on:

```
docker ps
```

Finally, you can curl that container and see if apache is running:

```
# curl http://localhost:<port_from_docker_ps>

# curl http://localhost:49156
Apache is Working

```






## [NEXT LAB](configFlannel.md)
