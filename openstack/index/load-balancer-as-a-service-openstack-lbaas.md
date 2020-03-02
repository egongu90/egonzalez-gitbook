# Load balancer as a service OpenStack LbaaS

The following guide will show you how to deploy a LoadBalancer in Openstack with Neutron, but first, you should understand how it works, and what his components do.

A Load Balancer is composed of the following components:

* Pool - A pool is a group of servers\(members\) who are designed to make

  the same job, generally, a pool of web servers is used for balancing

  traffic between the members of the pool. Here we will configure the

  Load Balancing Method \(ROUND\_ROBIN,LEAST\_CONNECTIONS,SOURCE\_IP\)

* Members - Members are instances, a server, any aplication that you

  can balance the load. They are assigned as pool members.

* VIP - VIPs are Virtual IPs that logically represents the pool

  members. It is the IP where the load will be balanced between

  instances.

* Healthmonitor - Healthmonitor will check if the members of a pool are

  healthy, if an member is not working or the port/protocol monitored

  is down, healthmonitor will send a message to the pool to not balance

  the load to this member.

Now will create a Pool with 2 members, this Pool have a VIP and a Healthmonitor on it.

First we create a Pool

```text
[stack@localhost devstack]$ neutron lb-pool-create --lb-method ROUND_ROBIN --name LoadBalancerPool --protocol HTTP --subnet-id e5a90ab2-918e-412b-9723-0d822804f022
Created a new pool:
+------------------------+--------------------------------------+
| Field                  | Value                                |
+------------------------+--------------------------------------+
| admin_state_up         | True                                 |
| description            |                                      |
| health_monitors        |                                      |
| health_monitors_status |                                      |
| id                     | 3eb0d41c-3df5-4beb-9758-ebfef56909df |
| lb_method              | ROUND_ROBIN                          |
| members                |                                      |
| name                   | LoadBalancerPool                     |
| protocol               | HTTP                                 |
| provider               | haproxy                              |
| status                 | PENDING_CREATE                       |
| status_description     |                                      |
| subnet_id              | e5a90ab2-918e-412b-9723-0d822804f022 |
| tenant_id              | b1aaddea9f694e60aea5f1c0d1dd7c24     |
| vip_id                 |                                      |
+------------------------+--------------------------------------+
```

Next boot 2 instances in the same network

```text
[stack@localhost devstack]$ nova boot --flavor m1.tiny --image 6a3a7880-bc6f-454d-9a62-d9c2d268ef78 --security-groups default --nic net-id=daddce32-b6e8-4e3f-bd55-32459ed327ea WebServer1
[stack@localhost devstack]$ nova boot --flavor m1.tiny --image 6a3a7880-bc6f-454d-9a62-d9c2d268ef78 --security-groups default --nic net-id=daddce32-b6e8-4e3f-bd55-32459ed327ea WebServer2

[stack@localhost devstack]$ nova list
+--------------------------------------+------------+--------+------------+-------------+------------------+
| ID                                   | Name       | Status | Task State | Power State | Networks         |
+--------------------------------------+------------+--------+------------+-------------+------------------+
| c10e63c6-f342-4d1c-ae22-146c392ce398 | WebServer1 | BUILD  | spawning   | NOSTATE     | private=10.0.0.3 |
| ceef9e6b-6198-4118-8027-00898dee1abe | WebServer2 | BUILD  | spawning   | NOSTATE     | private=10.0.0.4 |
+--------------------------------------+------------+--------+------------+-------------+------------------+
```

Assign both instances to the Pool

```text
[stack@localhost devstack]$ neutron lb-member-create --address 10.0.0.3 --protocol-port 80 LoadBalancerPool
Created a new member:
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| address            | 10.0.0.3                             |
| admin_state_up     | True                                 |
| id                 | a6de6bf0-3191-4721-aa01-5781ff05876e |
| pool_id            | 3eb0d41c-3df5-4beb-9758-ebfef56909df |
| protocol_port      | 80                                   |
| status             | PENDING_CREATE                       |
| status_description |                                      |
| tenant_id          | b1aaddea9f694e60aea5f1c0d1dd7c24     |
| weight             | 1                                    |
+--------------------+--------------------------------------+

[stack@localhost devstack]$ neutron lb-member-create --address 10.0.0.4 --protocol-port 80 LoadBalancerPool
Created a new member:
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| address            | 10.0.0.4                             |
| admin_state_up     | True                                 |
| id                 | 9688a770-6494-4599-88fa-6afcd18c4dd1 |
| pool_id            | 3eb0d41c-3df5-4beb-9758-ebfef56909df |
| protocol_port      | 80                                   |
| status             | PENDING_CREATE                       |
| status_description |                                      |
| tenant_id          | b1aaddea9f694e60aea5f1c0d1dd7c24     |
| weight             | 1                                    |
+--------------------+--------------------------------------+
```

Then create a Healthmonitor and associate it to the Pool

```text
[stack@localhost devstack]$ neutron lb-healthmonitor-create --timeout 3 --max-retries 3 --delay 60 --type HTTP
Created a new health_monitor:
+----------------+--------------------------------------+
| Field          | Value                                |
+----------------+--------------------------------------+
| admin_state_up | True                                 |
| delay          | 60                                   |
| expected_codes | 200                                  |
| http_method    | GET                                  |
| id             | cb73f8fd-14ea-4937-aa10-019e3da8432f |
| max_retries    | 3                                    |
| pools          |                                      |
| tenant_id      | b1aaddea9f694e60aea5f1c0d1dd7c24     |
| timeout        | 3                                    |
| type           | HTTP                                 |
| url_path       | /                                    |
+----------------+--------------------------------------+
[stack@localhost devstack]$ neutron lb-healthmonitor-associate cb73f8fd-14ea-4937-aa10-019e3da8432f LoadBalancerPool
Associated health monitor cb73f8fd-14ea-4937-aa10-019e3da8432f
```

Create a VIP to the Pool

```text
[stack@localhost devstack]$ neutron lb-vip-create --name LoadBalancerVIP --protocol-port 80 --protocol HTTP --subnet-id e5a90ab2-918e-412b-9723-0d822804f022 LoadBalancerPool
Created a new vip:
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| address             | 10.0.0.5                             |
| admin_state_up      | True                                 |
| connection_limit    | -1                                   |
| description         |                                      |
| id                  | 4e3c2b84-a286-4999-a258-51c44965a81a |
| name                | LoadBalancerVIP                      |
| pool_id             | 3eb0d41c-3df5-4beb-9758-ebfef56909df |
| port_id             | d4ed46ac-aabf-40b6-8f28-1a2013971391 |
| protocol            | HTTP                                 |
| protocol_port       | 80                                   |
| session_persistence |                                      |
| status              | PENDING_CREATE                       |
| status_description  |                                      |
| subnet_id           | e5a90ab2-918e-412b-9723-0d822804f022 |
| tenant_id           | b1aaddea9f694e60aea5f1c0d1dd7c24     |
+---------------------+--------------------------------------+
```

Create a floating IP to the VIP

```text
[stack@localhost devstack]$ neutron floatingip-create 23101147-e724-4574-82c7-a05ccb661d4d
Created a new floatingip:
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| fixed_ip_address    |                                      |
| floating_ip_address | 172.24.4.3                           |
| floating_network_id | 23101147-e724-4574-82c7-a05ccb661d4d |
| id                  | 62fbf609-77db-4471-b6ae-9fe25a091a21 |
| port_id             |                                      |
| router_id           |                                      |
| status              | DOWN                                 |
| tenant_id           | b1aaddea9f694e60aea5f1c0d1dd7c24     |
+---------------------+--------------------------------------+
```

Associate the floating IP with the VIP port

```text
[stack@localhost devstack]$ neutron floatingip-associate 62fbf609-77db-4471-b6ae-9fe25a091a21 d4ed46ac-aabf-40b6-8f28-1a2013971391
Associated floating IP 62fbf609-77db-4471-b6ae-9fe25a091a21
```

Create security rules to allow HTTP, SSH and ICMP traffic

```text
[stack@localhost devstack]$ neutron security-group-rule-create --protocol TCP --port-range-min 80 --port-range-max 80 be0b2264-744a-48b8-9a1e-033227d78f2b
Created a new security_group_rule:
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| id                | 4635cbb6-d939-40b3-ac11-637c8b63b027 |
| port_range_max    | 80                                   |
| port_range_min    | 80                                   |
| protocol          | tcp                                  |
| remote_group_id   |                                      |
| remote_ip_prefix  |                                      |
| security_group_id | be0b2264-744a-48b8-9a1e-033227d78f2b |
| tenant_id         | b1aaddea9f694e60aea5f1c0d1dd7c24     |
+-------------------+--------------------------------------+

[stack@localhost devstack]$ neutron security-group-rule-create --protocol icmp be0b2264-744a-48b8-9a1e-033227d78f2b
Created a new security_group_rule:
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| id                | 988329a1-d686-4541-8950-a22c721f847b |
| port_range_max    |                                      |
| port_range_min    |                                      |
| protocol          | icmp                                 |
| remote_group_id   |                                      |
| remote_ip_prefix  |                                      |
| security_group_id | be0b2264-744a-48b8-9a1e-033227d78f2b |
| tenant_id         | b1aaddea9f694e60aea5f1c0d1dd7c24     |
+-------------------+--------------------------------------+

[stack@localhost devstack]$ neutron security-group-rule-create --protocol TCP --port-range-min 22 --port-range-max 22 be0b2264-744a-48b8-9a1e-033227d78f2b
Created a new security_group_rule:
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| id                | d18724dc-2eda-4031-be88-202a73c30c24 |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| protocol          | tcp                                  |
| remote_group_id   |                                      |
| remote_ip_pref                          |
| security_group_id | d7412bb3-9824-4eb7-bc4b-cd80ab6a570d |
| tenant_id         | b1aaddea9f694e60aea5f1c0d1dd7c24     |
+-------------------+--------------------------------------+
```

Login to both instances and run the command below to run a "webserver".

```text
[stack@localhost devstack]$ ssh cirros@INSTANCEIP
The authenticity of host '10.0.0.3 (10.0.0.3)' can't be established.
RSA key fingerprint is 94:00:8e:fe:9a:9d:af:ef:bc:e3:fd:9d:ad:d3:ab:a3.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.0.0.3' (RSA) to the list of known hosts.

$ while true; do echo -e 'HTTP/1.0 200 OK \r\n\r\nServer1' | sudo nc -l -p 80 ; done
$ while true; do echo -e 'HTTP/1.0 200 OK \r\n\r\nServer2' | sudo nc -l -p 80 ; done
```

If we check with curl the VIP's floating IP, we'll see that in every connection one of both servers reply with his name.

```text
[stack@localhost ~]$ curl http://172.24.4.3
Server1
[stack@localhost ~]$ curl http://172.24.4.3
Server2
[stack@localhost ~]$ curl http://172.24.4.3
Server1
[stack@localhost ~]$ curl http://172.24.4.3
Server2
```

