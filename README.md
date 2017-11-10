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
yum -y install lxc-devel
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

groupadd lxd

mkdir -p /var/log/lxd
systemctl enable lxd
systemctl start lxd
ln -s /etc/systemd/system/lxd.service /etc/systemd/system/multi-user.target.wants/lxd.net
Configure LXD

The first time LXD is run, we need to setup the storage backend and virtual network bridge. For optimal performance, we recommend using zfs-backed storage. Create a virtual network bridge, enabling ipv4 but disabling ipv6.

lxd init
After installing, do ``` sudo groupadd lxd``` and add your user to that group.
```

## Unprivileged containers
Add the following line to */etc/default/grub*
```GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX user_namespace.enable=1"```

Update Grub
```grub2-mkconfig -o /boot/grub2/grub.cfg```

Reboot

## Running a container

* Start a container: ```lxc launch images:image-name your-container```

* Find available images: ```lxc image list images:``` or ```lxc image list images: 'centos'```

* Access container: ```lxc exec your-container bash```
   
## Proxy config (Optionnal)
```
lxc config set core.proxy_http http://url-to-proxy
lxc config set core.proxy_http https://url-to-proxy
```
