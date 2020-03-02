# Deploy OpenStack designate with kolla-ansible

During Ocata release, OpenStack DNS-as-a-Service \(Designate\) support was implemented in OpenStack kolla project.

This post will guide you through a basic deployment and tests of designate service.

Install required dependencies and tools for kolla-ansible and designate.

```text
# yum install -y epel-release
# yum install -y python-pip python-devel libffi-devel gcc openssl-devel ansible ntp wget bind-utils
# pip install -U pip
```

Install Docker and downgrade to 1.12.6. At the time of writing this post libvirt had issues to connect with D-Bus due SElinux issues with Docker 1.13.

```text
# curl -sSL https://get.docker.io | bash
# yum downgrade docker-engine-1.12.6 docker-engine-selinux-1.12.6
# yum install -y python-docker-py
```

Configure Docker daemon to allow insecure-registry \(Use the IP where your remote registry will be located\).

```text
# mkdir -p /etc/systemd/system/docker.service.d
# tee /etc/systemd/system/docker.service.d/kolla.conf <<-'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --insecure-registry 172.28.128.3:4000
MountFlags=shared
EOF
```

Reload systemd daemons and start/stop/disable/enable the following services.

```text
# systemctl daemon-reload
# systemctl stop libvirtd
# systemctl disable libvirtd
# systemctl enable ntpd docker
# systemctl start ntpd docker
```

Download Ocata registry created in tarballs.openstack.org, skip this step if images used are custom builds or downloaded from DockerHub. 

Create kolla registry from downloaded tarball.

```text
# wget https://tarballs.openstack.org/kolla/images/centos-binary-registry-ocata.tar.gz
# mkdir /opt/kolla_registry
# sudo tar xzf centos-binary-registry-ocata.tar.gz -C /opt/kolla_registry
# docker run -d -p 4000:5000 --restart=always -v /opt/kolla_registry/:/var/lib/registry --name registry registry:2
```

Install kolla-ansible.

```text
# pip install kolla-ansible
# cp -r /usr/share/kolla-ansible/etc_examples/kolla /etc/kolla/
# cp /usr/share/kolla-ansible/ansible/inventory/* .
```

Configure kolla globals.yml configuration file with the following content. 

Change values when necessary \(IP addresses, interface names\). 

This is a sample minimal configuration.

```text
# vi /etc/kolla/globals.yml
---
kolla_internal_vip_address: "172.28.128.10"
kolla_base_distro: "centos"
kolla_install_type: "binary"
docker_registry: "172.28.128.3:4000"
docker_namespace: "lokolla"
network_interface: "enp0s8"
neutron_external_interface: "enp0s9"
```

Configure designate options in globals.yml. 

dns\_interface must be network reachable from nova instances if internal DNS resolution is needed.

```text
enable_designate: "yes"
dns_interface: "enp0s8"
designate_backend: "bind9"
designate_ns_record: "sample.openstack.org"
```

Configure inventory, add the nodes in their respective groups.

```text
# vi ~/multinode
```

Generate passwords.

```text
# kolla-genpwd
```

Ensure the environment is ready to deploy with prechecks. Until prechecks does not succeed do not start deployment. Fix what is necessary.

```text
# kolla-ansible prechecks -i ~/multinode
```

Pull Docker images on the servers, this can be skipped because will be made in deploy step, but doing it first will ensure all the nodes have the images you need and will minimize the deployment time.

```text
# kolla-ansible pull -i ~/multinode
```

Deploy kolla-ansible and do a woot for kolla ;\)

```text
# kolla-ansible deploy -i ~/multinode
```

Create credentials file and source it.

```text
# kolla-ansible post-deploy -i ~/multinode
# source /etc/kolla/admin-openrc.sh
```

Check that all containers are running and none of them are restarting or exiting.

```text
# docker ps -a --filter status=exited --filter status=restarting
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

Install required python clients

```text
# pip install python-openstackclient python-designateclient python-neutronclient
```

Execute a base OpenStack configuration \(public and internal networks, cirros image\). 

Do no execute this script if custom networks are going to be used.

```text
# sh /usr/share/kolla-ansible/init-runonce
```

Create a sample designate zone.

```text
# openstack zone create --email admin@sample.openstack.org sample.openstack.org.
+----------------+--------------------------------------+
| Field          | Value                                |
+----------------+--------------------------------------+
| action         | CREATE                               |
| attributes     |                                      |
| created_at     | 2017-02-22T13:14:39.000000           |
| description    | None                                 |
| email          | admin@sample.openstack.org           |
| id             | 4a44b0c9-bd07-4f5c-8908-523f453f269d |
| masters        |                                      |
| name           | sample.openstack.org.                |
| pool_id        | 85d18aec-453e-45ae-9eb3-748841a1da12 |
| project_id     | 937d49af6cfe4ef080a79f9a833d7c7d     |
| serial         | 1487769279                           |
| status         | PENDING                              |
| transferred_at | None                                 |
| ttl            | 3600                                 |
| type           | PRIMARY                              |
| updated_at     | None                                 |
| version        | 1                                    |
+----------------+--------------------------------------+
```

Configure designate sink to make use of the previously created zone, sink will need zone\_id to automatically create neutron and nova records into designate.

```text
# mkdir -p /etc/kolla/config/designate/designate-sink/
# vi /etc/kolla/config/designate/designate-sink.conf
[handler:nova_fixed]
zone_id = 4a44b0c9-bd07-4f5c-8908-523f453f269d
[handler:neutron_floatingip]
zone_id = 4a44b0c9-bd07-4f5c-8908-523f453f269d
```

After configure designate-sink.conf, reconfigure designate to make use of this configuration.

```text
# kolla-ansible reconfigure -i ~/multinode --tags designate
```

List networks.

```text
# neutron net-list
+--------------------------------------+----------+----------------------------------+--------------------------------------------------+
| id                                   | name     | tenant_id                        | subnets                                          |
+--------------------------------------+----------+----------------------------------+--------------------------------------------------+
| 3b56c605-5a01-45be-9ed6-e4c3285e4366 | demo-net | 937d49af6cfe4ef080a79f9a833d7c7d | 7f28f050-77b2-426e-b963-35b682077993 10.0.0.0/24 |
| 6954d495-fb8c-4b0b-98a9-9672a7f65b7c | public1  | 937d49af6cfe4ef080a79f9a833d7c7d | 9bd9feca-40a7-4e82-b912-e51b726ad746 10.0.2.0/24 |
+--------------------------------------+----------+----------------------------------+--------------------------------------------------+
```

Update the network with a dns\_domain.

```text
# neutron net-update 3b56c605-5a01-45be-9ed6-e4c3285e4366 --dns_domain sample.openstack.org.
Updated network: 3b56c605-5a01-45be-9ed6-e4c3285e4366
```

Ensure dns\_domain is properly applied.

```text
# neutron net-show 3b56c605-5a01-45be-9ed6-e4c3285e4366
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| availability_zone_hints   |                                      |
| availability_zones        | nova                                 |
| created_at                | 2017-02-22T13:13:06Z                 |
| description               |                                      |
| dns_domain                | sample.openstack.org.                |
| id                        | 3b56c605-5a01-45be-9ed6-e4c3285e4366 |
| ipv4_address_scope        |                                      |
| ipv6_address_scope        |                                      |
| mtu                       | 1450                                 |
| name                      | demo-net                             |
| port_security_enabled     | True                                 |
| project_id                | 937d49af6cfe4ef080a79f9a833d7c7d     |
| provider:network_type     | vxlan                                |
| provider:physical_network |                                      |
| provider:segmentation_id  | 27                                   |
| revision_number           | 6                                    |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   | 7f28f050-77b2-426e-b963-35b682077993 |
| tags                      |                                      |
| tenant_id                 | 937d49af6cfe4ef080a79f9a833d7c7d     |
| updated_at                | 2017-02-22T13:25:16Z                 |
+---------------------------+--------------------------------------+
```

Create a new instance in the previously updated network.

```text
# openstack server create \
    --image cirros \
    --flavor m1.tiny \
    --key-name mykey \
    --nic net-id=3b56c605-5a01-45be-9ed6-e4c3285e4366 \
    demo1
```

Once the instance is ACTIVE, check the IP associated.

```text
# openstack server list
+--------------------------------------+-------+--------+-------------------+------------+
| ID                                   | Name  | Status | Networks          | Image Name |
+--------------------------------------+-------+--------+-------------------+------------+
| d483e4ee-58c2-4e1e-9384-85174630428e | demo1 | ACTIVE | demo-net=10.0.0.3 | cirros     |
+--------------------------------------+-------+--------+-------------------+------------+
```

List records in the designate zone.

As you can see there is a record in designate associated with the instance IP.

```text
# openstack recordset list sample.openstack.org.
+--------------------------------------+----------------------------------+------+-------------------------------------------+--------+--------+
| id                                   | name                             | type | records                                   | status | action |
+--------------------------------------+----------------------------------+------+-------------------------------------------+--------+--------+
| 4f70531e-c325-4ffd-a8d3-8172bd5163b8 | sample.openstack.org.            | SOA  | sample.openstack.org.                     | ACTIVE | NONE   |
|                                      |                                  |      | admin.sample.openstack.org. 1487770304    |        |        |
|                                      |                                  |      | 3586 600 86400 3600                       |        |        |
| a9a09c5f-ccf1-4b52-8400-f36e8faa9549 | sample.openstack.org.            | NS   | sample.openstack.org.                     | ACTIVE | NONE   |
| aa6cd25d-186e-425b-9153-699d8b0811de | 10-0-0-3.sample.openstack.org.   | A    | 10.0.0.3                                  | ACTIVE | NONE   |
| 713650a5-a45e-470b-9539-74e110b15115 | demo1.None.sample.openstack.org. | A    | 10.0.0.3                                  | ACTIVE | NONE   |
| 6506e6f6-f535-45eb-9bfb-4ac1f16c5c9b | demo1.sample.openstack.org.      | A    | 10.0.0.3                                  | ACTIVE | NONE   |
+--------------------------------------+----------------------------------+------+-------------------------------------------+--------+--------+
```

Validate that designate resolves the DNS record. 

You can use designate mDNS service or directly to bind9 servers to validate the test.

```text
# dig +short -p 5354 @172.28.128.3 demo1.sample.openstack.org. A
10.0.0.3
# dig +short -p 53 @172.28.128.3 demo1.sample.openstack.org. A
10.0.0.3
```

If you find any issue with designate in kolla-ansible or kolla, please fill a bug [https://bugs.launchpad.net/kolla-ansible/+filebug](https://bugs.launchpad.net/kolla-ansible/+filebug)

Regards, Eduardo Gonzalez

