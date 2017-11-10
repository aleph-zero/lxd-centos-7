# lxd-centos-7
Tips for using LXD on CentOS 7

## General notes
LXD seems to generally work on CentOS 7. The main limitation as far as I can tell is that you must run containers in privileged mode. The root cause is explained here: https://bugs.centos.org/view.php?id=10273. Unfortunately it does not seem to be high on CentOS's priority, so we may have to live with privileged containers for a while.

## Installation
This github explains how to get LXD up and running on a fresh CentOS 7 ```uname -r 3.10.0-514.el7.x86_64``` : https://gist.github.com/AdamIsrael/e6b4a540233a7be6d28decd917b60c3d. You can ignore the part about Juju. I'm copying the instructions here b/c I'm lazy.
```
Configure epel repository

yum -y install epel-release
yum repolist
Install LXD

Install lxc libraries

yum -y install python-requests
rpm --nodeps -i https://copr-be.cloud.fedoraproject.org/results/alonid/yum-plugin-copr/epel-7-x86_64/00110045-yum-plugin-copr/yum-plugin-copr-1.1.31-508.el7.centos.noarch.rpm
yum copr enable thm/lxc2.0
yum -y install lxc lxc-devel
Build from source

yum install -y golang-bin git make dnsmasq squashfs-tools libacl-devel
mkdir ~/go
export GOPATH=~/go
go get github.com/lxc/lxd
cd $GOPATH/src/github.com/lxc/lxd
make
Add to path

cp $GOPATH/bin/* /usr/bin
Start LXD daemon

We want to setup systemd to manage LXD, and ensure that LXD will start after a reboot.

Create the file /etc/systemd/system/lxd.service with the following contents:

[Unit]
Description=LXD Container Hypervisor
Requires=network.service

[Service]
ExecStart=/usr/bin/lxd --group lxd --logfile=/var/log/lxd/lxd.log
KillMode=process
TimeoutStopSec=40
KillSignal=SIGPWR
Restart=on-failure
LimitNOFILE=-1
LimitNPROC=-1
Once you've saved that file, enable, start, and set the service to start on boot:

addgroup lxd

mkdir -p /var/log/lxd
systemctl enable lxd
systemctl start lxd
ln -s /etc/systemd/system/lxd.service /etc/systemd/system/multi-user.target.wants/lxd.net
Configure LXD

The first time LXD is run, we need to setup the storage backend and virtual network bridge. For optimal performance, we recommend using zfs-backed storage. Create a virtual network bridge, enabling ipv4 but disabling ipv6.

lxd init
After installing, do ``` sudo groupadd lxd``` and add your user to that group.
```

## Post-install verification

There is a handy tool called **lxc-checkconfig** that tells you some interesting stuff. Not sure what the **newuidmap** and **newgidmap** stuff is, but it seems to be related to unprivileged containers, which we can't use anyway (see above).

```
[ burner ~ ] [ 06:15:11 ] > lxc-checkconfig 
Kernel configuration not found at /proc/config.gz; searching...
Kernel configuration found at /boot/config-3.10.0-514.el7.x86_64
--- Namespaces ---
Namespaces: enabled
Utsname namespace: enabled
Ipc namespace: enabled
Pid namespace: enabled
User namespace: enabled
newuidmap is not installed
newgidmap is not installed
Network namespace: enabled
Multiple /dev/pts instances: enabled

--- Control groups ---
Cgroup: enabled
Cgroup clone_children flag: enabled
Cgroup device: enabled
Cgroup sched: enabled
Cgroup cpu account: enabled
Cgroup memory controller: enabled
Cgroup cpuset: enabled

--- Misc ---
Veth pair device: enabled
Macvlan: enabled
Vlan: enabled
Bridges: enabled
Advanced netfilter: enabled
CONFIG_NF_NAT_IPV4: enabled
CONFIG_NF_NAT_IPV6: enabled
CONFIG_IP_NF_TARGET_MASQUERADE: enabled
CONFIG_IP6_NF_TARGET_MASQUERADE: enabled
CONFIG_NETFILTER_XT_TARGET_CHECKSUM: enabled
FUSE (for use with lxcfs): enabled

--- Checkpoint/Restore ---
checkpoint restore: enabled
CONFIG_FHANDLE: enabled
CONFIG_EVENTFD: enabled
CONFIG_EPOLL: enabled
CONFIG_UNIX_DIAG: enabled
CONFIG_INET_DIAG: enabled
CONFIG_PACKET_DIAG: enabled
CONFIG_NETLINK_DIAG: enabled
File capabilities: enabled

Note : Before booting a new kernel, you can check its configuration
usage : CONFIG=/path/to/config /usr/bin/lxc-checkconfig

```

## Running a container

I haven't been able to launch a container directly due to the unprivileged issue. Following the docs and running something like ```lxc launch images:image-name your-container``` wouldn't work for me.

Instead follow these instructions.
  * Find available images:
  ```
  [ burner ~ ] [ 05:59:42 ] > lxc image list images: 'centos'
+------------------------+--------------+--------+---------------------------------+--------+---------+------------------------------+
|         ALIAS          | FINGERPRINT  | PUBLIC |           DESCRIPTION           |  ARCH  |  SIZE   |         UPLOAD DATE          |
+------------------------+--------------+--------+---------------------------------+--------+---------+------------------------------+
| centos/6 (3 more)      | 549db7214381 | yes    | Centos 6 amd64 (20170504_02:16) | x86_64 | 65.42MB | May 4, 2017 at 12:00am (UTC) |
+------------------------+--------------+--------+---------------------------------+--------+---------+------------------------------+
| centos/6/i386 (1 more) | 28fb7504e3ed | yes    | Centos 6 i386 (20170504_02:16)  | i686   | 65.34MB | May 4, 2017 at 12:00am (UTC) |
+------------------------+--------------+--------+---------------------------------+--------+---------+------------------------------+
| centos/7 (3 more)      | 41c7bb494bbd | yes    | Centos 7 amd64 (20170504_02:16) | x86_64 | 65.33MB | May 4, 2017 at 12:00am (UTC) |
+------------------------+--------------+--------+---------------------------------+--------+---------+------------------------------+
```
  * Copy image locally: ```lxc image copy images:41c7bb494bbd local: --alias centos-7```
  * Create the container from the image without starting it: ```lxc init centos-7 c2```
  * Configure the image to be **unprivileged**: ```lxc config set c2 security.privileged true```
  * Verify that it has been changed to unpriviledged:
  ```
   [ burner ~ ] [ 06:35:38 ] >  lxc config get c2 security.privileged
   true
   ```
  * Start the container: ```lxc start c2```
  * Confirm it is running:
  ```
  [ burner ~ ] [ 06:37:02 ] > lxc list
+------+---------+----------------------+------+------------+-----------+
| NAME |  STATE  |         IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
+------+---------+----------------------+------+------------+-----------+
| c2   | RUNNING |                      |      | PERSISTENT | 0         |
+------+---------+----------------------+------+------------+-----------+
```
  * Done
   
  
