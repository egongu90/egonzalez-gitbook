---
description: Manual installation from source code
---

# Magnum in RDO OpenStack Liberty

 Want to install Magnum \(Containers as a Service\) in an OpenStack environment based on packages from RDO project? Here are the steps to do it:

\|Primary steps are the same as official Magnum guide, major differences come from DevStack or manual installations vs packages from RDO project. 

Also, some of the steps are explained to show how Magnum should work, as well this guide can help you understand Magnum integration with your current environment. 

I\'m not going to use Barbican service for certs management, you will see how to use Magnum without Barbican too.

* For now, there is not RDO packages for magnum, so we are going to

  install it from source code.

* As i know, currently magnum packages are under development and will

  be added in future OpenStack versions to RDO project packages.

  \(Probably Mitaka or Newton\)

Passwords used at this demo are:

* temporal \(Databases and OpenStack users\)
* guest \(RabbitMQ\)

IPs used are:

* 192.168.200.208 \(Service APIs\)
* 192.168.100.0/24 \(External network range\)
* 10.0.0.0/24 \(Tenant network range\)
* 8.8.8.8 \(Google DNS server\)

First we need to install some dependencies and packages needed for next steps.

```text
sudo yum install -y gcc python-setuptools python-devel git libffi-devel openssl-devel wget
```

Install pip

```text
easy_install pip
```

Clone Magnum source code from OpenStack git repository, ensure you use Liberty branch, if not, Magnum dependencies will break all OpenStack services dependencies and lost your current environment \(Trust me, i\'m talking from my own experience\)

```text
git clone https://git.openstack.org/openstack/magnum -b stable/liberty
```

Move to your newly created folder and install Magnum \(dependency requirements and Magnum\)

```text
cd magnum
sudo pip install -e .
```

Once Magnum is installed, create Magnum database and Magnum user

```text
mysql -uroot -p
CREATE DATABASE IF NOT EXISTS magnum DEFAULT CHARACTER SET utf8;
GRANT ALL PRIVILEGES ON magnum.* TO'magnum'@'localhost' IDENTIFIED BY 'temporal';
GRANT ALL PRIVILEGES ON magnum.* TO'magnum'@'%' IDENTIFIED BY 'temporal';
```

Create Magnum folder and copy sample configuration files.

```text
mkdir /etc/magnum
sudo cp etc/magnum/magnum.conf.sample /etc/magnum/magnum.conf
sudo cp etc/magnum/policy.json /etc/magnum/policy.json
```

Edit Magnum main configuration file

```text
vi /etc/magnum/magnum.conf
```

Configure messaging backend to RabbitMQ

```text
[DEFAULT]

rpc_backend = rabbit
notification_driver = messaging
```

Bind Magnum API port to listen on all the interfaces, you can also especify on which IP Magnum API will be listening if you are concerned about security risks.

```text
[api]

host = 0.0.0.0
```

Configure RabbitMQ backend

```text
[oslo_messaging_rabbit]

rabbit_host = 192.168.200.208
rabbit_userid = guest
rabbit_password = guest
rabbit_virtual_host = /
```

Set database connection

```text
[database]

connection=mysql://magnum:temporal@192.168.200.208/magnum
```

Set cert\_manager\_type to local, this option will disable Barbican service, you will need to create a folder \(We will do it in next steps\)

```text
[certificates]

cert_manager_type = local
```

As all OpenStack services, Keystone authentication is required.

* Check what your service tenant name it is \(RDO default name is

  \"services\" other installations usually use \"service\" name.

```text
[keystone_authtoken]

auth_uri=http://192.168.200.208:5000/v2.0
identity_uri=http://192.168.200.208:35357
auth_strategy=keystone
admin_user=magnum
admin_password=temporal
admin_tenant_name=services
```

As we saw before, create local certificates folder to avoid using Barbican service. This is the step we previously commented

```text
mkdir -p /var/lib/magnum/certificates/
```

Clone python-magnumclient and install it, this package will provide us commands to use Magnum

```text
git clone https://git.openstack.org/openstack/python-magnumclient -b stable/liberty
cd python-magnumclient
sudo pip install -e .
```

Create Magnum user at keystone

```text
openstack user create --password temporal magnum
```

Add admin role to Magnum user at tenant services

```text
openstack role add --project services --user magnum admin
```

Create container service

```text
openstack service create --name magnum --description "Magnum Container Service" container
```

Finally create Magnum endpoints

```text
openstack endpoint create --region RegionOne --publicurl 'http://192.168.200.208:9511/v1' --adminurl 'http://192.168.200.208:9511/v1' --internalurl 'http://192.168.200.208:9511/v1' magnum
```

Sync Magnum database, this step will create Magnum tables at the database

```text
magnum-db-manage --config-file /etc/magnum/magnum.conf upgrade
```

Open two terminal session and execute one command on each terminal to start both services. If you encounter any issue, logs can be found at these terminal

```text
magnum-api --config-file /etc/magnum/magnum.conf
magnum-conductor --config-file /etc/magnum/magnum.conf
```

Check if Magnum service is fine

```text
magnum service-list
+----+------------+------------------+-------+
| id | host       | binary           | state |
+----+------------+------------------+-------+
| 1  | controller | magnum-conductor | up    |
+----+------------+------------------+-------+
```

Download fedora atomic image

```text
wget https://fedorapeople.org/groups/magnum/fedora-21-atomic-5.qcow2
```

Create a Glance image with Atomic.qcow2 file

```text
glance image-create --name fedora-21-atomic-5 \
                    --visibility public \
                    --disk-format qcow2 \
                    --os-distro fedora-atomic \
                    --container-format bare < fedora-21-atomic-5.qcow2
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | cebefc0c21fb8567e662bf9f2d5b78b0     |
| container_format | bare                                 |
| created_at       | 2016-03-19T15:55:21Z                 |
| disk_format      | qcow2                                |
| id               | 7293891d-cfba-48a9-a4db-72c29c65f681 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | fedora-21-atomic-5                   |
| os_distro        | fedora-atomic                        |
| owner            | e3cca42ed57745148e0c342a000d99e9     |
| protected        | False                                |
| size             | 891355136                            |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2016-03-19T15:55:28Z                 |
| virtual_size     | None                                 |
| visibility       | public                               |
+------------------+--------------------------------------+
```

Create a ssh key if not exists, this command won\'t create a new ssh key if already exists

```text
test -f ~/.ssh/id_rsa.pub || ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
```

Add the key to nova, mine is called egonzalez

```text
nova keypair-add --pub-key ~/.ssh/id_rsa.pub egonzalez
```

\| Now we are going to test our new Magnum service, you have various methods to do it. \| I will use Docker Swarm method because is the simplest one for this demo purposes. Go through Magnum documentation to check other container methods as Kubernetes is.

Create a baymodel with atomic image and swarm, select a flavor with at least 10GB of disk

```text
magnum baymodel-create --name demoswarmbaymodel \
                       --image-id fedora-21-atomic-5 \
                       --keypair-id egonzalez \
                       --external-network-id public \
                       --dns-nameserver 8.8.8.8 \
                       --flavor-id testflavor \
                       --docker-volume-size 1 \
                       --coe swarm
+---------------------+--------------------------------------+
| Property            | Value                                |
+---------------------+--------------------------------------+
| http_proxy          | None                                 |
| updated_at          | None                                 |
| master_flavor_id    | None                                 |
| fixed_network       | None                                 |
| uuid                | 887edbc7-0805-4796-be78-dfcddad8eb03 |
| no_proxy            | None                                 |
| https_proxy         | None                                 |
| tls_disabled        | False                                |
| keypair_id          | egonzalez                            |
| public              | False                                |
| labels              | {}                                   |
| docker_volume_size  | 1                                    |
| external_network_id | public                               |
| cluster_distro      | fedora-atomic                        |
| image_id            | fedora-21-atomic-5                   |
| registry_enabled    | False                                |
| apiserver_port      | None                                 |
| name                | demoswarmbaymodel                    |
| created_at          | 2016-03-19T17:22:43+00:00            |
| network_driver      | None                                 |
| ssh_authorized_key  | None                                 |
| coe                 | swarm                                |
| flavor_id           | testflavor                           |
| dns_nameserver      | 8.8.8.8                              |
+---------------------+--------------------------------------+
```

Create a bay with the previous bay model, we are going to create one master node and one worker, specify all that apply to your environment

```text
magnum bay-create --name demoswarmbay --baymodel demoswarmbaymodel --master-count 1 --node-count 1
+--------------------+--------------------------------------+
| Property           | Value                                |
+--------------------+--------------------------------------+
| status             | None                                 |
| uuid               | a2388916-db30-41bf-84eb-df0b65979eaf |
| status_reason      | None                                 |
| created_at         | 2016-03-19T17:23:00+00:00            |
| updated_at         | None                                 |
| bay_create_timeout | 0                                    |
| api_address        | None                                 |
| baymodel_id        | 887edbc7-0805-4796-be78-dfcddad8eb03 |
| node_count         | 1                                    |
| node_addresses     | None                                 |
| master_count       | 1                                    |
| discovery_url      | None                                 |
| name               | demoswarmbay                         |
+--------------------+--------------------------------------+
```

Check bay status, for now it should be in CREATE\_IN\_PROGRESS state

```text
magnum bay-show demoswarmbay
+--------------------+--------------------------------------+
| Property           | Value                                |
+--------------------+--------------------------------------+
| status             | CREATE_IN_PROGRESS                   |
| uuid               | a2388916-db30-41bf-84eb-df0b65979eaf |
| status_reason      |                                      |
| created_at         | 2016-03-19T17:23:00+00:00            |
| updated_at         | 2016-03-19T17:23:01+00:00            |
| bay_create_timeout | 0                                    |
| api_address        | None                                 |
| baymodel_id        | 887edbc7-0805-4796-be78-dfcddad8eb03 |
| node_count         | 1                                    |
| node_addresses     | []                                   |
| master_count       | 1                                    |
| discovery_url      | None                                 |
| name               | demoswarmbay                         |
+--------------------+--------------------------------------+
```

If all is going fine, nova should have two new instances\(in ACTIVE state\), one for the master node and second for the worker.

```text
nova list
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-------------------------------------------------------------------------------+
| ID                                   | Name                                                  | Status | Task State | Power State | Networks                                                                      |
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-------------------------------------------------------------------------------+
| e38eb88c-bb6b-427d-a2c5-cdfe868796f0 | de-44kx2l4q4wc-0-d6j5svvjxmne-swarm_node-xafkm2jskf5j | ACTIVE | -          | Running     | demoswarmbay-agf6y3qnjoyw-fixed_network-g37bcmc52akv=10.0.0.4, 192.168.100.16 |
| 5acc579d-152a-4656-9eb8-e800b7ab3bcf | demoswarmbay-agf6y3qnjoyw-swarm_master-fllwhrpuabbq   | ACTIVE | -          | Running     | demoswarmbay-agf6y3qnjoyw-fixed_network-g37bcmc52akv=10.0.0.3, 192.168.100.15 |
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-------------------------------------------------------------------------------+
```

You can see how heat stack is going

```text
heat stack-list
+--------------------------------------+---------------------------+--------------------+---------------------+--------------+
| id                                   | stack_name                | stack_status       | creation_time       | updated_time |
+--------------------------------------+---------------------------+--------------------+---------------------+--------------+
| 3a64fa60-4df8-498f-aceb-a0cb8cfc0b18 | demoswarmbay-agf6y3qnjoyw | CREATE_IN_PROGRESS | 2016-03-19T17:22:59 | None         |
+--------------------------------------+---------------------------+--------------------+---------------------+--------------+
```

We can see what tasks are executing during stack creation

```text
heat event-list demoswarmbay-agf6y3qnjoyw
+-------------------------------------+--------------------------------------+------------------------+--------------------+---------------------+
| resource_name                       | id                                   | resource_status_reason | resource_status    | event_time          |
+-------------------------------------+--------------------------------------+------------------------+--------------------+---------------------+
| demoswarmbay-agf6y3qnjoyw           | 004c9388-b8ab-4541-ada8-99b65203e41d | Stack CREATE started   | CREATE_IN_PROGRESS | 2016-03-19T17:23:01 |
| master_wait_handle                  | d6f0798a-bfde-4bad-9c73-e108bd101009 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:01 |
| secgroup_manager                    | e2e0eb08-aeeb-4290-9ad5-bd20fe243f07 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:02 |
| disable_selinux                     | d7290592-ab81-4d7a-b2fa-902975904a25 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:03 |
| agent_wait_handle                   | 65ec5553-56a4-4416-9748-bfa0ae35737a | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:03 |
| add_proxy                           | 46bdcff8-4606-406f-8c99-7f48adc4de57 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:03 |
| write_docker_socket                 | ab5402ea-44af-4433-84aa-a63256817a9a | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:04 |
| make_cert                           | 3b9817a5-606f-41ab-8799-b411c017f05d | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:04 |
| cfn_signal                          | 0add665a-3fdf-4408-ab15-76332aa326fe | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:04 |
| remove_docker_key                   | 94f4106e-f139-4d9f-9974-8821d04be103 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:05 |
| configure_swarm                     | f7e0ebd5-1893-43d1-bd29-81a7e39de0c0 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:05 |
| extrouter                           | a94a8f68-c237-4dbc-9513-cdbe3de1465e | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:05 |
| enable_services                     | c250f532-99bd-43d7-9d15-b2d3ae16567a | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:06 |
| write_docker_service                | 2c9d8954-4446-4578-a871-0910e8996571 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:06 |
| cloud_init_wait_handle              | 6cc51d2d-56e9-458b-a21b-bc553e0c8291 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:06 |
| fixed_network                       | 3125395f-c689-4481-bf01-94bb2f701993 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:07 |
| agent_wait_handle                   | 2db801e8-c2b5-47b0-ac16-122dba3a22d6 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:08 |
| remove_docker_key                   | 75e2c7a6-a2ce-4026-aeeb-739c4a522f48 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:08 |
| secgroup_manager                    | ac51a029-26c1-495a-bc13-232cfb8c1060 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:08 |
| write_docker_socket                 | 58e08b52-a12a-43e9-b41d-071750294024 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:08 |
| master_wait_handle                  | 3e741b76-6470-47d4-b13e-3f8f446be53c | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:08 |
| cfn_signal                          | 96c26b4f-1e99-478e-a8e5-9dcc4486e1b3 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:08 |
| enable_services                     | beedc358-ee72-4b34-a6b9-1b47ffc15306 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:08 |
| add_proxy                           | caae3a07-d5f1-4eb0-8a82-02ea634f77ae | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:08 |
| make_cert                           | 79363643-e5e4-4d1b-ad8a-5a56e1f6a8e7 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:08 |
| cloud_init_wait_handle              | 0457b008-6da8-44fd-abef-cb99bd4d0518 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:09 |
| configure_swarm                     | baf1e089-c627-4b24-a571-63b3c9c14e28 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:09 |
| extrouter                           | 184614d9-2280-4cb4-9253-f538463dbdf4 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:09 |
| write_docker_service                | 80e66b4e-d40a-4243-bb27-0d2a6b68651f | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:09 |
| disable_selinux                     | d8a64822-2571-4dcf-9da5-b3ec73e771eb | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:09 |
| fixed_network                       | 528b0ced-23f6-4c22-8cbc-357ba0ee5bc5 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:09 |
| write_swarm_manager_failure_service | 9fa100a3-b4a9-465c-8b33-dd000cb4866a | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:10 |
| write_swarm_agent_failure_service   | a7c09833-929e-4711-a3e9-39923d23b2f2 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:10 |
| fixed_subnet                        | 23d8b0a6-a7a3-4f71-9c18-ba6255cf071a | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:10 |
| write_swarm_master_service          | d24a6099-3cad-41ce-8d4b-a7ad0661aaea | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:11 |
| fixed_subnet                        | 1a2b7397-1d09-4544-bb9f-985c2f64cb09 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:13 |
| write_swarm_manager_failure_service | 615a2a7a-5266-487b-bbe1-fcaa82f43243 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:13 |
| write_swarm_agent_failure_service   | 3f8c54b4-6644-49a0-ad98-9bc6b4332a07 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:13 |
| write_swarm_master_service          | 2f58b3c8-d1cc-4590-a328-0e775e495bcf | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:13 |
| extrouter_inside                    | f3da7f2f-643e-4f29-a00f-d2595d7faeaf | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:14 |
| swarm_master_eth0                   | 1d6a510d-520c-4796-8990-aa8f7dd59757 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:16 |
| swarm_master_eth0                   | 3fd85913-7399-49be-bb46-5085ff953611 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:19 |
| extrouter_inside                    | 33749e30-cbea-4093-b36a-94967e299002 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:19 |
| write_heat_params                   | 054e0af5-e3e0-4bc0-92b5-b40aeedc39ab | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:19 |
| swarm_nodes                         | df7af58c-8148-4b51-bd65-b0734d9051b5 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:20 |
| write_swarm_agent_service           | ab1e8b1e-2837-4693-b791-e1311f85fa63 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:21 |
| swarm_master_floating               | d99ffe66-cb02-4279-99dc-a1f3e2ca817c | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:22 |
| write_heat_params                   | 33d9999f-6c93-453d-8565-ac99db021f8f | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:25 |
| write_swarm_agent_service           | 02a1b7f6-2660-4345-ad08-42b66ffaaad5 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:25 |
| swarm_master_floating               | 8ce6ecd8-c421-4e4a-ab81-cba4b5ccedf4 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:25 |
| swarm_master_init                   | 3787dcc8-e644-412b-859b-63a434b9ee6c | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:26 |
| swarm_master_init                   | a1dd67bb-49c7-4507-8af0-7758b76b57e1 | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:28 |
| swarm_master                        | d12b915e-3087-4e17-9954-8233926b504b | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:29 |
| swarm_master                        | a34ad52a-def7-460b-b5b7-410000207b3e | state changed          | CREATE_COMPLETE    | 2016-03-19T17:23:48 |
| master_wait_condition               | 0c9331a4-8ad0-46e0-bf2a-35943021a1a3 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:49 |
| cloud_init_wait_condition           | de3707a0-f46a-44a9-b4b8-ff50e12cc77f | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:49 |
| agent_wait_condition                | a1a810a4-9c19-4983-aaa8-e03f308c1e39 | state changed          | CREATE_IN_PROGRESS | 2016-03-19T17:23:49 |
+-------------------------------------+--------------------------------------+------------------------+--------------------+---------------------+
```

Once all tasks are completed, we can create containers in the bay we created in previous steps.

```text
magnum container-create --name demo-container \
                        --image docker.io/cirros:latest \
                        --bay demoswarmbay \
                        --command "ping -c 4 192.168.100.2"
+------------+----------------------------------------+
| Property   | Value                                  |
+------------+----------------------------------------+
| uuid       | 36595858-8657-d465-3e5a-dfcddad8a238   |
| links      | ...                                    |
| bay_uuid   | a2388916-db30-41bf-84eb-df0b65979eaf   |
| updated_at | None                                   |
| image      | cirros                                 |
| command    | ping -c 4 192.168.100.2                |
| created_at | 2016-03-19T17:30:00+00:00              |
| name       | demo-container                         |
+------------+----------------------------------------+
```

\| Container is created, but not started. \| Start the container

```text
magnum container-start demo-container
```

Check container logs, you should see 4 pings succeed to our external router gateway.

```text
magnum container-logs demo-container

PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
64 bytes from 192.168.100.2: icmp_seq=1 ttl=64 time=0.083 ms
64 bytes from 192.168.100.2: icmp_seq=2 ttl=64 time=0.068 ms
64 bytes from 192.168.100.2: icmp_seq=3 ttl=64 time=0.043 ms
64 bytes from 192.168.100.2: icmp_seq=4 ttl=64 time=0.099 ms
```

You can delete the container

```text
magnum container-delete demo-container
```

While doing this demo, i missed adding branch name while cloning Magnum source code, when i installed Magnum all package dependencies where installed from master, who was Mitaka instead of Liberty, which broke my environment.

I suffered the following issues:

Issues with packages

```text
ImportError: No module named MySQLdb
```

Was solved installing MySQL-python from pip instead of yum

```text
pip install MySQL-python
```

Issues with policies, admin privileges weren\'t recognized by Magnum api.

```text
PolicyNotAuthorized: magnum-service:get_all{{ bunch of stuff }} disallowed by policy
```

Was solved removing admin\_api rule at Magnum policy.json file

```text
vi /etc/magnum/policy.json

#    "admin_api": "rule:context_is_admin",
```

\| Unfortunately, nova was completely broken and it was not working at all, so i installed a new environment and added branch while cloning source code. \| Next issue i found was Barbican, who was not installed, i used the steps mentioned at this post to solve this issue.

Hope this guide helps you integrating Magnum Container as a Service in OpenStack.

Regards, Eduardo Gonzalez

