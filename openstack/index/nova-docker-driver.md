# Nova Docker driver

Cloud computing has evolved too fast over the last years, currently is a totally different thing as the 5 years ago cloud, today is a common thing listening words like containers, instances, microservices, queue messages on linkedin, twitter, etc.

OpenStack is not a lazy community, new capabilities are daily added to the OpenStack catalog reaching more users and business needs who are discovered at the several summits and meetups over the world. One of that needs is the capability to easy create and manage docker containers.&#x20;

Now we have two main methods, directly launching instances as containers from nova driver or with heat/kubernetes/messos.&#x20;

The second method is the one with more followers, but there are some projects which are using nova driver as Solum, for this reason I'm going to show you how to configure docker as nova driver.

The fist step is install docker on the compute nodes

```
curl -sSL https://get.docker.com/ | sh

+ sh -c 'sleep 3; yum -y -q install docker-engine'
advertencia:/var/cache/yum/x86_64/7/docker-main-repo/packages/docker-engine-selinux-1.9.1-1.el7.centos.noarch.rpm: EncabezadoV4 RSA/SHA512 Signature, ID de clave 2c52609d: NOKEY
No se ha instalado la llave pública de docker-engine-selinux-1.9.1-1.el7.centos.noarch.rpm
Importando llave GPG 0x2C52609D:
Usuarioid  : "Docker Release Tool (releasedocker) <docker@docker.com>"
Huella       : 5811 8e89 f3a9 1289 7c07 0adb f762 2157 2c52 609d
Desde      : https://yum.dockerproject.org/gpg
Full path required for exclude: net:[4026532228].
Full path required for exclude: net:[4026532228].
Full path required for exclude: net:[4026532285].
Full path required for exclude: net:[4026532285].
Full path required for exclude: net:[4026532228].
Full path required for exclude: net:[4026532228].
Full path required for exclude: net:[4026532285].
Full path required for exclude: net:[4026532285].
```

Add nova user to docker group, docker group will be created during docker installation

```
usermod -aG docker nova
```

Start docker service

```
sudo systemctl start docker
```

Test docker installation with the following command, a Hello from Docker message should be prompted

```
sudo docker run hello-world

Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
b901d36b6f2f: Pull complete
0a6ba66e537a: Pull complete
Digest: sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7
Status: Downloaded newer image for hello-world:latest

Hello from Docker.
This message shows that your installation appears to be working correctly.
```

Once docker runs in a proper way, enable docker service at boot

```
sudo systemctl enable docker
ln -s '/usr/lib/systemd/system/docker.service' '/etc/systemd/system/multi-user.target.wants/docker.service'
```

Give docker socket the apropiate permissions

```
chmod 666  /var/run/docker.sock
```

Restart nova-compute service

```
systemctl restart openstack-nova-compute
```

Install git and pip if not present on the system

```
sudo yum install -y git
sudo easy_install pip
```

Clone docker driver for nova from OpenStack repositories

```
git clone -b stable/liberty https://github.com/openstack/nova-docker
```

Install basic requirements

```
cd nova-docker
sudo  pip install -r requirements.txt
```

Install docker driver

```
python setup.py install
```

Edit nova.conf and allow docker driver as compute driver

```
vi /etc/nova/nova.conf
compute_driver=novadocker.virt.docker.DockerDriver
```

Create the following directory

```
mkdir /etc/nova/rootwrap.d
```

Create a file with the following content to allow setting networking in docker containers

```
vi /etc/nova/rootwrap.d/docker.filters

[Filters]
# nova/virt/docker/driver.py: 'ln', '-sf', '/var/run/netns/.*'
ln: CommandFilter, /bin/ln, root
```

Edit glance-api.conf and allow docker as container format

```
vi /etc/glance/glance-api.conf
container_formats=ami,ari,aki,bare,ovf,ova,docker
```

Restart glance-api to apply changes

```
systemctl restart openstack-glance-api
```

Pull a docker image, i use hipache as testing image

```
docker pull hipache

Using default tag: latest
latest: Pulling from library/hipache
0a85502c06c9: Pull complete
0998bf8fb9e9: Pull complete
a6785352b25c: Pull complete
e9ae3c220b23: Pull complete
84d61e35041c: Pull complete
0fd25fcc737a: Pull complete
c0af65e2f918: Pull complete
dc335e9e58f4: Pull complete
7245129ed8a4: Pull complete
52a015bc8761: Pull complete
d38065541924: Pull complete
0b8658d6c429: Pull complete
188468e0ae8d: Pull complete
741abf992884: Pull complete
Digest: sha256:7774cf9155a8cc83b6964c7ea0d655143c152debc6d11d4f6dfa918c7a7ea099
Status: Downloaded newer image for hipache:latest
```

Upload the image to glance

```
docker save hipache | openstack image create hipache --public --container-format docker --disk-format raw

+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | e93b7c1ddeb2d38419bf44aaf07cc811                     |
| container_format | docker                                               |
| created_at       | 2015-12-18T10:06:31Z                                 |
| disk_format      | raw                                                  |
| file             | /v2/images/7f05f7d6-88af-4d0f-adad-66ca025404fa/file |
| id               | 7f05f7d6-88af-4d0f-adad-66ca025404fa                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | hipache                                              |
| owner            | 74675bfffc3c4e1a9d9fb2f1388217d4                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 384304640                                            |
| status           | active                                               |
| updated_at       | 2015-12-18T10:07:03Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
```

Once the image is active at glance, create a new instance, the instance won\\'t be a KVM virtual machine, now will be a docker container

```
nova boot --flavor m1.tiny --image hipache --nic net-id=a1aa6336-9ae2-4ffb-99f5-1b6d1130989c --key-name mykey test1
```

After a while, the instance should be in ACTIVE state

```
watch nova list
+--------------------------------------+-------+--------+------------+-------------+-----------------------------+
| ID                                   | Name  | Status | Task State | Power State | Networks                    |
+--------------------------------------+-------+--------+------------+-------------+-----------------------------+
| fb192405-4150-4c2d-98ad-316141f48cc5 | test1 | ACTIVE | -          | Running     | private_network=192.168.1.3 |
+--------------------------------------+-------+--------+------------+-------------+-----------------------------+
```

If all the steps worked fine, you can use docker as nova backend.&#x20;

Regards
