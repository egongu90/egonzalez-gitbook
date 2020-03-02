# OpenStack segregation with availability zones and host aggregates

When a new OpenStack cloud born, usually all servers run over the same hardware and specifications, often, all servers are in the same building, room, rack, even a chassis when the cloud is in the first growth paces.

After a while, workloads increase and the current hardware is not enough to process that workloads. At this point, your hardware is old and new hardware is bought. This hardware has different storage disks, CPU, RAM and so on. You passed from 10\'s of servers to 100\'s. DataCenter racks, rooms and buildings are too small and the growing cloud needs redundancy between cities or countries.

OpenStack offers a few solutions for that purpose, called Regions, Cells, Availability Zones and Host Aggregates. \| Now, we are going to focus on Availability Zones and Host Aggregates, which are the way to segregate computational workloads.

* Host Aggregates:
  * Host Aggregates represent a logical set of

    properties/characteristics a group of hosts owns in the form of

    metadata. Imagine some of your servers have SSD disks and the

    other ones SATA, you can map those properties SSD/SATA to a

    group of hosts, when a image or flavor with the metaparameter

    associated is launched, Nova Scheduler will filter the available

    hosts with the meta parameter value and boot the instance on

    hosts with the desired property. Host Aggregates are managed by

    OpenStack admins.
* Availability Zones
  * Availability Zones represent a logical partition of the

    infrastructure\(not necessary but is the common use case\) in the

    form of racks, rooms, buildings, etc. Customers can launch

    instances in the desired Availability Zone.

Usually, Host Aggregates are mapped to Availability Zones allowing customers to use the desired set of hardware or characteristics to boot instances.

At the end of this guide you will know how to:

1. Create Availability Zones and Host Aggregates.
2. Adding hosts to Host Aggregates and Availability Zones.
3. Launch instances directly to Availability Zones.
4. Configure nova scheduler for Host Aggregates usage.
5. Configure Images and Flavors for scheduling to Host Aggregates.
6. Launch instances based on flavors and image parameters.

Let's start: \00/

\| Create two Host Aggregate called `"az1-ag"/"az2-ag"`, this command also, will create two Availability Zones called `"az1"/"az2"`. \| By default, when a Host Aggregate is created with an Availability Zone, a metadata key called `"availability_zone=NAME_OF_AZ`" will be created.

```text
# nova aggregate-create az1-ag az1
+----+--------+-------------------+-------+-------------------------+
| Id | Name   | Availability Zone | Hosts | Metadata                |
+----+--------+-------------------+-------+-------------------------+
| 2  | az1-ag | az1               |       | 'availability_zone=az1' |
+----+--------+-------------------+-------+-------------------------+
# nova aggregate-create az2-ag az2
+----+--------+-------------------+-------+-------------------------+
| Id | Name   | Availability Zone | Hosts | Metadata                |
+----+--------+-------------------+-------+-------------------------+
| 3  | az2-ag | az2               |       | 'availability_zone=az2' |
+----+--------+-------------------+-------+-------------------------+
```

Add one or more compute nodes to Host Aggregates.

```text
# nova aggregate-add-host 2 compute1az
Host compute1az has been successfully added for aggregate 2 
+----+--------+-------------------+--------------+-------------------------+
| Id | Name   | Availability Zone | Hosts        | Metadata                |
+----+--------+-------------------+--------------+-------------------------+
| 2  | az1-ag | az1               | 'compute1az' | 'availability_zone=az1' |
+----+--------+-------------------+--------------+-------------------------+
# nova aggregate-add-host 3 compute2az
Host compute2az has been successfully added for aggregate 3 
+----+--------+-------------------+--------------+-------------------------+
| Id | Name   | Availability Zone | Hosts        | Metadata                |
+----+--------+-------------------+--------------+-------------------------+
| 3  | az2-ag | az2               | 'compute2az' | 'availability_zone=az2' |
+----+--------+-------------------+--------------+-------------------------+
```

Details about a Host Aggregate can be reviewed with:

```text
# nova aggregate-details az1-ag
+----+--------+-------------------+--------------+-------------------------+
| Id | Name   | Availability Zone | Hosts        | Metadata                |
+----+--------+-------------------+--------------+-------------------------+
| 2  | az1-ag | az1               | 'compute1az' | 'availability_zone=az1' |
+----+--------+-------------------+--------------+-------------------------+
# nova aggregate-details az2-ag
+----+--------+-------------------+--------------+-------------------------+
| Id | Name   | Availability Zone | Hosts        | Metadata                |
+----+--------+-------------------+--------------+-------------------------+
| 3  | az2-ag | az2               | 'compute2az' | 'availability_zone=az2' |
+----+--------+-------------------+--------------+-------------------------+
```

List Availability Zones and check status.

```text
# nova availability-zone-list
+-----------------------+----------------------------------------+
| Name                  | Status                                 |
+-----------------------+----------------------------------------+
| internal              | available                              |
| |- controlleraz       |                                        |
| | |- nova-conductor   | enabled :-) 2016-01-14T19:08:16.000000 |
| | |- nova-consoleauth | enabled :-) 2016-01-14T19:08:16.000000 |
| | |- nova-scheduler   | enabled :-) 2016-01-14T19:08:16.000000 |
| | |- nova-cert        | enabled :-) 2016-01-14T19:08:13.000000 |
| az2                   | available                              |
| |- compute2az         |                                        |
| | |- nova-compute     | enabled :-) 2016-01-14T19:08:12.000000 |
| az1                   | available                              |
| |- compute1az         |                                        |
| | |- nova-compute     | enabled :-) 2016-01-14T19:08:12.000000 |
+-----------------------+----------------------------------------+
```

Other method you can use:

```text
# nova service-list
+----+------------------+--------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host         | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+--------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-consoleauth | controlleraz | internal | enabled | up    | 2016-01-14T19:54:36.000000 | -               |
| 2  | nova-scheduler   | controlleraz | internal | enabled | up    | 2016-01-14T19:54:36.000000 | -               |
| 3  | nova-conductor   | controlleraz | internal | enabled | up    | 2016-01-14T19:54:35.000000 | -               |
| 4  | nova-cert        | controlleraz | internal | enabled | up    | 2016-01-14T19:54:33.000000 | -               |
| 5  | nova-compute     | compute2az   | az2      | enabled | up    | 2016-01-14T19:54:32.000000 | -               |
| 6  | nova-compute     | compute1az   | az1      | enabled | up    | 2016-01-14T19:54:32.000000 | -               |
+----+------------------+--------------+----------+---------+-------+----------------------------+-----------------+
```

Launch two instances using `"--availability-zone AZ`" option, you can even select the compute node to use, just use `"--availability-zone AZ:COMPUTE_NODE`".

```text
# nova boot --flavor m1.tiny --image cirros --nic net-id=6d62149e-74d3-4e52-9813-53ad207309f4 --availability-zone az1 instanceaz1
# nova boot --flavor m1.tiny --image cirros --nic net-id=6d62149e-74d3-4e52-9813-53ad207309f4 --availability-zone az2 instanceaz2
```

Ensure the instances are running in the desired Availability Zone.

```text
# nova show instanceaz1 | grep OS-EXT-AZ | awk '{print$2":"$4}'
OS-EXT-AZ:availability_zone:az1
# nova show instanceaz2 | grep OS-EXT-AZ | awk '{print$2":"$4}'
OS-EXT-AZ:availability_zone:az2
```

List Glance images.

```text
# glance image-list
+--------------------------------------+----------+
| ID                                   | Name     |
+--------------------------------------+----------+
| a6d7a606-f725-480a-9b1b-7b3ae39b93d4 | cirros   |
| a6540d72-dff7-4fb1-bc64-a8ea69e65178 | imageaz1 |
| 9c7e2d55-0b96-43e2-9231-88e426edb350 | imageaz2 |
+--------------------------------------+----------+
```

Update the images with custom properties, i use `"availability_zone"` because is the default meta parameter a Host Aggregate owns when is inside Availability Zones.

```text
# glance image-update --property availability_zone=az1 a6540d72-dff7-4fb1-bc64-a8ea69e65178
+-------------------+--------------------------------------+
| Property          | Value                                |
+-------------------+--------------------------------------+
| availability_zone | az1                                  |
| checksum          | 133eae9fb1c98f45894a4e60d8736619     |
| container_format  | bare                                 |
| created_at        | 2016-01-14T19:59:04Z                 |
| disk_format       | qcow2                                |
| id                | a6540d72-dff7-4fb1-bc64-a8ea69e65178 |
| min_disk          | 0                                    |
| min_ram           | 0                                    |
| name              | imageaz1                             |
| owner             | 0571c6769c3f46acb195eeb01b87ae38     |
| protected         | False                                |
| size              | 13200896                             |
| status            | active                               |
| tags              | []                                   |
| updated_at        | 2016-01-14T20:13:05Z                 |
| virtual_size      | None                                 |
| visibility        | private                              |
+-------------------+--------------------------------------+
# glance image-update --property availability_zone=az2 9c7e2d55-0b96-43e2-9231-88e426edb350
+-------------------+--------------------------------------+
| Property          | Value                                |
+-------------------+--------------------------------------+
| availability_zone | az2                                  |
| checksum          | 133eae9fb1c98f45894a4e60d8736619     |
| container_format  | bare                                 |
| created_at        | 2016-01-14T19:59:10Z                 |
| disk_format       | qcow2                                |
| id                | 9c7e2d55-0b96-43e2-9231-88e426edb350 |
| min_disk          | 0                                    |
| min_ram           | 0                                    |
| name              | imageaz2                             |
| owner             | 0571c6769c3f46acb195eeb01b87ae38     |
| protected         | False                                |
| size              | 13200896                             |
| status            | active                               |
| tags              | []                                   |
| updated_at        | 2016-01-14T20:13:27Z                 |
| virtual_size      | None                                 |
| visibility        | private                              |
+-------------------+--------------------------------------+
```

Boot two instances, now we use images with custom properties, those properties will map to Availability Zones\(you can use other type of parameters mapping to Host Aggregates characteristics\).

```text
# nova boot --flavor m1.tiny --image imageaz1 --nic net-id=6d62149e-74d3-4e52-9813-53ad207309f4 instanceimageaz1
# nova boot --flavor m1.tiny --image imageaz2 --nic net-id=6d62149e-74d3-4e52-9813-53ad207309f4 instanceimageaz2
```

Ensure the instances booted in the desired Availability Zone.

```text
# nova show instanceimageaz1 | grep OS-EXT-AZ | awk '{print$2":"$4}'
OS-EXT-AZ:availability_zone:az1
# nova show instanceimageaz2 | grep OS-EXT-AZ | awk '{print$2":"$4}'
OS-EXT-AZ:availability_zone:az2
```

\| Other method to launch instances is with parameters in flavors. \| Create two flavors.

```text
# nova flavor-create --is-public true flavoraz1 6 512 1 1
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| 7  | flavoraz1 | 512       | 1    | 0         |      | 1     | 1.0         | True      |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
# nova flavor-create --is-public true flavoraz2 7 512 1 1
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| 8  | flavoraz2 | 512       | 1    | 0         |      | 1     | 1.0         | True      |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
```

Add metadata to a Host Aggregate with some characteristic property as can be fast HD or cheap HW.

```text
# nova aggregate-set-metadata az1-ag fast=true
Metadata has been successfully updated for aggregate 2.
+----+--------+-------------------+--------------+--------------------------------------+
| Id | Name   | Availability Zone | Hosts        | Metadata                             |
+----+--------+-------------------+--------------+--------------------------------------+
| 2  | az1-ag | az1               | 'compute1az' | 'availability_zone=az1', 'fast=true' |
+----+--------+-------------------+--------------+--------------------------------------+
# nova aggregate-set-metadata az2-ag cheap=true
Metadata has been successfully updated for aggregate 3.
+----+--------+-------------------+--------------+---------------------------------------+
| Id | Name   | Availability Zone | Hosts        | Metadata                              |
+----+--------+-------------------+--------------+---------------------------------------+
| 3  | az2-ag | az2               | 'compute2az' | 'availability_zone=az2', 'cheap=true' |
+----+--------+-------------------+--------------+---------------------------------------+
```

Update the previous created flavors with the associated metadata key with the Host Aggregate.

```text
# nova flavor-key flavoraz1 set  aggregate_instance_extra_specs:fast=true
# nova flavor-key flavoraz2 set  aggregate_instance_extra_specs:cheap=true
```

Ensure, the properties are properly created.

```text
# nova flavor-show 7 | grep fast | awk '{print$4$5}'
{"aggregate_instance_extra_specs:fast":"true"}
# nova flavor-show 8 | grep cheap | awk '{print$4$5}'
{"aggregate_instance_extra_specs:cheap":"true"}
```

By default, Nova Scheduler don't allow filtering by extra Specs inserted in flavors or images.

First, ensure the following scheduler filters are allowed in Control nodes.

```text
# egrep ^scheduler_default_filters /etc/nova/nova.conf 
scheduler_default_filters=AggregateInstanceExtraSpecsFilter,RetryFilter,AvailabilityZoneFilter,RamFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter
```

If a change has been done in nova.conf file, restart nova services

```text
# openstack-service restart nova
```

Boot another two instances, now using custom flavors

```text
# nova boot --flavor flavoraz1 --image cirros --nic net-id=6d62149e-74d3-4e52-9813-53ad207309f4 instanceflavoraz1
# nova boot --flavor flavoraz2 --image cirros --nic net-id=6d62149e-74d3-4e52-9813-53ad207309f4 instanceflavoraz2
```

Check where the instances are running.

```text
# nova show instanceflavoraz1 | grep OS-EXT-AZ | awk '{print$2":"$4}'
OS-EXT-AZ:availability_zone:az1
# nova show instanceflavoraz2 | grep OS-EXT-AZ | awk '{print$2":"$4}'
OS-EXT-AZ:availability_zone:az2
```

That's all for now \| Hope this guide helps.

Regards

