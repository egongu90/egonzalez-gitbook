---
description: from RDO packages
---

# OpenStack kolla deployment

OpenStack, Ansible, Docker, production ready, HA, etc. Nothing can be so interesting as Kolla. \| Kolla includes all you need to create, maintain and operate an OpenStack environment. \| All the services will be installed along the nodes you specify inside docker containers with high availability and load balancing between services by default, you don\'t need to care about an external tool for these purposes. \| In future posts, i will talk in more detail about Kolla and how works, also more tips or deployment types. For now, go to the official documentation. \| At this demo, i will use:

* x1 Deployment node: Laptop with 12GB of RAM and a single CPU
* x3 Target nodes: VMs with 24GB of RAM and 2 vCPU each one.
* All nodes connected to a shared connection with 300Mbs

### ALL NODES

Before deploy OpenStack with Kolla, we need to ensure all the nodes got time synchronized.

```text
yum -y install ntp
systemctl enable ntpd.service
systemctl start ntpd.service
```

Next, stop and disable libvirt service to avoid conflicts with libvirt containers.

```text
systemctl stop libvirtd
systemctl disable libvirtd
```

Install docker

```text
curl -sSL https://get.docker.io | bash
```

Add the user you are using to docker group so this user can issue docker commands without sudo. Logoff and login to apply changes.

```text
sudo usermod -aG docker root
```

Create a file called kolla.conf with the following content.

```text
vi /etc/systemd/system/docker.service.d/kolla.conf
[Service]
MountFlags=shared
```

Restart and enable docker service

```text
systemctl restart docker
systemctl enable docker
```

Install some packages who are needed by next steps.

```text
yum install -y python-devel libffi-devel openssl-devel gcc git python-pip python-openstackclient
```

#### DEPLOY NODE

Install EPEL repository

```text
yum install -y epel-release
```

Install ansible

```text
yum install -y ansible
```

Clone Kolla mitaka/stable code.

```text
git clone https://git.openstack.org/openstack/kolla -b stable/mitaka
```

Install kolla and dependencies.

```text
pip install kolla/
```

Copy kolla configuration files to /etc/

```text
cd kolla
cp -r etc/kolla /etc/
```

Create kolla build config file

```text
pip install tox
tox -e genconfig
```

Edit kolla-build file with the following content

```text
vi /etc/kolla/kolla-build.conf 

base = centos
base_tag = mitaka
push = true
install_type = rdo
registry = docker.io
```

Login with your DockerHub account, sometimes, login doesn\'t works as expected. Review auth url at authentication file in ~/.docker/ directory. After Austin Summit i will post exact changes i made in the URL.

```text
docker login
```

\| Create and push the images to your DockerHub account. \| If images are not automatically pushed to the remote repository, push them manually once image creation finished. \| Building images can last various hours, in my experience sometimes were built in 3 hours and another times in 9 hours. And much more if you are going to push them to your DockerHub instead of a private registry.

```text
kolla-build -n egonzalez90 --push
```

Review all docker images kolla has created.

```text
[egonzalez@localhost kolla]$ docker images | grep mitaka
egonzalez90/centos-binary-cinder-api                  mitaka              ba2cca4b09fa        16 hours ago        814.5 MB
egonzalez90/centos-binary-cinder-volume               mitaka              1d31a049f327        16 hours ago        802.4 MB
egonzalez90/centos-binary-cinder-rpcbind              mitaka              5f7bc909f41b        16 hours ago        804.2 MB
egonzalez90/centos-binary-mesos-slave                 mitaka              57a0e00d1901        16 hours ago        651.6 MB
egonzalez90/centos-binary-swift-rsyncd                mitaka              36f5b9c9d4c5        16 hours ago        565.3 MB
egonzalez90/centos-binary-cinder-backup               mitaka              a7a8161398fe        16 hours ago        775.3 MB
egonzalez90/centos-binary-cinder-scheduler            mitaka              a5c5b79a25f6        16 hours ago        775.3 MB
egonzalez90/centos-binary-marathon                    mitaka              704ce8261a7f        16 hours ago        770.4 MB
egonzalez90/centos-binary-chronos                     mitaka              974525562cea        16 hours ago        732.8 MB
egonzalez90/centos-binary-swift-object                mitaka              e09b529bad32        16 hours ago        582.9 MB
egonzalez90/centos-binary-swift-account               mitaka              573b8e5bd3c7        16 hours ago        582.9 MB
egonzalez90/centos-binary-swift-container             mitaka              c63d9a5be014        16 hours ago        583.2 MB
egonzalez90/centos-binary-mesos-master                mitaka              2610881df9c0        16 hours ago        536.8 MB
egonzalez90/centos-binary-swift-proxy-server          mitaka              3632ee65ace9        16 hours ago        584.7 MB
egonzalez90/centos-binary-ceilometer-api              mitaka              808cd12e9287        16 hours ago        598.6 MB
egonzalez90/centos-binary-ceilometer-compute          mitaka              59e7a5e3bd79        16 hours ago        612.6 MB
egonzalez90/centos-binary-ceilometer-central          mitaka              de094dabf9fd        16 hours ago        612.6 MB
egonzalez90/centos-binary-magnum-api                  mitaka              6ce41a1856f8        16 hours ago        690 MB
egonzalez90/centos-binary-glance-api                  mitaka              2a1c8702341a        16 hours ago        688.5 MB
egonzalez90/centos-binary-ceilometer-notification     mitaka              7ccb484383ae        16 hours ago        594 MB
egonzalez90/centos-binary-ceilometer-collector        mitaka              c2e043f6e2b1        16 hours ago        595.4 MB
egonzalez90/centos-binary-magnum-conductor            mitaka              19674f37dc9b        16 hours ago        790.8 MB
egonzalez90/centos-binary-aodh-api                    mitaka              c35c48dee3c4        16 hours ago        593.2 MB
egonzalez90/centos-binary-glance-registry             mitaka              a72949aaaf45        16 hours ago        688.5 MB
egonzalez90/centos-binary-aodh-expirer                mitaka              ffa9bc296a02        16 hours ago        593.2 MB
egonzalez90/centos-binary-aodh-evaluator              mitaka              c214eac9bbd9        16 hours ago        593.2 MB
egonzalez90/centos-binary-neutron-metadata-agent      mitaka              0cea7ba50b8e        16 hours ago        817.9 MB
egonzalez90/centos-binary-aodh-listener               mitaka              c5d255b20d4e        16 hours ago        593.2 MB
egonzalez90/centos-binary-aodh-notifier               mitaka              dbd4c8d5515d        16 hours ago        593.2 MB
egonzalez90/centos-binary-neutron-server              mitaka              688d6800684b        16 hours ago        817.9 MB
egonzalez90/centos-binary-gnocchi-api                 mitaka              5f8daeb7a511        17 hours ago        840.8 MB
egonzalez90/centos-binary-neutron-openvswitch-agent   mitaka              3c2f03d388fa        17 hours ago        843.4 MB
egonzalez90/centos-binary-nova-compute                mitaka              aef19eb18b41        17 hours ago        1.076 GB
egonzalez90/centos-binary-neutron-linuxbridge-agent   mitaka              672550e296af        17 hours ago        843.1 MB
egonzalez90/centos-binary-nova-libvirt                mitaka              46cd6d68a29d        17 hours ago        1.127 GB
egonzalez90/centos-binary-gnocchi-statsd              mitaka              8369b97d0fb7        17 hours ago        840.7 MB
egonzalez90/centos-binary-neutron-dhcp-agent          mitaka              b6a6de5c4d3f        17 hours ago        817.9 MB
egonzalez90/centos-binary-neutron-l3-agent            mitaka              6d4956cd63e6        17 hours ago        817.9 MB
egonzalez90/centos-binary-nova-spicehtml5proxy        mitaka              6db500ef18b0        17 hours ago        629.5 MB
egonzalez90/centos-binary-nova-compute-ironic         mitaka              89f4f8ba32b9        17 hours ago        1.04 GB
egonzalez90/centos-binary-nova-conductor              mitaka              71e00696b65a        17 hours ago        629.4 MB
egonzalez90/centos-binary-nova-novncproxy             mitaka              4153ed5cdfa5        17 hours ago        630 MB
egonzalez90/centos-binary-nova-api                    mitaka              7bf702527a50        17 hours ago        629.4 MB
egonzalez90/centos-binary-nova-ssh                    mitaka              0c71e10ba8bb        17 hours ago        630.4 MB
egonzalez90/centos-binary-nova-network                mitaka              ff2ed3dc65ab        17 hours ago        630.4 MB
egonzalez90/centos-binary-heat-api                    mitaka              3f3bac2b91b4        17 hours ago        592.2 MB
egonzalez90/centos-binary-nova-consoleauth            mitaka              f7f558ed3061        17 hours ago        629.5 MB
egonzalez90/centos-binary-nova-scheduler              mitaka              f9b8750d4812        17 hours ago        629.4 MB
egonzalez90/centos-binary-heat-engine                 mitaka              69b416b2481c        17 hours ago        592.2 MB
egonzalez90/centos-binary-heat-api-cfn                mitaka              220acaf5f692        18 hours ago        592.2 MB
egonzalez90/centos-binary-manila-api                  mitaka              3e21270b4e91        18 hours ago        588.4 MB
egonzalez90/centos-binary-trove-api                   mitaka              68868b718307        18 hours ago        585.8 MB
egonzalez90/centos-binary-manila-share                mitaka              45e069ec5233        18 hours ago        637.8 MB
egonzalez90/centos-binary-trove-guestagent            mitaka              484a9b5b5631        18 hours ago        586.1 MB
egonzalez90/centos-binary-trove-conductor             mitaka              2817941fed43        18 hours ago        585.8 MB
egonzalez90/centos-binary-trove-taskmanager           mitaka              16fc85e299a1        18 hours ago        585.8 MB
egonzalez90/centos-binary-manila-scheduler            mitaka              075beb4c058e        18 hours ago        588.4 MB
egonzalez90/centos-binary-designate-api               mitaka              0dfb2e4b971d        18 hours ago        589.8 MB
egonzalez90/centos-binary-designate-central           mitaka              d4ab5d846989        18 hours ago        589.8 MB
egonzalez90/centos-binary-designate-poolmanager       mitaka              17570055aa01        18 hours ago        594.3 MB
egonzalez90/centos-binary-designate-sink              mitaka              16e1113010dd        18 hours ago        589.8 MB
egonzalez90/centos-binary-designate-backend-bind9     mitaka              a83d15642a07        18 hours ago        594.3 MB
egonzalez90/centos-binary-cinder-base                 mitaka              ebc196468197        18 hours ago        775.3 MB
egonzalez90/centos-binary-ironic-pxe                  mitaka              3b825ca5e758        18 hours ago        595.2 MB
egonzalez90/centos-binary-ironic-api                  mitaka              53b3a144266a        18 hours ago        591.6 MB
egonzalez90/centos-binary-zookeeper                   mitaka              91270c923346        18 hours ago        544.8 MB
egonzalez90/centos-binary-designate-mdns              mitaka              2de6dfb55068        18 hours ago        589.8 MB
egonzalez90/centos-binary-ironic-inspector            mitaka              631d5c362116        18 hours ago        597.4 MB
egonzalez90/centos-binary-ironic-conductor            mitaka              aceccff4bef0        18 hours ago        620.3 MB
egonzalez90/centos-binary-horizon                     mitaka              b8a5f7db8daf        18 hours ago        690.6 MB
egonzalez90/centos-binary-swift-base                  mitaka              c98164063b84        18 hours ago        563.7 MB
egonzalez90/centos-binary-mesos-base                  mitaka              a50e0e1e8edc        18 hours ago        536.5 MB
egonzalez90/centos-binary-ceilometer-base             mitaka              07164b2054b8        18 hours ago        574.2 MB
egonzalez90/centos-binary-glance-base                 mitaka              b40e34f047d7        18 hours ago        688.5 MB
egonzalez90/centos-binary-magnum-base                 mitaka              bad9157e57ba        18 hours ago        668.3 MB
egonzalez90/centos-binary-aodh-base                   mitaka              9a919ceb1213        19 hours ago        573.5 MB
egonzalez90/centos-binary-neutron-base                mitaka              7669e9646a22        19 hours ago        817.9 MB
egonzalez90/centos-binary-gnocchi-base                mitaka              509a5c7395fb        19 hours ago        817.5 MB
egonzalez90/centos-binary-keystone                    mitaka              231990ed7b4d        19 hours ago        606.4 MB
egonzalez90/centos-binary-nova-base                   mitaka              a4523a00e9b2        19 hours ago        608.8 MB
egonzalez90/centos-binary-zaqar                       mitaka              43b8675a9bda        19 hours ago        607.4 MB
egonzalez90/centos-binary-heat-base                   mitaka              10662065592f        19 hours ago        572.6 MB
egonzalez90/centos-binary-manila-base                 mitaka              215fc8275580        19 hours ago        588.4 MB
egonzalez90/centos-binary-trove-base                  mitaka              0eda6621a5c3        19 hours ago        566.5 MB
egonzalez90/centos-binary-designate-base              mitaka              dc53110d609c        19 hours ago        570.2 MB
egonzalez90/centos-binary-dind                        mitaka              f2e7bbe028b4        19 hours ago        539.3 MB
egonzalez90/centos-binary-tempest                     mitaka              28cceef2319d        19 hours ago        628 MB
egonzalez90/centos-binary-ironic-base                 mitaka              7b52957bf3a0        19 hours ago        572 MB
egonzalez90/centos-binary-openvswitch-db-server       mitaka              a624dd2d260d        19 hours ago        379 MB
egonzalez90/centos-binary-openvswitch-vswitchd        mitaka              4c36af8e0e44        20 hours ago        379 MB
egonzalez90/centos-binary-ceph-mon                    mitaka              81486c6a7605        20 hours ago        553.3 MB
egonzalez90/centos-binary-kolla-toolbox               mitaka              3fc4535c3d5e        20 hours ago        675.4 MB
egonzalez90/centos-binary-elasticsearch               mitaka              0a81ba71ec7f        20 hours ago        576.4 MB
egonzalez90/centos-binary-keepalived                  mitaka              3559905c7d86        20 hours ago        409.3 MB
egonzalez90/centos-binary-ceph-osd                    mitaka              26dc5c40e160        20 hours ago        553.3 MB
egonzalez90/centos-binary-heka                        mitaka              919dd5a93ca3        20 hours ago        420.6 MB
egonzalez90/centos-binary-rabbitmq                    mitaka              4ab020955a66        20 hours ago        552.7 MB
egonzalez90/centos-binary-mesosphere-base             mitaka              a9f2a4c7cf1c        20 hours ago        381.9 MB
egonzalez90/centos-binary-openstack-base              mitaka              46a527edf49a        20 hours ago        539.3 MB
egonzalez90/centos-binary-ceph-rgw                    mitaka              f57ab1371bd3        20 hours ago        553.3 MB
egonzalez90/centos-binary-openvswitch-base            mitaka              f91c5a909b2c        20 hours ago        379 MB
egonzalez90/centos-binary-mariadb                     mitaka              8fe89c13a637        20 hours ago        678.6 MB
egonzalez90/centos-binary-cron                        mitaka              a239ea240c2e        20 hours ago        366.7 MB
egonzalez90/centos-binary-mongodb                     mitaka              48946c962d7e        20 hours ago        539.2 MB
egonzalez90/centos-binary-ceph-base                   mitaka              02be30a43c6e        20 hours ago        553.3 MB
egonzalez90/centos-binary-haproxy                     mitaka              b8d8ac3e371d        20 hours ago        367.4 MB
egonzalez90/centos-binary-memcached                   mitaka              175026eb6466        20 hours ago        404.1 MB
egonzalez90/centos-binary-kibana                      mitaka              885aeb0b2b97        20 hours ago        490.9 MB
egonzalez90/centos-binary-mesos-dns                   mitaka              95e29f8429e7        21 hours ago        361 MB
egonzalez90/centos-binary-base                        mitaka              b104d01004c6        21 hours ago        349.2 MB
```

### TARGET HOSTS

In target nodes, a newer version of pip and docker-py is needed, install it.

```text
sudo pip install -U pip
pip install -U docker-py
```

### DEPLOY KOLLA

Kolla ships a tool to create random passwords, issue this command to run this tool. Also, you can modify passwords file at /etc/kolla/ directory.

```text
kolla-genpwd
```

\| Edit globals.yml file with the following content, use your own info if necessary. \| Change docker\_namespace with your docker account name.

```text
vi /etc/kolla/globals.yml

kolla_base_distro: "centos"
kolla_install_type: "binary"
openstack_release: "mitaka" ## Tag at docker hub
kolla_internal_vip_address: "192.168.1.90"
docker_registry: "docker.io"
docker_namespace: "egonzalez90"
network_interface: "eth2"
neutron_external_interface: "ens9"
```

Edit the inventory file with your server\'s IPs or hostnames.

```text
vi ansible/inventory/multinode

[control]
# These hostname must be resolvable from your deployment host
192.168.1.77
192.168.1.74
192.168.1.78

# The network nodes are where your l3-agent and loadbalancers will run
# This can be the same as a host in the control group
[network]
192.168.1.77
192.168.1.74
192.168.1.78

[compute]
192.168.1.77
192.168.1.74
192.168.1.78

# When compute nodes and control nodes use different interfaces,
# you can specify "api_interface" and another interfaces like below:
#compute01 neutron_external_interface=eth0 api_interface=em1 storage_interface=em1 tunnel_interface=em1

[storage]
192.168.1.77
192.168.1.74
192.168.1.78
```

Create an SSH key to login into target servers.

```text
[root@kolla-deployment-node kolla]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
bd:3e:ce:7c:2a:6b:a7:99:ed:04:cf:c2:60:5f:2f:12 root@kolla-deployment-node
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|                 |
|         .       |
|      o E o      |
|     . + * o     |
|        = * .    |
|        o@o..    |
|       .=BO+     |
+-----------------+
```

Copy the SSH key you have previously created to all your target nodes.

```text
[root@kolla-deployment-node kolla]# ssh-copy-id root@192.168.1.77
[root@kolla-deployment-node kolla]# ssh-copy-id root@192.168.1.74
[root@kolla-deployment-node kolla]# ssh-copy-id root@192.168.1.78
```

\| Ensure all hostnames can be resolved between all the nodes, this is a necessary step, if not, rabbitmq will fail. \| If using a DNS server you can skip this task. \| Configure hosts file.

```text
vi /etc/hosts

192.168.1.77 node1
192.168.1.74 node2
192.168.1.78 node3
```

Copy hosts file to the other nodes.

```text
scp /etc/hosts root@node2:/etc/hosts
scp /etc/hosts root@node3:/etc/hosts
```

Execute the prechecks tool to ensure all requisites are ok.

```text
[root@kolla-deployment-node kolla]# kolla-ansible prechecks -i ansible/inventory/multinode 
Pre-deployment checking : ansible-playbook -i ansible/inventory/multinode -e @/etc/kolla/globals.yml -e @/etc/kolla/passwords.yml -e CONFIG_DIR=/etc/kolla  /usr/share/kolla/ansible/prechecks.yml 

PLAY [all] ******************************************************************** 

GATHERING FACTS *************************************************************** 
ok: [192.168.1.77]
ok: [192.168.1.74]
ok: [192.168.1.78]
.......................
PLAY RECAP ******************************************************************** 
192.168.1.74               : ok=63   changed=0    unreachable=0    failed=0   
192.168.1.77               : ok=63   changed=0    unreachable=0    failed=0   
192.168.1.78               : ok=63   changed=0    unreachable=0    failed=0   
```

\| Once all requistes are passed, start the installation of OpenStack by Kolla. \| The first time usually take a long time, because docker images need to be pulled into target hosts, and more if pull comes from DockerHub registry instead of a local one.

```text
[root@kolla-deployment-node kolla]# kolla-ansible deploy -i ansible/inventory/multinode
Deploying Playbooks : ansible-playbook -i ansible/inventory/multinode -e @/etc/kolla/globals.yml -e @/etc/kolla/passwords.yml -e CONFIG_DIR=/etc/kolla  -e action=deploy /usr/share/kolla/ansible/site.yml 

PLAY [ceph-mon;ceph-osd;ceph-rgw] ********************************************* 

GATHERING FACTS *************************************************************** 
ok: [192.168.1.77]
ok: [192.168.1.74]
ok: [192.168.1.78]

TASK: [common | Ensuring config directories exist] **************************** 
skipping: [192.168.1.77] => (item=heka)
skipping: [192.168.1.74] => (item=heka)
skipping: [192.168.1.77] => (item=cron)
skipping: [192.168.1.78] => (item=heka)
skipping: [192.168.1.74] => (item=cron)
skipping: [192.168.1.77] => (item=cron/logrotate)
skipping: [192.168.1.74] => (item=cron/logrotate)
skipping: [192.168.1.78] => (item=cron)
skipping: [192.168.1.78] => (item=cron/logrotate)

.......................

PLAY RECAP ******************************************************************** 
192.168.1.74               : ok=301  changed=93   unreachable=0    failed=0   
192.168.1.77               : ok=301  changed=93   unreachable=0    failed=0   
192.168.1.78               : ok=301  changed=93   unreachable=0    failed=0   
```

Execute this tool to create a credential file.

```text
[root@kolla-deployment-node kolla]# kolla-ansible post-deploy

Post-Deploying Playbooks : ansible-playbook -i /usr/share/kolla/ansible/inventory/all-in-one -e @/etc/kolla/globals.yml -e @/etc/kolla/passwords.yml -e CONFIG_DIR=/etc/kolla  /usr/share/kolla/ansible/post-deploy.yml 

PLAY [Creating admin openrc file on the deploy node] ************************** 

GATHERING FACTS *************************************************************** 
ok: [localhost]

TASK: [template ] ************************************************************* 
changed: [localhost]

PLAY RECAP ******************************************************************** 
localhost                  : ok=2    changed=1    unreachable=0    failed=0   
```

Source credential file.

```text
[root@kolla-deployment-node kolla]# source /etc/kolla/admin-openrc.sh
```

\| Kolla ships a tool to create a base Openstack configuration layout, this will create networks, routers, images, etc. \| Execute it in the newly OpenStack environment.

```text
[root@kolla-deployment-node kolla]# tools/init-runonce
Downloading glance image.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 12.6M  100 12.6M    0     0   873k      0  0:00:14  0:00:14 --:--:-- 1823k
Creating glance image.
[=============================>] 100%
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
| container_format | bare                                 |
| created_at       | 2016-04-15T19:41:20.000000           |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | 0b5ec320-ace9-4b34-93cb-54fa6f2c70f5 |
| is_public        | False                                |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros                               |
| owner            | a9c2e6c6a55b40619d4f12f05aea03f1     |
| protected        | False                                |
| size             | 13287936                             |
| status           | active                               |
| updated_at       | 2016-04-15T19:42:35.000000           |
| virtual_size     | None                                 |
+------------------+--------------------------------------+
Configuring neutron.
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2016-04-15T19:43:07                  |
| description               |                                      |
| id                        | 12c74cdb-9218-4d8b-ab24-d5bc7f17d8c5 |
| ipv4_address_scope        |                                      |
| ipv6_address_scope        |                                      |
| is_default                | False                                |
| mtu                       | 1500                                 |
| name                      | public1                              |
| provider:network_type     | flat                                 |
| provider:physical_network | physnet1                             |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| tenant_id                 | a9c2e6c6a55b40619d4f12f05aea03f1     |
| updated_at                | 2016-04-15T19:43:07                  |
+---------------------------+--------------------------------------+
Created a new subnet:
+-------------------+----------------------------------------------+
| Field             | Value                                        |
+-------------------+----------------------------------------------+
| allocation_pools  | {"start": "10.0.2.150", "end": "10.0.2.199"} |
| cidr              | 10.0.2.0/24                                  |
| created_at        | 2016-04-15T19:43:47                          |
| description       |                                              |
| dns_nameservers   |                                              |
| enable_dhcp       | False                                        |
| gateway_ip        | 10.0.2.1                                     |
| host_routes       |                                              |
| id                | 274bee58-68bb-4a96-bae5-41c03022a363         |
| ip_version        | 4                                            |
| ipv6_address_mode |                                              |
| ipv6_ra_mode      |                                              |
| name              | 1-subnet                                     |
| network_id        | 12c74cdb-9218-4d8b-ab24-d5bc7f17d8c5         |
| subnetpool_id     |                                              |
| tenant_id         | a9c2e6c6a55b40619d4f12f05aea03f1             |
| updated_at        | 2016-04-15T19:43:47                          |
+-------------------+----------------------------------------------+
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2016-04-15T19:44:42                  |
| description               |                                      |
| id                        | 9bb7cca0-e7ea-4601-8770-7296473bdfff |
| ipv4_address_scope        |                                      |
| ipv6_address_scope        |                                      |
| mtu                       | 1450                                 |
| name                      | demo-net                             |
| provider:network_type     | vxlan                                |
| provider:physical_network |                                      |
| provider:segmentation_id  | 94                                   |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| tenant_id                 | a9c2e6c6a55b40619d4f12f05aea03f1     |
| updated_at                | 2016-04-15T19:44:43                  |
+---------------------------+--------------------------------------+
Created a new subnet:
+-------------------+--------------------------------------------+
| Field             | Value                                      |
+-------------------+--------------------------------------------+
| allocation_pools  | {"start": "10.0.0.2", "end": "10.0.0.254"} |
| cidr              | 10.0.0.0/24                                |
| created_at        | 2016-04-15T19:45:25                        |
| description       |                                            |
| dns_nameservers   | 8.8.8.8                                    |
| enable_dhcp       | True                                       |
| gateway_ip        | 10.0.0.1                                   |
| host_routes       |                                            |
| id                | 28ef0e39-33a4-43ea-b1a6-8ea01d7c3379       |
| ip_version        | 4                                          |
| ipv6_address_mode |                                            |
| ipv6_ra_mode      |                                            |
| name              | demo-subnet                                |
| network_id        | 9bb7cca0-e7ea-4601-8770-7296473bdfff       |
| subnetpool_id     |                                            |
| tenant_id         | a9c2e6c6a55b40619d4f12f05aea03f1           |
| updated_at        | 2016-04-15T19:45:25                        |
+-------------------+--------------------------------------------+
Created a new router:
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | True                                 |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| description             |                                      |
| distributed             | False                                |
| external_gateway_info   |                                      |
| ha                      | False                                |
| id                      | 53a09f8a-576a-4f83-82b0-995a26f83deb |
| name                    | demo-router                          |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tenant_id               | a9c2e6c6a55b40619d4f12f05aea03f1     |
+-------------------------+--------------------------------------+
Added interface ed81ba4c-0e51-4cd9-9810-0a9b883102c2 to router demo-router.
Set gateway for router demo-router
Created a new security_group_rule:
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| id                | 4f836611-830d-48e7-a81c-7aa65a2573a4 |
| port_range_max    |                                      |
| port_range_min    |                                      |
| protocol          | icmp                                 |
| remote_group_id   |                                      |
| remote_ip_prefix  | 0.0.0.0/0                            |
| security_group_id | c9e76d1f-d58c-4621-b402-1295d9e5168d |
| tenant_id         | a9c2e6c6a55b40619d4f12f05aea03f1     |
+-------------------+--------------------------------------+
Created a new security_group_rule:
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| id                | 8cb6c081-0388-4d94-98f8-58190c574133 |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| protocol          | tcp                                  |
| remote_group_id   |                                      |
| remote_ip_prefix  | 0.0.0.0/0                            |
| security_group_id | c9e76d1f-d58c-4621-b402-1295d9e5168d |
| tenant_id         | a9c2e6c6a55b40619d4f12f05aea03f1     |
+-------------------+--------------------------------------+
Created a new security_group_rule:
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| id                | 76142824-3cb2-43a5-bbd7-635aedd05666 |
| port_range_max    | 8000                                 |
| port_range_min    | 8000                                 |
| protocol          | tcp                                  |
| remote_group_id   |                                      |
| remote_ip_prefix  | 0.0.0.0/0                            |
| security_group_id | c9e76d1f-d58c-4621-b402-1295d9e5168d |
| tenant_id         | a9c2e6c6a55b40619d4f12f05aea03f1     |
+-------------------+--------------------------------------+
Created a new security_group_rule:
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| id                | ce77b36f-a9ed-4c10-ba1f-2697ad1c8138 |
| port_range_max    | 8080                                 |
| port_range_min    | 8080                                 |
| protocol          | tcp                                  |
| remote_group_id   |                                      |
| remote_ip_prefix  | 0.0.0.0/0                            |
| security_group_id | c9e76d1f-d58c-4621-b402-1295d9e5168d |
| tenant_id         | a9c2e6c6a55b40619d4f12f05aea03f1     |
+-------------------+--------------------------------------+
Configuring nova public key and quotas.
```

Check nova services status

```text
[egonzalez@localhost kolla]$ nova service-list
+----+------------------+-------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host  | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+-------+----------+---------+-------+----------------------------+-----------------+
| 40 | nova-consoleauth | node3 | internal | enabled | up    | 2016-04-15T20:15:44.000000 | -               |
| 43 | nova-consoleauth | node1 | internal | enabled | up    | 2016-04-15T20:15:46.000000 | -               |
| 46 | nova-consoleauth | node2 | internal | enabled | up    | 2016-04-15T20:15:48.000000 | -               |
| 49 | nova-scheduler   | node3 | internal | enabled | up    | 2016-04-15T20:15:50.000000 | -               |
| 52 | nova-scheduler   | node2 | internal | enabled | up    | 2016-04-15T20:15:42.000000 | -               |
| 55 | nova-scheduler   | node1 | internal | enabled | up    | 2016-04-15T20:15:43.000000 | -               |
| 58 | nova-conductor   | node1 | internal | enabled | up    | 2016-04-15T20:15:36.000000 | -               |
| 64 | nova-conductor   | node2 | internal | enabled | up    | 2016-04-15T20:15:37.000000 | -               |
| 70 | nova-conductor   | node3 | internal | enabled | up    | 2016-04-15T20:15:35.000000 | -               |
| 79 | nova-compute     | node3 | nova     | enabled | up    | 2016-04-15T20:15:43.000000 | -               |
| 85 | nova-compute     | node2 | nova     | enabled | up    | 2016-04-15T20:15:50.000000 | -               |
| 88 | nova-compute     | node1 | nova     | enabled | up    | 2016-04-15T20:15:51.000000 | -               |
+----+------------------+-------+----------+---------+-------+----------------------------+-----------------+
```

Check Neutron agents status.

```text
[egonzalez@localhost kolla]$ neutron agent-list
+--------------------------------------+--------------------+-------+-------+----------------+---------------------------+
| id                                   | agent_type         | host  | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-------+-------+----------------+---------------------------+
| 08d12ccd-74cd-4e8e-9cda-3d3d2e191191 | Metadata agent     | node3 | :-)   | True           | neutron-metadata-agent    |
| 0916aa0e-6d07-4398-99a5-e0e9123cef37 | DHCP agent         | node1 | :-)   | True           | neutron-dhcp-agent        |
| 14707eaf-2d37-4eaf-964a-82b63d1bdc96 | Open vSwitch agent | node3 | :-)   | True           | neutron-openvswitch-agent |
| 265a0acc-e31a-4098-842a-b139e8095056 | L3 agent           | node2 | :-)   | True           | neutron-l3-agent          |
| 50869311-b3bb-4fb3-9676-d1f56d77deb0 | Metadata agent     | node2 | :-)   | True           | neutron-metadata-agent    |
| 5c48b20a-1b57-4e3b-865a-f0f298ea0af8 | DHCP agent         | node2 | :-)   | True           | neutron-dhcp-agent        |
| 89470cc7-6430-45a2-8ee2-852e0ba85cff | Open vSwitch agent | node2 | :-)   | True           | neutron-openvswitch-agent |
| ba689300-c49a-46a7-8c85-e7a6daa5f2cb | DHCP agent         | node3 | :-)   | True           | neutron-dhcp-agent        |
| baadfe87-db69-491b-b7ad-7f16c1468632 | Metadata agent     | node1 | :-)   | True           | neutron-metadata-agent    |
| bc823fff-11a3-4f81-90d5-8f9e4a7a617a | L3 agent           | node3 | :-)   | True           | neutron-l3-agent          |
| d26c860d-e5e3-4da0-b0af-f8ad3a69e9f6 | L3 agent           | node1 | :-)   | True           | neutron-l3-agent          |
| e90277e7-3e46-42d0-a2fd-dce412f503dd | Open vSwitch agent | node1 | :-)   | True           | neutron-openvswitch-agent |
+--------------------------------------+--------------------+-------+-------+----------------+---------------------------+
```

Create a new instance and see what happens.

```text
[egonzalez@localhost kolla]$ openstack server create --image cirros --flavor m1.tiny --nic net-id=demo-net demo-instance
```

Check how the instance is going.

```text
[egonzalez@localhost kolla]$ openstack server list
+--------------------------------------+---------------+--------+-------------------+
| ID                                   | Name          | Status | Networks          |
+--------------------------------------+---------------+--------+-------------------+
| b234e514-2975-47fd-a618-8ef6aa9ff2bc | demo-instance | ACTIVE | demo-net=10.0.0.3 |
+--------------------------------------+---------------+--------+-------------------+
```

Thats all for now, in future posts we will see in more detail how Kolla works.

Cheers, Eduardo Gonzalez

