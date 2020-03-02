# OpenStack tacker and service function chaining sfc with kolla

In this blog post I will show how to deploy OpenStack Tacker with \| Service Function Chaining \(SFC\) with OpenStack kolla project and make a few \| verifications and tests to ensure fully NFV and SFC functionality.

Tacker and SFC is only supported in kolla during Pike release or later.

### Tacker, NFV and SFC concepts

Tacker is an OpenStack service for NFV Orchestration with a general purpose VNF Manager to deploy and operate Virtual Network Functions \(VNFs\) and Network Services on an NFV Platform. It is based on ETSI MANO Architectural \| Framework. [Tacker documentation](https://docs.openstack.org/tacker/latest/)

Network functions virtualization \(NFV\) is a network architecture concept that uses the technologies of IT virtualization to virtualize entire classes of network node functions into building blocks that may connect, or chain together, to create communication services. [ETSI NFV specs](http://www.etsi.org/technologies-clusters/technologies/nfv)

Service Function Chaining is a mechanism for overriding the basic destination based forwarding that is typical of IP networks. It is conceptually related to Policy Based Routing in physical networks but it is typically thought of as a Software Defined Networking technology. It is often used in conjunction with security functions although it may be used for a broader range of features. [ETSI SFC spec](https://tools.ietf.org/html/rfc7665)

### Kolla

Kolla is a highly opinionated deployment tool out of the box. This permits Kolla to be deployable with the simple configuration of three key/value pairs. As an operator's experience with OpenStack grows and the desire to customize OpenStack services increases, Kolla offers full capability to override every OpenStack service configuration option in the deployment. [kolla documentation](https://docs.openstack.org/kolla-ansible/latest/)

#### Requirements

Kolla depends on the following requirements to be met for a fully operational multinode OpenStack cluster with Tacker and SFC features:

* Core compute stack \(nova, neutron, glance, etc\)
* Heat
* Mistral and Redis
* Barbican
* Networking-sfc

#### Deployment

Install base kolla and dependencies following [kolla\'s quickstart guide](https://docs.openstack.org/kolla-ansible/latest/quickstart.html)

Configure `globals.yml` and enable services in requirements, optionally other services can be enabled altogether. Refer to kolla documentation for other option/values information.

```text
$ vi /etc/kolla/globals.yml

---
kolla_base_distro: "centos"
kolla_install_type: "source"
kolla_internal_vip_address: "192.168.100.10"
docker_registry: "192.168.100.1:4000"
docker_namespace: "lokolla"
network_interface: "ens9"
neutron_external_interface: "ens10"

# Tacker configuration
enable_tacker: "yes"
enable_neutron_sfc: "yes"
enable_mistral: "yes"
enable_redis: "yes"
enable_barbican: "yes"
#enable_heat: "yes" # Ensure it is not disabled
```

Configure inventory file.

```text
$ vi <inventory_file>

[control]
192.168.100.244
192.168.100.186
192.168.100.159

[network]
192.168.100.244
192.168.100.186
192.168.100.159

[compute]
192.168.100.130
192.168.100.131
192.168.100.132

[monitoring]
192.168.100.244
192.168.100.186
192.168.100.159

[storage]
192.168.100.244
192.168.100.186
192.168.100.159
```

Generate passwords

```text
$ kolla-genpwd
```

Deploy OpenStack.

```text
$ kolla-ansible -i ~/multinode deploy
```

Once deployment finish, generate credential file and create base networks and a cirros image.

```text
$ kolla-ansible -i ~/multinode post-deploy
$ source /etc/kolla/admin-openrc.sh
$ sh init-runonce
```

### Tacker and SFC demo

In kolla-ansible repository a tacker demo is present. [Tacker demo](https://github.com/openstack/kolla-ansible/tree/master/contrib/demos/tacker)

```text
$ cd <kolla-ansible repo>/contrib/demos/tacker/
$ ls -l
total 16
-rw-r--r-- 1 root root  615 Aug 24 20:21 cleanup-tacker
-rw-r--r-- 1 root root 1937 Aug 24 20:21 deploy-tacker-demo
-rw-r--r-- 1 root root 2649 Aug 24 20:21 deploy-tacker-demo-sfc
-rw-r--r-- 1 root root  396 Aug 18 13:53 README.rst
```

Before starting the demo, install tacker and networking-sfc clients.

```text
$ pip install python-tackerclient networking-sfc
```

#### Demo description

Tacker demo for SFC will create the following resources:

* Tacker default VIM
* Tacker VNFD
* Tacker VNF
* kolla\_sfc\_client instance with a floating IP
* kolla\_sfc\_server instance with a floating IP
* Tacker VNFFGD
* Tacker VNFFG

After demo is deployed will be able to:

* Create sample web server in kolla\_sfc\_server instance.
* Request web service from kolla\_sfc\_client

Traffic flows:

* Request from kolla\_sfc\_client instance
* Tacker VNF instance will receive the traffic and redirect to

  kolla\_sfc\_server

* kolla\_sfc\_server instance receive request and reply with \"W00t

  from Kolla HTTP server!\" message.

#### Execute tacker demo

In tacker demo directory initialize execution.

```text
$ sh deploy-tacker-demo-sfc

Generating sample config
Registering sample VIM
Created a new vim:
+----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field          | Value                                                                                                                                                                                                                                                  |
+----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| auth_cred      | {"username": "admin", "password": "***", "project_name": "admin", "user_domain_name": "Default", "key_type": "barbican_key", "secret_uuid": "***", "auth_url": "http://192.168.100.10:35357/v3", "project_id": null, "project_domain_name": "Default"} |
| auth_url       | http://192.168.100.10:35357/v3                                                                                                                                                                                                                         |
| created_at     | 2017-08-28 08:49:01.385013                                                                                                                                                                                                                             |
| description    | kolla sample vim                                                                                                                                                                                                                                       |
| id             | 0cb20dff-b6d2-44ab-9124-cdeb018269a2                                                                                                                                                                                                                   |
| is_default     | True                                                                                                                                                                                                                                                   |
| name           | kolla-sample-vim                                                                                                                                                                                                                                       |
| placement_attr | {"regions": ["RegionOne"]}                                                                                                                                                                                                                             |
| status         | PENDING                                                                                                                                                                                                                                                |
| tenant_id      | 9fb078d4c7e54a92b3068eb5c0f83ec5                                                                                                                                                                                                                       |
| type           | openstack                                                                                                                                                                                                                                              |
| updated_at     |                                                                                                                                                                                                                                                        |
| vim_project    | {"name": "admin", "project_domain_name": "Default"}                                                                                                                                                                                                    |
+----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Creating sample VNFD
Created a new vnfd:
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| created_at      | 2017-08-28 08:49:03.915848           |
| description     | Demo example                         |
| id              | d9633774-f9a4-492c-8055-ff3b2bc08581 |
| name            | kolla-sample-vnfd                    |
| service_types   | vnfd                                 |
| template_source | onboarded                            |
| tenant_id       | 9fb078d4c7e54a92b3068eb5c0f83ec5     |
| updated_at      |                                      |
+-----------------+--------------------------------------+
Creating sample VNF
Created a new vnf:
+----------------+--------------------------------------+
| Field          | Value                                |
+----------------+--------------------------------------+
| created_at     | 2017-08-28 08:49:08.921243           |
| description    | Demo example                         |
| error_reason   |                                      |
| id             | a3f73d1b-6d6b-44c9-a6ef-a808f12bc633 |
| instance_id    | 57246b92-fdf2-416f-921a-6760e05c74b4 |
| mgmt_url       |                                      |
| name           | kolla-sample-vnf                     |
| placement_attr | {"vim_name": "kolla-sample-vim"}     |
| status         | PENDING_CREATE                       |
| tenant_id      | 9fb078d4c7e54a92b3068eb5c0f83ec5     |
| updated_at     |                                      |
| vim_id         | 0cb20dff-b6d2-44ab-9124-cdeb018269a2 |
| vnfd_id        | d9633774-f9a4-492c-8055-ff3b2bc08581 |
+----------------+--------------------------------------+
Creating SFC demo instances

+-------------------------------------+----------------------------------------------------------+
| Field                               | Value                                                    |
+-------------------------------------+----------------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                                   |
| OS-EXT-AZ:availability_zone         | nova                                                     |
| OS-EXT-SRV-ATTR:host                | controller                                               |
| OS-EXT-SRV-ATTR:hypervisor_hostname | controller                                               |
| OS-EXT-SRV-ATTR:instance_name       | instance-0000000c                                        |
| OS-EXT-STS:power_state              | Running                                                  |
| OS-EXT-STS:task_state               | None                                                     |
| OS-EXT-STS:vm_state                 | active                                                   |
| OS-SRV-USG:launched_at              | 2017-08-28T08:50:00.000000                               |
| OS-SRV-USG:terminated_at            | None                                                     |
| accessIPv4                          |                                                          |
| accessIPv6                          |                                                          |
| addresses                           | demo-net=10.0.0.3                                        |
| adminPass                           | HGW57Pe5r8pC                                             |
| config_drive                        |                                                          |
| created                             | 2017-08-28T08:49:41Z                                     |
| flavor                              | m1.tiny (1)                                              |
| hostId                              | bec7629dd00bde2fd03ac3c939eea34fa1a2f7e4a6f8337b0e08bca4 |
| id                                  | 90304f68-6b38-4753-b0c2-a62835abebde                     |
| image                               | cirros (f0a80381-2bd0-4c53-8300-377a7e4bf065)            |
| key_name                            | None                                                     |
| name                                | kolla_sfc_server                                         |
| progress                            | 0                                                        |
| project_id                          | 9fb078d4c7e54a92b3068eb5c0f83ec5                         |
| properties                          |                                                          |
| security_groups                     | name='default'                                           |
| status                              | ACTIVE                                                   |
| updated                             | 2017-08-28T08:50:00Z                                     |
| user_id                             | 2d948bf4056c4e0d878a0f3f4765d3f9                         |
| volumes_attached                    |                                                          |
+-------------------------------------+----------------------------------------------------------+

+-------------------------------------+----------------------------------------------------------+
| Field                               | Value                                                    |
+-------------------------------------+----------------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                                   |
| OS-EXT-AZ:availability_zone         | nova                                                     |
| OS-EXT-SRV-ATTR:host                | compute1                                                 |
| OS-EXT-SRV-ATTR:hypervisor_hostname | compute1                                                 |
| OS-EXT-SRV-ATTR:instance_name       | instance-0000000e                                        |
| OS-EXT-STS:power_state              | Running                                                  |
| OS-EXT-STS:task_state               | None                                                     |
| OS-EXT-STS:vm_state                 | active                                                   |
| OS-SRV-USG:launched_at              | 2017-08-28T08:50:31.000000                               |
| OS-SRV-USG:terminated_at            | None                                                     |
| accessIPv4                          |                                                          |
| accessIPv6                          |                                                          |
| addresses                           | demo-net=10.0.0.7                                        |
| adminPass                           | ZhQQG2vsetkV                                             |
| config_drive                        |                                                          |
| created                             | 2017-08-28T08:50:12Z                                     |
| flavor                              | m1.tiny (1)                                              |
| hostId                              | 95d62e067390ab4fbaaebf971f9cc70c98c371532b6f9bfa08389fee |
| id                                  | e126fd2f-bdca-4e78-abc0-f0a2d4739a30                     |
| image                               | cirros (f0a80381-2bd0-4c53-8300-377a7e4bf065)            |
| key_name                            | None                                                     |
| name                                | kolla_sfc_client                                         |
| progress                            | 0                                                        |
| project_id                          | 9fb078d4c7e54a92b3068eb5c0f83ec5                         |
| properties                          |                                                          |
| security_groups                     | name='default'                                           |
| status                              | ACTIVE                                                   |
| updated                             | 2017-08-28T08:50:31Z                                     |
| user_id                             | 2d948bf4056c4e0d878a0f3f4765d3f9                         |
| volumes_attached                    |                                                          |
+-------------------------------------+----------------------------------------------------------+
Tacker SFC config files
Creating VNFFGD
Created a new vnffgd:
+-----------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field           | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
+-----------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| description     |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| id              | 97d9f9e9-f9c3-45b6-9050-000226d37ec9                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| name            | kolla-sample-vnffgd                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| template        | {"vnffgd": {"imports": ["/var/lib/kolla/venv/lib/python2.7/site-packages/tacker/tosca/lib/tacker_defs.yaml", "/var/lib/kolla/venv/lib/python2.7/site-packages/tacker/tosca/lib/tacker_nfv_defs.yaml"], "description": "Sample VNFFG template", "topology_template": {"node_templates": {"Forwarding_path1": {"type": "tosca.nodes.nfv.FP.Tacker", "description": "creates path (CP12->CP12)", "properties": {"policy": {"type": "ACL", "criteria": [{"network_src_port_id": "2779e692-f979-467c-81ae-34a176e12ed4"}, {"network_id": "9ab78f83-40b7-4435-be5c-eb40de435793"}, {"ip_proto": 6}, {"destination_port_range": "80-80"}]}, "path": [{"capability": "CP11", "forwarder": "kolla-sample-vnfd"}], "id": 51}}}, "description": "Sample VNFFG template", "groups": {"VNFFG1": {"type": "tosca.groups.nfv.VNFFG", "description": "HTTP to Corporate Net", "members": ["Forwarding_path1"], "properties": {"vendor": "tacker", "connection_point": ["CP11"], "version": 1.0, "constituent_vnfs": ["kolla-sample-vnfd"], "number_of_endpoints": 1, "dependent_virtual_link": ["VL1"]}}}}, "tosca_definitions_version": "tosca_simple_profile_for_nfv_1_0_0"}} |
| template_source | onboarded                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| tenant_id       | 9fb078d4c7e54a92b3068eb5c0f83ec5                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
+-----------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Creating VNFFG
Created a new vnffg:
+------------------+---------------------------------------------------------------+
| Field            | Value                                                         |
+------------------+---------------------------------------------------------------+
| description      |                                                               |
| forwarding_paths | cff3d46e-9544-4775-82c3-0ac3c1f3864c                          |
| id               | a1e2a010-0c64-40ad-a54a-96d7e9d8e6a5                          |
| name             | kolla-sample-vnffg                                            |
| status           | PENDING_CREATE                                                |
| tenant_id        | 9fb078d4c7e54a92b3068eb5c0f83ec5                              |
| vnf_mapping      | {"kolla-sample-vnfd": "a3f73d1b-6d6b-44c9-a6ef-a808f12bc633"} |
| vnffgd_id        | 97d9f9e9-f9c3-45b6-9050-000226d37ec9                          |
+------------------+---------------------------------------------------------------+
Tacker sfc client floating ip address: 192.168.150.102
Tacker sfc server floating ip address: 192.168.150.110

Done.

To create simple HTTP server in tacker_sfc_server instance run:

ssh cirros@192.168.150.110 'while true; \
    do echo -e "HTTP/1.0 200 OK\r\n\r\nW00t from Kolla HTTP server!" | sudo nc -l -p 80 ; done &'
```

Once finished, script will show server and client floating IP addresses, also a sample command to start a basic HTTP server in tacker\_sfc\_server instance.

#### Validate resources

Verify tacker resources are created.

```text
$ tacker vim-list

+--------------------------------------+----------------------------------+------------------+-----------+------------+------------------------------+-----------+
| id                                   | tenant_id                        | name             | type      | is_default | placement_attr               | status    |
+--------------------------------------+----------------------------------+------------------+-----------+------------+------------------------------+-----------+
| 0cb20dff-b6d2-44ab-9124-cdeb018269a2 | 9fb078d4c7e54a92b3068eb5c0f83ec5 | kolla-sample-vim | openstack | True       | {u'regions': [u'RegionOne']} | REACHABLE |
+--------------------------------------+----------------------------------+------------------+-----------+------------+------------------------------+-----------+

$ tacker vnf-list
+--------------------------------------+------------------+-----------------------+--------+--------------------------------------+--------------------------------------+
| id                                   | name             | mgmt_url              | status | vim_id                               | vnfd_id                              |
+--------------------------------------+------------------+-----------------------+--------+--------------------------------------+--------------------------------------+
| a3f73d1b-6d6b-44c9-a6ef-a808f12bc633 | kolla-sample-vnf | {"VDU1": "10.0.0.12"} | ACTIVE | 0cb20dff-b6d2-44ab-9124-cdeb018269a2 | d9633774-f9a4-492c-8055-ff3b2bc08581 |
+--------------------------------------+------------------+-----------------------+--------+--------------------------------------+--------------------------------------+

$ tacker vnfd-list
+--------------------------------------+-------------------+-----------------+--------------+
| id                                   | name              | template_source | description  |
+--------------------------------------+-------------------+-----------------+--------------+
| d9633774-f9a4-492c-8055-ff3b2bc08581 | kolla-sample-vnfd | onboarded       | Demo example |
+--------------------------------------+-------------------+-----------------+--------------+
```

Verify nova and heat resources are created.

```text
$ openstack server list

+--------------------------------------+-------------------------------------------------------+--------+------------------------------------+--------+-----------------------------------------------------------------------------------------------------------------------+
| ID                                   | Name                                                  | Status | Networks                           | Image  | Flavor                                                                                                                |
+--------------------------------------+-------------------------------------------------------+--------+------------------------------------+--------+-----------------------------------------------------------------------------------------------------------------------+
| e126fd2f-bdca-4e78-abc0-f0a2d4739a30 | kolla_sfc_client                                      | ACTIVE | demo-net=10.0.0.7, 192.168.150.102 | cirros | m1.tiny                                                                                                               |
| 90304f68-6b38-4753-b0c2-a62835abebde | kolla_sfc_server                                      | ACTIVE | demo-net=10.0.0.3, 192.168.150.110 | cirros | m1.tiny                                                                                                               |
| 61e2ec3a-444f-4048-bc8d-a599e29e14bd | ta-3d1b-6d6b-44c9-a6ef-a808f12bc633-VDU1-hvpraqctwpm7 | ACTIVE | demo-net=10.0.0.12                 | cirros | tacker.vnfm.infra_drivers.openstack.openstack_OpenStack-a3f73d1b-6d6b-44c9-a6ef-a808f12bc633-VDU1_flavor-4vsmp3jlvilk |
+--------------------------------------+-------------------------------------------------------+--------+------------------------------------+--------+-----------------------------------------------------------------------------------------------------------------------+

$ openstack stack list
+--------------------------------------+----------------------------------------------------------------------------------------------+----------------------------------+-----------------+----------------------+--------------+
| ID                                   | Stack Name                                                                                   | Project                          | Stack Status    | Creation Time        | Updated Time |
+--------------------------------------+----------------------------------------------------------------------------------------------+----------------------------------+-----------------+----------------------+--------------+
| 57246b92-fdf2-416f-921a-6760e05c74b4 | tacker.vnfm.infra_drivers.openstack.openstack_OpenStack-a3f73d1b-6d6b-44c9-a6ef-a808f12bc633 | 9fb078d4c7e54a92b3068eb5c0f83ec5 | CREATE_COMPLETE | 2017-08-28T08:49:11Z | None         |
+--------------------------------------+----------------------------------------------------------------------------------------------+----------------------------------+-----------------+----------------------+--------------+
```

Verify networking-sfc resources.

```text
$ openstack sfc port chain list

+--------------------------------------+-------------------------------+-------------------------------------------+-------------------------------------------+------------------------------------------------+
| ID                                   | Name                          | Port Pair Groups                          | Flow Classifiers                          | Chain Parameters                               |
+--------------------------------------+-------------------------------+-------------------------------------------+-------------------------------------------+------------------------------------------------+
| ec35dfac-dc9d-40b8-8103-b510761753ae | kolla-sample-vnffg-port-chain | [u'9b73262f-f25d-400b-8aff-062d66a3bd76'] | [u'063231fc-f697-4bd9-bfb6-b89f89ff6117'] | {u'symmetric': False, u'correlation': u'mpls'} |
+--------------------------------------+-------------------------------+-------------------------------------------+-------------------------------------------+------------------------------------------------+

$ openstack sfc port chain show kolla-sample-vnffg-port-chain
+------------------+------------------------------------------------+
| Field            | Value                                          |
+------------------+------------------------------------------------+
| chain_id         | 1                                              |
| chain_parameters | {u'symmetric': False, u'correlation': u'mpls'} |
| description      | port-chain for Tacker VNFFG                    |
| flow_classifiers | [u'063231fc-f697-4bd9-bfb6-b89f89ff6117']      |
| id               | ec35dfac-dc9d-40b8-8103-b510761753ae           |
| name             | kolla-sample-vnffg-port-chain                  |
| port_pair_groups | [u'9b73262f-f25d-400b-8aff-062d66a3bd76']      |
| project_id       | 9fb078d4c7e54a92b3068eb5c0f83ec5               |
+------------------+------------------------------------------------+

$ openstack sfc port pair group list
+--------------------------------------+----------------------------------+-------------------------------------------+---------------------------------------------------------------------------------------------+
| ID                                   | Name                             | Port Pair                                 | Port Pair Group Parameters                                                                  |
+--------------------------------------+----------------------------------+-------------------------------------------+---------------------------------------------------------------------------------------------+
| 9b73262f-f25d-400b-8aff-062d66a3bd76 | kolla-sample-vnf-port-pair-group | [u'bb944348-2610-4068-8c87-9288904edf11'] | {u'lb_fields': [], u'ppg_n_tuple_mapping': {u'ingress_n_tuple': {}, u'egress_n_tuple': {}}} |
+--------------------------------------+----------------------------------+-------------------------------------------+---------------------------------------------------------------------------------------------+
$ openstack sfc port pair group show kolla-sample-vnf-port-pair-group
+----------------------------+---------------------------------------------------------------------------------------------+
| Field                      | Value                                                                                       |
+----------------------------+---------------------------------------------------------------------------------------------+
| description                | port pair group for kolla-sample-vnf                                                        |
| group_id                   | 1                                                                                           |
| id                         | 9b73262f-f25d-400b-8aff-062d66a3bd76                                                        |
| name                       | kolla-sample-vnf-port-pair-group                                                            |
| port_pair_group_parameters | {u'lb_fields': [], u'ppg_n_tuple_mapping': {u'ingress_n_tuple': {}, u'egress_n_tuple': {}}} |
| port_pairs                 | [u'bb944348-2610-4068-8c87-9288904edf11']                                                   |
| project_id                 | 9fb078d4c7e54a92b3068eb5c0f83ec5                                                            |
+----------------------------+---------------------------------------------------------------------------------------------+

$ openstack sfc flow classifier list
+--------------------------------------+------+----------+-----------+----------------+--------------------------------------+--------------------------+
| ID                                   | Name | Protocol | Source-IP | Destination-IP | Logical-Source-Port                  | Logical-Destination-Port |
+--------------------------------------+------+----------+-----------+----------------+--------------------------------------+--------------------------+
| 063231fc-f697-4bd9-bfb6-b89f89ff6117 |      | tcp      | None      | None           | 2779e692-f979-467c-81ae-34a176e12ed4 | None                     |
+--------------------------------------+------+----------+-----------+----------------+--------------------------------------+--------------------------+

$ openstack sfc flow classifier show 063231fc-f697-4bd9-bfb6-b89f89ff6117
+----------------------------+--------------------------------------+
| Field                      | Value                                |
+----------------------------+--------------------------------------+
| description                |                                      |
| destination_ip_prefix      | None                                 |
| destination_port_range_max | 80                                   |
| destination_port_range_min | 80                                   |
| ethertype                  | IPv4                                 |
| id                         | 063231fc-f697-4bd9-bfb6-b89f89ff6117 |
| l7_parameters              | {}                                   |
| logical_destination_port   | None                                 |
| logical_source_port        | 2779e692-f979-467c-81ae-34a176e12ed4 |
| name                       |                                      |
| project_id                 | 9fb078d4c7e54a92b3068eb5c0f83ec5     |
| protocol                   | tcp                                  |
| source_ip_prefix           | None                                 |
| source_port_range_max      | None                                 |
| source_port_range_min      | None                                 |
+----------------------------+--------------------------------------+
```

#### Verify traffic flows

Execute the command to create a sample web server in tacker\_sfc\_server.

```text
$ ssh cirros@192.168.150.110 'while true; \
>     do echo -e "HTTP/1.0 200 OK\r\n\r\nW00t from Kolla HTTP server!" | sudo nc -l -p 80 ; done &'

The authenticity of host '192.168.150.110 (192.168.150.110)' can't be established.
RSA key fingerprint is c6:14:b1:d9:84:b5:83:54:47:8e:20:eb:81:a2:f7:62.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.150.110' (RSA) to the list of known hosts.
cirros@192.168.150.110's password:
```

Connect to tacker\_sfc\_client through the floating IP address

```text
$ ssh cirros@192.168.150.102

The authenticity of host '192.168.150.102 (192.168.150.102)' can't be established.
RSA key fingerprint is 5e:51:88:93:70:90:0e:24:55:81:47:b4:d6:28:4b:f9.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.150.102' (RSA) to the list of known hosts.
cirros@192.168.150.102's password:
```

Curl to tacker\_sfc\_server internal/fixed IP address. 

Should receive \"W00t from Kolla HTTP server!\" message

```text
$ curl http://10.0.0.3
W00t from Kolla HTTP server!
```

Find hypervisor where tacker VNF instance is running.

```text
$ openstack server list -c Name -c Host -c Networks -c Status --long

+-------------------------------------------------------+--------+------------------------------------+------------+
| Name                                                  | Status | Networks                           | Host       |
+-------------------------------------------------------+--------+------------------------------------+------------+
| kolla_sfc_client                                      | ACTIVE | demo-net=10.0.0.7, 192.168.150.102 | compute1   |
| kolla_sfc_server                                      | ACTIVE | demo-net=10.0.0.3, 192.168.150.110 | controller |
| ta-3d1b-6d6b-44c9-a6ef-a808f12bc633-VDU1-hvpraqctwpm7 | ACTIVE | demo-net=10.0.0.12                 | compute1   |
+-------------------------------------------------------+--------+------------------------------------+------------+
```

Find tacker VNF instance port ID

```text
$ openstack port list --server ta-3d1b-6d6b-44c9-a6ef-a808f12bc633-VDU1-hvpraqctwpm7 -c ID
+--------------------------------------+
| ID                                   |
+--------------------------------------+
| e5da60a7-a348-4bee-a52a-96ae33b53a26 |
+--------------------------------------+
```

In the host where the instance is running, locate the tap interface. 

Tap interface is `tap<first 11 ID digits>`. 

Start tcpdump in port 80 in the tap interface.

```text
$ tcpdump port 80 -eni tape5da60a7-a3

tcpdump: WARNING: tape5da60a7-a3: no IPv4 address assigned
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tape5da60a7-a3, link-type EN10MB (Ethernet), capture size 65535 bytes
```

Now curl again from tacker\_sfc\_client instance.

```text
$ curl http://10.0.0.3
W00t from Kolla HTTP server!
```

In the tcpdump should see traffic flowing to tacker\_sfc\_server from tacker\_sfc\_client

```text
10:18:39.207908 fa:16:3e:6d:65:14 > fa:16:3e:2f:3e:90, ethertype IPv4 (0x0800), length 74: 10.0.0.7.40475 > 10.0.0.3.http: Flags [S], seq 3060324847, win 14100, options [mss 1410,sackOK,TS val 346030 ecr 0,nop,wscale 3], length 0
10:18:39.209263 fa:16:3e:2f:3e:90 > fa:16:3e:d7:6f:3b, ethertype IPv4 (0x0800), length 74: 10.0.0.7.40475 > 10.0.0.3.http: Flags [S], seq 3060324847, win 14100, options [mss 1410,sackOK,TS val 346030 ecr 0,nop,wscale 3], length 0
10:18:39.214001 fa:16:3e:6d:65:14 > fa:16:3e:2f:3e:90, ethertype IPv4 (0x0800), length 66: 10.0.0.7.40475 > 10.0.0.3.http: Flags [.], ack 2793310193, win 1763, options [nop,nop,TS val 346032 ecr 352982], length 0
10:18:39.214924 fa:16:3e:2f:3e:90 > fa:16:3e:d7:6f:3b, ethertype IPv4 (0x0800), length 66: 10.0.0.7.40475 > 10.0.0.3.http: Flags [.], ack 1, win 1763, options [nop,nop,TS val 346032 ecr 352982], length 0
10:18:39.222308 fa:16:3e:6d:65:14 > fa:16:3e:2f:3e:90, ethertype IPv4 (0x0800), length 201: 10.0.0.7.40475 > 10.0.0.3.http: Flags [P.], seq 0:135, ack 1, win 1763, options [nop,nop,TS val 346034 ecr 352982], length 135
10:18:39.222333 fa:16:3e:6d:65:14 > fa:16:3e:2f:3e:90, ethertype IPv4 (0x0800), length 66: 10.0.0.7.40475 > 10.0.0.3.http: Flags [.], ack 49, win 1763, options [nop,nop,TS val 346034 ecr 352983], length 0
10:18:39.224660 fa:16:3e:2f:3e:90 > fa:16:3e:d7:6f:3b, ethertype IPv4 (0x0800), length 201: 10.0.0.7.40475 > 10.0.0.3.http: Flags [P.], seq 0:135, ack 1, win 1763, options [nop,nop,TS val 346034 ecr 352982], length 135
10:18:39.224773 fa:16:3e:2f:3e:90 > fa:16:3e:d7:6f:3b, ethertype IPv4 (0x0800), length 66: 10.0.0.7.40475 > 10.0.0.3.http: Flags [.], ack 49, win 1763, options [nop,nop,TS val 346034 ecr 352983], length 0
10:18:39.250113 fa:16:3e:6d:65:14 > fa:16:3e:2f:3e:90, ethertype IPv4 (0x0800), length 66: 10.0.0.7.40475 > 10.0.0.3.http: Flags [F.], seq 135, ack 50, win 1763, options [nop,nop,TS val 346041 ecr 352990], length 0
10:18:39.252871 fa:16:3e:2f:3e:90 > fa:16:3e:d7:6f:3b, ethertype IPv4 (0x0800), length 66: 10.0.0.7.40475 > 10.0.0.3.http: Flags [F.], seq 135, ack 50, win 1763, options [nop,nop,TS val 346041 ecr 352990], length 0
```

Check br-int ovs flows.

```text
$ docker exec openvswitch_db ovs-ofctl dump-flows br-int

NXST_FLOW reply (xid=0x4):
 cookie=0x4d21ac58ca610153, duration=1952.266s, table=0, n_packets=20, n_bytes=1892, idle_age=300, priority=30,tcp,in_port=4,nw_src=10.0.0.7,tp_dst=80 actions=NORMAL
 cookie=0x4d21ac58ca610153, duration=1951.781s, table=0, n_packets=20, n_bytes=1892, idle_age=300, priority=30,tcp,in_port=5,nw_src=10.0.0.7,tp_dst=80 actions=group:1
 cookie=0x4d21ac58ca610153, duration=3824.910s, table=0, n_packets=0, n_bytes=0, idle_age=3824, priority=20,mpls actions=resubmit(,10)
 cookie=0x5efe7af1c4c4da43, duration=1986.768s, table=0, n_packets=0, n_bytes=0, idle_age=1986, priority=10,icmp6,in_port=5,icmp_type=136 actions=resubmit(,24)
 cookie=0x5efe7af1c4c4da43, duration=1986.765s, table=0, n_packets=9, n_bytes=378, idle_age=296, priority=10,arp,in_port=5 actions=resubmit(,24)
 cookie=0x5efe7af1c4c4da43, duration=1986.771s, table=0, n_packets=151, n_bytes=16475, idle_age=300, priority=9,in_port=5 actions=resubmit(,25)
 cookie=0x5efe7af1c4c4da43, duration=3828.290s, table=0, n_packets=575, n_bytes=63516, idle_age=296, priority=0 actions=resubmit(,60)
 cookie=0x4d21ac58ca610153, duration=1952.498s, table=5, n_packets=20, n_bytes=1892, idle_age=300, priority=0,ip,dl_dst=fa:16:3e:2f:3e:90 actions=push_mpls:0x8847,load:0x1ff->OXM_OF_MPLS_LABEL[],set_mpls_ttl(255),mod_vlan_vid:2,resubmit(,10)
 cookie=0x4d21ac58ca610153, duration=1951.976s, table=10, n_packets=20, n_bytes=1892, idle_age=300, priority=1,mpls,dl_vlan=2,dl_dst=fa:16:3e:2f:3e:90,mpls_label=511 actions=strip_vlan,pop_mpls:0x0800,output:4
 cookie=0x4d21ac58ca610153, duration=3824.909s, table=10, n_packets=0, n_bytes=0, idle_age=3824, priority=0 actions=drop
 cookie=0x5efe7af1c4c4da43, duration=3828.292s, table=23, n_packets=0, n_bytes=0, idle_age=3828, priority=0 actions=drop
 cookie=0x5efe7af1c4c4da43, duration=1986.769s, table=24, n_packets=0, n_bytes=0, idle_age=1986, priority=2,icmp6,in_port=5,icmp_type=136,nd_target=fe80::f816:3eff:fe6d:6514 actions=resubmit(,60)
 cookie=0x5efe7af1c4c4da43, duration=1986.766s, table=24, n_packets=9, n_bytes=378, idle_age=296, priority=2,arp,in_port=5,arp_spa=10.0.0.7 actions=resubmit(,25)
 cookie=0x5efe7af1c4c4da43, duration=3828.286s, table=24, n_packets=0, n_bytes=0, idle_age=3828, priority=0 actions=drop
 cookie=0x5efe7af1c4c4da43, duration=1986.774s, table=25, n_packets=159, n_bytes=16783, idle_age=296, priority=2,in_port=5,dl_src=fa:16:3e:6d:65:14 actions=resubmit(,60)
 cookie=0x5efe7af1c4c4da43, duration=3828.287s, table=60, n_packets=836, n_bytes=90430, idle_age=296, priority=3 actions=NORMAL
```

Check br-tun ovs flows.

```text
$ docker exec openvswitch_db ovs-ofctl dump-flows br-tun

NXST_FLOW reply (xid=0x4):
 cookie=0xf206a4cf831522bb, duration=3829.004s, table=0, n_packets=514, n_bytes=51389, idle_age=299, priority=1,in_port=1 actions=resubmit(,2)
 cookie=0xf206a4cf831522bb, duration=2049.122s, table=0, n_packets=111, n_bytes=13035, idle_age=299, priority=1,in_port=4 actions=resubmit(,4)
 cookie=0xf206a4cf831522bb, duration=2049.077s, table=0, n_packets=99, n_bytes=12561, idle_age=1976, priority=1,in_port=5 actions=resubmit(,4)
 cookie=0xf206a4cf831522bb, duration=3829.003s, table=0, n_packets=0, n_bytes=0, idle_age=3829, priority=0 actions=drop
 cookie=0xf206a4cf831522bb, duration=3829.001s, table=2, n_packets=14, n_bytes=588, idle_age=303, priority=1,arp,dl_dst=ff:ff:ff:ff:ff:ff actions=resubmit(,21)
 cookie=0xf206a4cf831522bb, duration=3828.999s, table=2, n_packets=444, n_bytes=44273, idle_age=299, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0xf206a4cf831522bb, duration=3828.998s, table=2, n_packets=56, n_bytes=6528, idle_age=1971, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)
 cookie=0xf206a4cf831522bb, duration=3828.997s, table=3, n_packets=0, n_bytes=0, idle_age=3828, priority=0 actions=drop
 cookie=0xf206a4cf831522bb, duration=2051.673s, table=4, n_packets=210, n_bytes=25596, idle_age=299, priority=1,tun_id=0x35 actions=mod_vlan_vid:2,resubmit(,10)
 cookie=0xf206a4cf831522bb, duration=3828.996s, table=4, n_packets=0, n_bytes=0, idle_age=3828, priority=0 actions=drop
 cookie=0xf206a4cf831522bb, duration=3828.995s, table=6, n_packets=0, n_bytes=0, idle_age=3828, priority=0 actions=drop
 cookie=0xf206a4cf831522bb, duration=3828.994s, table=10, n_packets=322, n_bytes=39749, idle_age=299, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,cookie=0xf206a4cf831522bb,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:OXM_OF_IN_PORT[]),output:1
 cookie=0xf206a4cf831522bb, duration=2045.177s, table=20, n_packets=60, n_bytes=7135, idle_age=299, priority=2,dl_vlan=2,dl_dst=fa:16:3e:dd:b9:4d actions=strip_vlan,load:0x35->NXM_NX_TUN_ID[],output:4
 cookie=0xf206a4cf831522bb, duration=2045.168s, table=20, n_packets=180, n_bytes=17440, idle_age=1976, priority=2,dl_vlan=2,dl_dst=fa:16:3e:71:84:df actions=strip_vlan,load:0x35->NXM_NX_TUN_ID[],output:5
 cookie=0xf206a4cf831522bb, duration=2019.366s, table=20, n_packets=20, n_bytes=1892, idle_age=303, priority=2,dl_vlan=2,dl_dst=fa:16:3e:d7:6f:3b actions=strip_vlan,load:0x35->NXM_NX_TUN_ID[],output:4
 cookie=0xf206a4cf831522bb, duration=304.629s, table=20, n_packets=0, n_bytes=0, hard_timeout=300, idle_age=304, hard_age=299, priority=1,vlan_tci=0x0002/0x0fff,dl_dst=fa:16:3e:dd:b9:4d actions=load:0->NXM_OF_VLAN_TCI[],load:0x35->NXM_NX_TUN_ID[],output:4
 cookie=0xf206a4cf831522bb, duration=3828.992s, table=20, n_packets=0, n_bytes=0, idle_age=3828, priority=0 actions=resubmit(,22)
 cookie=0xf206a4cf831522bb, duration=2045.180s, table=21, n_packets=2, n_bytes=84, idle_age=1974, priority=1,arp,dl_vlan=2,arp_tpa=10.0.0.1 actions=load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xfa163eddb94d->NXM_NX_ARP_SHA[],load:0xa000001->NXM_OF_ARP_SPA[],move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:fa:16:3e:dd:b9:4d,IN_PORT
 cookie=0xf206a4cf831522bb, duration=2045.170s, table=21, n_packets=2, n_bytes=84, idle_age=1979, priority=1,arp,dl_vlan=2,arp_tpa=10.0.0.2 actions=load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xfa163e7184df->NXM_NX_ARP_SHA[],load:0xa000002->NXM_OF_ARP_SPA[],move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:fa:16:3e:71:84:df,IN_PORT
 cookie=0xf206a4cf831522bb, duration=2019.369s, table=21, n_packets=4, n_bytes=168, idle_age=303, priority=1,arp,dl_vlan=2,arp_tpa=10.0.0.3 actions=load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xfa163ed76f3b->NXM_NX_ARP_SHA[],load:0xa000003->NXM_OF_ARP_SPA[],move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:fa:16:3e:d7:6f:3b,IN_PORT
 cookie=0xf206a4cf831522bb, duration=3828.991s, table=21, n_packets=2, n_bytes=84, idle_age=303, priority=0 actions=resubmit(,22)
 cookie=0xf206a4cf831522bb, duration=2045.174s, table=22, n_packets=20, n_bytes=2512, idle_age=303, priority=1,dl_vlan=2 actions=strip_vlan,load:0x35->NXM_NX_TUN_ID[],output:4,output:5
 cookie=0xf206a4cf831522bb, duration=3828.990s, table=22, n_packets=20, n_bytes=1672, idle_age=1994, priority=0 actions=drop
```

Once Tacker and SFC is verified, all resources can be deleted.

```text
$ sh cleanup-tacker

Deleting VNFFG
All specified vnffg(s) deleted successfully
Deleting VNFFGD
All specified vnffgd(s) deleted successfully
Deleting sample sfc instances
Deleting sample VNF
All specified vnf(s) delete initiated successfully
Deleting sample VNFD
All specified vnfd(s) deleted successfully
Deleting sample VIM
All specified vim(s) deleted successfully
Removing sample config
```

In following posts will show how to tacker templates works and an in deep sfc traffic flows analysis.

Regards, Eduardo Gonzalez

