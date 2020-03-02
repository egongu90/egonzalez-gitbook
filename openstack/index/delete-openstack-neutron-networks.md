---
description: Fix to unable to complete operation on subnet
---

# Delete OpenStack neutron networks

```text
[root@rdoicehouse ~(keystone_admin)]# neutron router-list
+--------------------------------------+------------------+-----------------------------------------------------------------------------+
| id                                   | name             | external_gateway_info                                                       |
+--------------------------------------+------------------+-----------------------------------------------------------------------------+
 e34d94ad-7fe1-4704-8156-d255a2daa167 | demodeleterouter | {"network_id": "8b2ceda2-4d77-4c5c-ae21-6a7ba133e4fc", "enable_snat": true} |
--------------------------------------+------------------+-----------------------------------------------------------------------------+
[root@rdoicehouse ~(keystone_admin)]# neutron router-gateway-clear e34d94ad-7fe1-4704-8156-d255a2daa167
Removed gateway from router e34d94ad-7fe1-4704-8156-d255a2daa167

[root@rdoicehouse ~(keystone_admin)]# neutron router-port-list e34d94ad-7fe1-4704-8156-d255a2daa167

If Apply: 
         [[ neutron router-interface-delete <router-id> <subnet-id> ]]

[root@rdoicehouse ~(keystone_admin)]# neutron router-delete e34d94ad-7fe1-4704-8156-d255a2daa167
Deleted router: e34d94ad-7fe1-4704-8156-d255a2daa167

[root@rdoicehouse ~(keystone_admin)]# neutron subnet-list
+--------------------------------------+------------------+------------------+--------------------------------------------------------+
| id                                   | name             | cidr             | allocation_pools                                       |
+--------------------------------------+------------------+------------------+--------------------------------------------------------+
| d50e28f7-47ee-4bdf-8594-e1108f25586b | demosubnetdelete | 192.168.137.0/24 | {"start": "192.168.137.100", "end": "192.168.137.120"} |
| c93fd5a7-d672-4b0c-8f2e-6e74f487e45d | private_subnet   | 10.0.0.0/24      | {"start": "10.0.0.2", "end": "10.0.0.254"}             |
+--------------------------------------+------------------+------------------+--------------------------------------------------------+
[root@rdoicehouse ~(keystone_admin)]# neutron subnet-delete d50e28f7-47ee-4bdf-8594-e1108f25586b
409-{u'NeutronError': {u'message': u'Unable to complete operation on subnet d50e28f7-47ee-4bdf-8594-e1108f25586b. One or more ports have an IP allocation from this subnet.', u'type': u'SubnetInUse', u'detail': u''}}

[root@rdoicehouse ~(keystone_admin)]# neutron port-list
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------+
| id                                   | name | mac_address       | fixed_ips                                                                              |
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------+
| 4655e13a-9767-4750-8f44-4eee8410ca70 |      | fa:16:3e:03:c7:cf | {"subnet_id": "d50e28f7-47ee-4bdf-8594-e1108f25586b", "ip_address": "192.168.137.103"} |
| 767f2f83-f99a-46d1-b2c2-2e47bae4bb90 |      | fa:16:3e:ff:94:28 | {"subnet_id": "d50e28f7-47ee-4bdf-8594-e1108f25586b", "ip_address": "192.168.137.102"} |
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------+

[root@rdoicehouse ~(keystone_admin)]# neutron port-delete 767f2f83-f99a-46d1-b2c2-2e47bae4bb90
Deleted port: 767f2f83-f99a-46d1-b2c2-2e47bae4bb90

[root@rdoicehouse ~(keystone_admin)]# neutron port-delete 4655e13a-9767-4750-8f44-4eee8410ca70
409-{u'NeutronError': {u'message': u'Port 4655e13a-9767-4750-8f44-4eee8410ca70 has owner network:floatingip and therefore cannot be deleted directly via the port API.', u'type': u'L3PortInUse', u'detail': u''}}

[root@rdoicehouse ~(keystone_admin)]# neutron floatingip-list
+--------------------------------------+------------------+---------------------+---------+
| id                                   | fixed_ip_address | floating_ip_address | port_id |
+--------------------------------------+------------------+---------------------+---------+
| 0a74679b-b469-4ae5-97a0-08c3aeeb2129 |                  | 192.168.137.103     |         |
+--------------------------------------+------------------+---------------------+---------+

[root@rdoicehouse ~(keystone_admin)]# neutron floatingip-delete 0a74679b-b469-4ae5-97a0-08c3aeeb2129
Deleted floatingip: 0a74679b-b469-4ae5-97a0-08c3aeeb2129

[root@rdoicehouse ~(keystone_admin)]# neutron port-list

[root@rdoicehouse ~(keystone_admin)]# neutron subnet-delete d50e28f7-47ee-4bdf-8594-e1108f25586b
Deleted subnet: d50e28f7-47ee-4bdf-8594-e1108f25586b

[root@rdoicehouse ~(keystone_admin)]# neutron net-list
+--------------------------------------+------------+--------------------------------------------------+
| id                                   | name       | subnets                                          |
+--------------------------------------+------------+--------------------------------------------------+
| 77dd5a93-b63e-44be-84d6-f6ef4fd8771b | private    | c93fd5a7-d672-4b0c-8f2e-6e74f487e45d 10.0.0.0/24 |
| 8b2ceda2-4d77-4c5c-ae21-6a7ba133e4fc | demodelete |                                                  |
+--------------------------------------+------------+--------------------------------------------------+
[root@rdoicehouse ~(keystone_admin)]# neutron net-delete 8b2ceda2-4d77-4c5c-ae21-6a7ba133e4fc
Deleted network: 8b2ceda2-4d77-4c5c-ae21-6a7ba133e4fc
```

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron router-gateway-clear e34d94ad-7fe1-4704-8156-d255a2daa167 Removed gateway from router e34d94ad-7fe1-4704-8156-d255a2daa167

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron router-port-list e34d94ad-7fe1-4704-8156-d255a2daa167

If Apply: \[\[ neutron router-interface-delete   \]\]

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron router-delete e34d94ad-7fe1-4704-8156-d255a2daa167 Deleted router: e34d94ad-7fe1-4704-8156-d255a2daa167

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron subnet-list +--------------------------------------+------------------+------------------+--------------------------------------------------------+ \| id \| name \| cidr \| allocation\_pools \| +--------------------------------------+------------------+------------------+--------------------------------------------------------+ \| d50e28f7-47ee-4bdf-8594-e1108f25586b \| demosubnetdelete \| 192.168.137.0/24 \| {"start": "192.168.137.100", "end": "192.168.137.120"} \| \| c93fd5a7-d672-4b0c-8f2e-6e74f487e45d \| private\_subnet \| 10.0.0.0/24 \| {"start": "10.0.0.2", "end": "10.0.0.254"} \| +--------------------------------------+------------------+------------------+--------------------------------------------------------+ \[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron subnet-delete d50e28f7-47ee-4bdf-8594-e1108f25586b 409-{u'NeutronError': {u'message': u'Unable to complete operation on subnet d50e28f7-47ee-4bdf-8594-e1108f25586b. One or more ports have an IP allocation from this subnet.', u'type': u'SubnetInUse', u'detail': u''}}

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron port-list +--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------+ \| id \| name \| mac\_address \| fixed\_ips \| +--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------+ \| 4655e13a-9767-4750-8f44-4eee8410ca70 \| \| fa:16:3e:03:c7:cf \| {"subnet\_id": "d50e28f7-47ee-4bdf-8594-e1108f25586b", "ip\_address": "192.168.137.103"} \| \| 767f2f83-f99a-46d1-b2c2-2e47bae4bb90 \| \| fa:16:3e:ff:94:28 \| {"subnet\_id": "d50e28f7-47ee-4bdf-8594-e1108f25586b", "ip\_address": "192.168.137.102"} \| +--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------+

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron port-delete 767f2f83-f99a-46d1-b2c2-2e47bae4bb90 Deleted port: 767f2f83-f99a-46d1-b2c2-2e47bae4bb90

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron port-delete 4655e13a-9767-4750-8f44-4eee8410ca70 409-{u'NeutronError': {u'message': u'Port 4655e13a-9767-4750-8f44-4eee8410ca70 has owner network:floatingip and therefore cannot be deleted directly via the port API.', u'type': u'L3PortInUse', u'detail': u''}}

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron floatingip-list +--------------------------------------+------------------+---------------------+---------+ \| id \| fixed\_ip\_address \| floating\_ip\_address \| port\_id \| +--------------------------------------+------------------+---------------------+---------+ \| 0a74679b-b469-4ae5-97a0-08c3aeeb2129 \| \| 192.168.137.103 \| \| +--------------------------------------+------------------+---------------------+---------+

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron floatingip-delete 0a74679b-b469-4ae5-97a0-08c3aeeb2129 Deleted floatingip: 0a74679b-b469-4ae5-97a0-08c3aeeb2129

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron port-list

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron subnet-delete d50e28f7-47ee-4bdf-8594-e1108f25586b Deleted subnet: d50e28f7-47ee-4bdf-8594-e1108f25586b

\[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron net-list +--------------------------------------+------------+--------------------------------------------------+ \| id \| name \| subnets \| +--------------------------------------+------------+--------------------------------------------------+ \| 77dd5a93-b63e-44be-84d6-f6ef4fd8771b \| private \| c93fd5a7-d672-4b0c-8f2e-6e74f487e45d 10.0.0.0/24 \| \| 8b2ceda2-4d77-4c5c-ae21-6a7ba133e4fc \| demodelete \| \| +--------------------------------------+------------+--------------------------------------------------+ \[root@rdoicehouse ~\(keystone\_admin\)\]\# neutron net-delete 8b2ceda2-4d77-4c5c-ae21-6a7ba133e4fc Deleted network: 8b2ceda2-4d77-4c5c-ae21-6a7ba133e4fc

