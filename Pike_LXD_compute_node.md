# Configuration compute node for LXD
This doc is applied to Pike on CentOS 7. But I think it is also workable on other versions.

## Install compute node
Just follow official Pike document [compute install](https://docs.openstack.org/nova/pike/install/compute-install-rdo.html) to install compute node.

## Upgrade OS
Follow doc [install upgrade kernel version in centos 7](https://www.tecmint.com/install-upgrade-kernel-version-in-centos-7/)

Here, I downloaded 4.14.12 related rpms as local RPM repo. From [here](http://mirror.rackspace.com/elrepo/kernel/el7/x86_64/RPMS/) Then,
```markdown
# yum install kernel-ml

Set system boot to new kernel by default
# sed -i 's/GRUB_DEFAULT=.\*/GRUB_DEFAULT=0/g' /etc/default/grub
# grub2-mkconfig -o /boot/grub2/grub.cfg
# reboot
Make sure new kernel is ok
# uname -a
```

## Install zfs
Compile zfs from source refer to [Appendix A][1], then put zfs rpms into repo in local server [Appendix B][2].

You can also put rpms into [bintray](https://www.bintray.com) for global access. Refer to [Appendix C][3]

Here, I just use local repo.

```markdown
# yum install zfs
```
## Install LXD package
Compile pylxd, nova-lxd, nova-compute-lxd, lxc, lxd and lxd-client from source refer to [Appendix D][4], then put rpms repo in local server or https://www.bintray.com

```markdown
# yum install pylxd nova-lxd lxd lxd-client nova-compute-lxd
```
## Initiate LXD ZFS pool
```markdown
# systemctl start lxd.service
# lxd init

Do you want to configure a new storage pool (yes/no) [default=yes]?
Name of the new storage pool [default=default]: lxd
Name of the storage backend to use (dir, btrfs, ceph, lvm, zfs) [default=zfs]:
Create a new ZFS pool (yes/no) [default=yes]?
Would you like to use an existing block device (yes/no) [default=no]? yes
Path to the existing block device: /dev/sda4
Would you like LXD to be available over the network (yes/no) [default=no]?
Would you like stale cached images to be updated automatically (yes/no) [default=yes]?
Would you like to create a new network bridge (yes/no) [default=yes]? no
WARN[01-11|17:04:06] Did not find profile "default" so no default storage pool will be set. Manual intervention needed.
LXD has been successfully configured.
```

## Change nova configuration
```markdown
# vi /etc/nova/nova.conf
[DEFAULT]
compute_driver = lxd.LXDDriver

[libvirt]
virt_type = lxc

[lxd]
pool = lxd

# systemctl start openstack-nova-compute.service
# systemctl enable openstack-nova-compute.service
```
## AppendixA
### Compile zfs
```markdown
$ wget https://github.com/zfsonlinux/zfs/releases/download/zfs-0.7.5/zfs-0.7.5.tar.gz
$ wget https://github.com/zfsonlinux/zfs/releases/download/zfs-0.7.5/spl-0.7.5.tar.gz
$ tar -zxf zfs-0.7.5.tar.gz
$ tar -zxf spl-0.7.5.tar.gz
$ cd spl-0.7.5
$ ./configure --with-spec=redhat
$ make pkg-utils pkg-kmod spl-dkms
$ cd ../zfs-0.7.5
$ ./configure --with-spec=redhat
$ make pkg-utils pkg-kmod
```
## AppendixB
### Local Repo
Copy rpms to local web server, then run "createrepo ." to create local repo
```markdown
vi /etc/yum.repos.d/local.repo

[local-repo]
name=Local Repo
baseurl=http://10.0.0.101/local_repo
enabled=1
gpgcheck=0
```
# AppendixC
### Create repo on bintray
* Create a repository for rpm
```markdown
For example, https://bintray.com/softway/rpm
```
* Upload rpm to bintray
```markdown
curl -v -T nova-compute-lxd-16.0.0-1.x86_64.rpm \
    -usoftway:<API_KEY> https://api.bintray.com/content/softway/rpm/misc/0.1/7/x86_64/n/
```

# AppendixD
### Compile lxc
```markdown
$ mkdir lxc-stable-2.1
$ git clone -b stable-2.1 https://github.com/lxc/lxc.git .
$ cd lxc-stable-2.1
$ ./autogen.sh
$ ./configure --enable-lua --enable-doc --enable-api-docs
$ make rpm
rpms are under dir: ~/rpmbuild/RPMS/x86_64/
```
### Compile pylxd
```markdown
$ git clone https://github.com/lxc/pylxd.git
$ cd pylxd
$ python setup.py bdist_rpm 
```
### Compile nova-lxd
```markdown
$ git clone https://github.com/openstack/nova-lxd.git
$ cd nova-lxd
$ python setup.py bdist_rpm
```
### Compile lxd, lxd-client, nova-compute-lxd
```markdown
$ mkdir -p $GOPATH/src/github.com/lxc
$ cd $GOPATH/src/github.com/lxc
$ git clone -b lxd-2.21 https://github.com/lxc/lxd.git
$ cd lxd
$ make
```
lxc and lxd will be in $GOPATH/bin/lxd, then make rpm package

Fetch rpm stuff

```markdown
$ mkdir -p $GOPATH/src/github.com/soft-way
$ cd $GOPATH/src/github.com/soft-way
$ git clone https://github.com/soft-way/lxc.git
$ cp -rp $GOPATH/src/github.com/soft-way/lxc/lxd/* $GOPATH/src/github.com/lxc/lxd/
$ $GOPATH/src/github.com/lxc/lxd/
```
### lxd-2.21.0-1.x86_64.rpm (pkg-build/RPMS/x86_64/)
```markdown
$ cp $GOPATH/bin/lxd rpm/
$ help2man $GOPATH/bin/lxd > rpm/lxd.1
$ gzip rpm/lxd.1
```
For each markdown document under doc subdir, do below:
```markdown
$ md2man-roff ../doc/cloud-init.md > rpm/lxd.cloud-init.md.1
$ gzip rpm/lxd.cloud-init.md.1
$ go-bin-rpm generate -f rpm-lxd.json -o .
```
### lxd-client-2.21.0-1.x86_64.rpm (pkg-build/RPMS/x86_64/)
```markdown
$ cp $GOPATH/bin/lxc rpm/
$ help2man $GOPATH/bin/lxc > rpm/lxc.1
$ gzip rpm/lxc.1
$ go-bin-rpm generate -f rpm-lxd-client.json -o .
```
### nova-compute-lxd-16.0.0-1.x86_64.rpm (pkg-build/RPMS/x86_64/)
```markdown
$ go-bin-rpm generate -f rpm-nova-compute-lxd.json -o .
```
go-bin-rpm can be installed from https://github.com/mh-cbon/go-bin-rpm

[1]: #appendixa
[2]: #appendixb
[3]: #appendixc
[4]: #appendixd
