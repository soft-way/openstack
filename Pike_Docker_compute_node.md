# Configuration for compute node with docker
This doc is applied to Pike on CentOS 7. But I think it is also workable on other versions.

## Install compute node
Just follow official Pike document [compute install](https://docs.openstack.org/nova/pike/install/compute-install-rdo.html) to install compute node.

## Setup docker
Just folllow [docker doc](https://docs.docker.com/engine/installation/linux/docker-ce/centos/)
```markdown
Download docker rpms from https://download.docker.com/linux/centos/7/x86_64/stable/Packages/ to local repo, 
then run:
# yum install docker-ce
# systemctl start docker.service
# systemctl enable docker.service
```
## Kuryr libnetwork installation
Refer to [official kuryr installation](https://docs.openstack.org/kuryr-libnetwork/latest/install)
```markdown
$ openstack user create --domain default --password yourpass kuryr
$ openstack role add --project service --user kuryr admin
```
### Install kuryr-libnetwork rpm
 Refer to [Appendix A][1] on how to compile rpm.
 
 The rpms can also be put into [bintray](https://www.bintray.com) for global access. Refer to ![Appendix C](./Pike_LXD_compute_node.md#appendixc) 

```markdown
# yum install kuryr-libnetwork
# vi /etc/kuryr/kuryr.conf
[DEFAULT]
bindir = /usr/bin/kuryr-server
capability_scope = global

[neutron]
auth_uri = http://pike-c7:5000
auth_url = http://pike-c7:35357
username = kuryr
user_domain_name = default
password = servicepass
project_name = service
project_domain_name = default
auth_type = password

# systemctl start kuryr-libnetwork
# systemctl enable kuryr-libnetwork

```
### Configure Docker and Kuryr
```markdown
# mkdir -p /etc/systemd/system/docker.service.d
# vi /etc/systemd/system/docker.service.d/docker.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 \
        -H unix:///var/run/docker.sock \
        --cluster-advertise 10.0.0.104:2375 \
        --cluster-store etcd://10.0.0.101:2379 \
        --cluster-store-opt kv.cacertfile=/etc/etcd/ca.pem \
        --cluster-store-opt kv.certfile=/etc/etcd/compute-04.pem \
        --cluster-store-opt kv.keyfile=/etc/etcd/compute-04-key.pem

If the server is under proxy, add below file:
vi /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://10.0.0.101:8000/"

# systemctl restart docker.service

# vi /etc/kuryr/kuryr.conf
[DEFAULT]
bindir = /usr/bin/kuryr-server
capability_scope = global

[neutron]
auth_uri = http://pike-c7:5000
auth_url = http://pike-c7:35357
username = kuryr
user_domain_name = default
password = yourpass
project_name = service
project_domain_name = default
auth_type = password

# systemctl enable kuryr-libnetwork
# systemctl start kuryr-libnetwork
```

# AppendixA
### Compile kuryr-libnetwork
```markdown
$ git clone -b 0.2.0 https://github.com/openstack/kuryr-libnetwork.git
Add kuryr-libnetwork.service into rpm
$ cd kuryr-libnetwork
$ mkdir rpm
$ vi rpm/kuryr-libnetwork.service
[Unit]
Description = Kuryr-libnetwork - Docker network plugin for Neutron

[Service]
ExecStart = /usr/bin/kuryr-server --config-file /etc/kuryr/kuryr.conf
CapabilityBoundingSet = CAP_NET_ADMIN

[Install]
WantedBy = multi-user.target

Add pre-install script into rpm
$ vi rpm/preinst.sh
echo "preinst"
getent group kuryr >/dev/null || groupadd -r kuryr
if ! getent passwd kuryr >/dev/null; then
  useradd --home-dir "/var/lib/kuryr" \
      --create-home \
      -r \
      --shell /bin/false \
      -g kuryr \
      kuryr
fi
mkdir -p /etc/kuryr
chown kuryr:kuryr /etc/kuryr

mkdir -p /var/log/kuryr
chown -R root:root /var/log/kuryr

exit 0

Get dependency from requirements.txt and build rpm
$ python setup.py bdist_rpm --requires python2-babel >= 2.3.4,python-flask >= 0.10.1,python-ipaddress >= 1.0.16,python-jsonschema >= 2.3.0,python2-kuryr-lib >= 0.6.0,python-neutron-lib >= 1.9.1,python2-os-client-config >= 1.28.0,python2-oslo-concurrency >= 3.21.1,python2-oslo-config >= 4.11.1,python2-oslo-log >= 3.22.0,python2-oslo-utils >= 3.28.0,python2-pbr >= 3.1.1,python2-neutronclient >= 6.3.0,python2-six >= 1.10.0,python2-docker >= 2.4.2 --release 1 --pre-install rpm/preinst.sh

```

[1]: #appendixa
[2]: #appendixb
[3]: #appendixc
[4]: #appendixd
