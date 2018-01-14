# Configuration compute node for docker
This doc is applied to Pike on CentOS 7. But I think it is also workable on other versions.

## Install compute node
Just follow official Pike document [compute install](https://docs.openstack.org/nova/pike/install/compute-install-rdo.html) to install compute node.

## Install docker
Just folllow [docker doc](https://docs.docker.com/engine/installation/linux/docker-ce/centos/)
```markdown
Download docker rpms from https://download.docker.com/linux/centos/7/x86_64/stable/Packages/ to local repo, 
then run:
# yum install docker-ce
# systemctl start docker.service
# systemctl enable docker.service
```
##  Kuryr libnetwork installation
Refer to [official kuryr installation](https://docs.openstack.org/kuryr-libnetwork/latest/install)
```markdown
$ openstack user create --domain default --password yourpass kuryr
$ openstack role add --project service --user kuryr admin
```
### Install kuryr-libnetwork rpm
 Refer to [Appendix A][1] on how compile rpm.
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
# mkdir -p mkdir /etc/systemd/system/docker.service.d
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

# systemctl restart docker

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

[1]: #appendixa
[2]: #appendixb
[3]: #appendixc
[4]: #appendixd
