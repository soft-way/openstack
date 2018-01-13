# Configuration compute node as LXD
This doc is applied to Pike on CentOS 7. But I think it is also workable on other version.

## Install compute node
Just follow official Pike document
https://docs.openstack.org/nova/pike/install/compute-install-rdo.html

## Upgrade OS
Follow doc https://www.tecmint.com/install-upgrade-kernel-version-in-centos-7/

Here, I downloaded 4.14.12 related rpms as local RPM repo. From this mirror: http://mirror.rackspace.com/elrepo/kernel/el7/x86_64/RPMS/
Then,

\# yum install kernel-ml

### set system boot to new kernel by default
\# sed -i 's/GRUB_DEFAULT=.*/GRUB_DEFAULT=0/g' /etc/default/grub

\# grub2-mkconfig -o /boot/grub2/grub.cfg

\# reboot

### make sure new kernel is ok
\# uname -a

## install zfs
Compile zfs from source refer to Appendix, then put zfs rpms into local repo.

\# yum install zfs

## Install LXD package
Compile pylxd, nova-lxd, nova-compute-lxd, lxc, lxd and lxd-client from source refer to Appendix, then put rpms into local repo.

\# yum install pylxd nova-lxd lxd lxd-client nova-compute-lxd

## Initiate LXD ZFS pool
\# systemctl start lxd.service

\# lxd init

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


## Change nova configuration
vi /etc/nova/nova.conf

[DEFAULT]

compute_driver = lxd.LXDDriver

[libvirt]

virt_type = lxc

[lxd]

pool = lxd

\# systemctl start openstack-nova-compute.service
\# systemctl enable openstack-nova-compute.service


Appendix
## Compile LXD
\# ./configure --enable-lua --enable-doc --enable-api-docs

