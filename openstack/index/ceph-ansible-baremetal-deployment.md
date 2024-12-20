# Ceph Ansible baremetal deployment

How many times you tried to install Ceph? How many fails with no reason?&#x20;

All Ceph operator should agree with me when i say that Ceph installer doesn't really works as expected so far.&#x20;

Yes, i'm talking about ceph-deploy and the main reason why i'm posting this guide about deploying Ceph with Ansible.

At this post, i will show how to install a Ceph cluster with Ansible on baremetal servers.&#x20;

My configuration is as follows:

1. 3 x ceph monitors 8GB of RAM each one
2. 3 x OSD nodes 16GB of RAM and 3x100 GB of Disk
3. 1 x RadosGateway node 8GB of RAM

First, download Ceph-Ansible playbooks

```
git clone https://github.com/ceph/ceph-ansible/
Cloning into 'ceph-ansible'...
remote: Counting objects: 5764, done.
remote: Compressing objects: 100% (38/38), done.
remote: Total 5764 (delta 7), reused 0 (delta 0), pack-reused 5726
Receiving objects: 100% (5764/5764), 1.12 MiB | 1.06 MiB/s, done.
Resolving deltas: 100% (3465/3465), done.
Checking connectivity... done.
```

Move to the newly created folder called ceph-ansible

```
cd ceph-ansible/
```

Copy sample vars files, we will configure our environment in these variable files.

```
cp site.yml.sample site.yml
cp group_vars/all.sample group_vars/all
cp group_vars/mons.sample group_vars/mons
cp group_vars/osds.sample group_vars/osds
```

Next step is configure the inventory with our servers, i don\\'t really like use /etc/ansible/host file, i prefer create a new file per environment inside playbook\\'s folder.

Create a file with the following content, use you own IPs to match your servers on the desired role inside the cluster

```
[root@ansible ~]# vi inventory_hosts

[mons]
192.168.1.48
192.168.1.49
192.168.1.52

[osds]
192.168.1.50
192.168.1.53
192.168.1.54

[rgws]
192.168.1.55
```

Test connectivity to you servers pinging them through Ansible ping module

```
[root@ansible ~]# ansible -m ping -i inventory_hosts all
192.168.1.48 | success >> {
    "changed": false,
    "ping": "pong"
}

192.168.1.50 | success >> {
    "changed": false,
    "ping": "pong"
}

192.168.1.55 | success >> {
    "changed": false,
    "ping": "pong"
}

192.168.1.53 | success >> {
    "changed": false,
    "ping": "pong"
}

192.168.1.49 | success >> {
    "changed": false,
    "ping": "pong"
}

192.168.1.54 | success >> {
    "changed": false,
    "ping": "pong"
}

192.168.1.52 | success >> {
    "changed": false,
    "ping": "pong"
}
```

Edit site.yml file, i will remove/comment mds nodes since i\\'m not going to use them.

```
[root@ansible ~]# vi site.yml

- hosts: mons
  become: True
  roles:
  - ceph-mon

- hosts: agents
  become: True
  roles:
  - ceph-agent

- hosts: osds
  become: True
  roles:
  - ceph-osd

#- hosts: mdss
#  become: True
#  roles:
#  - ceph-mds

- hosts: rgws
  become: True
  roles:
  - ceph-rgw

- hosts: restapis
  become: True
  roles:
  - ceph-restapi
```

Edit main variable file, here we are going to configure our environment

```
[root@ansible ~]# vi group_vars/all
```

Here we configure from where ceph packages are going to be installed, for now we use upstream code with the stable release Infernalis.

```
## Configure package origin
ceph_origin: upstream
ceph_stable: true
ceph_stable_release: infernalis
```

Configure interface on which monitor will be listening

```
## Monitor options
monitor_interface: eth2
```

Here we configure some OSD options, like journal size and what networks will be used by public and cluster data replication

```
## OSD options
journal_size: 1024
public_network: 192.168.1.0/24
cluster_network: 192.168.200.0/24
```

Edit osds variable file

```
[root@ansible ~]# vi group_vars/osds
```

I will use auto discovery option to allow ceph ansible select empy or not used devices in my servers to create OSDs.

```
# Declare devices
osd_auto_discovery: True
journal_collocation: True
```

\| Of course you can use other options, i\\'ll highly suggest you to read variable comments, as they provide valuable information about usage. | We\\'re ready to deploy ceph with ansible with our custom inventory\_hosts file.

```
[root@ansible ~]# ansible-playbook site.yml -i inventory_hosts
```

After a while, you will have a fully functional ceph cluster.

\| Maybe you find some issues or bugs when running the playbooks. | There is a lot of efforts to fix issues on upstream repository. If a new bug is encountered, please, post a issue right here. | [https://github.com/ceph/ceph-ansible/issues](https://github.com/ceph/ceph-ansible/issues)

You can check your cluster status with ceph -s. we can see all OSDs are up and pgs active/clean.

```
[root@ceph-mon1 ~]# ceph -s
    cluster 5ff692ab-2150-41a4-8b6d-001a4da21c9c
     health HEALTH_OK
     monmap e1: 3 mons at {ceph-mon1=192.168.200.141:6789/0,ceph-mon2=192.168.200.180:6789/0,ceph-mon3=192.168.200.232:6789/0}
            election epoch 6, quorum 0,1,2 ceph-mon1,ceph-mon2,ceph-mon3
     osdmap e10: 9 osds: 9 up, 9 in
            flags sortbitwise
      pgmap v32: 64 pgs, 1 pools, 0 bytes data, 0 objects
            102256 kB used, 896 GB / 896 GB avail
                  64 active+clean
```

\| We are going to do some tests. | Create a pool

```
[root@ceph-mon1 ~]# ceph osd pool create test 128 128
pool 'test' created
```

Create a file big file

```
[root@ceph-mon1 ~]# dd if=/dev/zero of=/tmp/sample.txt bs=2M count=1000
1000+0 records in
1000+0 records out
2097152000 bytes (2.1 GB) copied, 16.7386 s, 125 MB/s
```

Upload the file to rados

```
[root@ceph-mon1 ~]# rados -p test put sample /tmp/sample.txt 
```

Check om which placement groups your file is saved

```
[root@ceph-mon1 ~]# ceph osd map test sample
osdmap e13 pool 'test' (1) object 'sample' -> pg 1.bddbf0b9 (1.39) -> up ([1,0], p1) acting ([1,0], p1)
```

Query the placement group where you file was uploaded, a similar output will prompts

```
[root@ceph-mon1 ~]# ceph pg 1.39 query
{
    "state": "active+clean",
    "snap_trimq": "[]",
    "epoch": 13,
    "up": [
        1,
        0
    ],
    "acting": [
        1,
        0
    ],
    "actingbackfill": [
        "0",
        "1"
    ],
    "info": {
        "pgid": "1.39",
        "last_update": "13'500",
        "last_complete": "13'500",
        "log_tail": "0'0",
        "last_user_version": 500,
        "last_backfill": "MAX",
        "last_backfill_bitwise": 0,
        "purged_snaps": "[]",
        "history": {
            "epoch_created": 11,
            "last_epoch_started": 12,
            "last_epoch_clean": 13,
            "last_epoch_split": 0,
            "last_epoch_marked_full": 0,
            "same_up_since": 11,
            "same_interval_since": 11,
            "same_primary_since": 11,
            "last_scrub": "0'0",
            "last_scrub_stamp": "2016-03-16 21:13:08.883121",
            "last_deep_scrub": "0'0",
            "last_deep_scrub_stamp": "2016-03-16 21:13:08.883121",
            "last_clean_scrub_stamp": "0.000000"
        },
        "stats": {
            "version": "13'500",
            "reported_seq": "505",
            "reported_epoch": "13",
            "state": "active+clean",
            "last_fresh": "2016-03-16 21:24:40.930724",
            "last_change": "2016-03-16 21:14:09.874086",
            "last_active": "2016-03-16 21:24:40.930724",
            "last_peered": "2016-03-16 21:24:40.930724",
            "last_clean": "2016-03-16 21:24:40.930724",
            "last_became_active": "0.000000",
            "last_became_peered": "0.000000",
            "last_unstale": "2016-03-16 21:24:40.930724",
            "last_undegraded": "2016-03-16 21:24:40.930724",
            "last_fullsized": "2016-03-16 21:24:40.930724",
            "mapping_epoch": 11,
            "log_start": "0'0",
            "ondisk_log_start": "0'0",
            "created": 11,
            "last_epoch_clean": 13,
            "parent": "0.0",
            "parent_split_bits": 0,
            "last_scrub": "0'0",
            "last_scrub_stamp": "2016-03-16 21:13:08.883121",
            "last_deep_scrub": "0'0",
            "last_deep_scrub_stamp": "2016-03-16 21:13:08.883121",
            "last_clean_scrub_stamp": "0.000000",
            "log_size": 500,
            "ondisk_log_size": 500,
            "stats_invalid": "0",
            "stat_sum": {
                "num_bytes": 2097152000,
                "num_objects": 1,
                "num_object_clones": 0,
                "num_object_copies": 2,
                "num_objects_missing_on_primary": 0,
                "num_objects_degraded": 0,
                "num_objects_misplaced": 0,
                "num_objects_unfound": 0,
                "num_objects_dirty": 1,
                "num_whiteouts": 0,
                "num_read": 0,
                "num_read_kb": 0,
                "num_write": 500,
                "num_write_kb": 2048000,
                "num_scrub_errors": 0,
                "num_shallow_scrub_errors": 0,
                "num_deep_scrub_errors": 0,
                "num_objects_recovered": 0,
                "num_bytes_recovered": 0,
                "num_keys_recovered": 0,
                "num_objects_omap": 0,
                "num_objects_hit_set_archive": 0,
                "num_bytes_hit_set_archive": 0,
                "num_flush": 0,
                "num_flush_kb": 0,
                "num_evict": 0,
                "num_evict_kb": 0,
                "num_promote": 0,
                "num_flush_mode_high": 0,
                "num_flush_mode_low": 0,
                "num_evict_mode_some": 0,
                "num_evict_mode_full": 0
            },
            "up": [
                1,
                0
            ],
            "acting": [
                1,
                0
            ],
            "blocked_by": [],
            "up_primary": 1,
            "acting_primary": 1
        },
        "empty": 0,
        "dne": 0,
        "incomplete": 0,
        "last_epoch_started": 12,
        "hit_set_history": {
            "current_last_update": "0'0",
            "history": []
        }
    },
    "peer_info": [
        {
            "peer": "0",
            "pgid": "1.39",
            "last_update": "13'500",
            "last_complete": "13'500",
            "log_tail": "0'0",
            "last_user_version": 0,
            "last_backfill": "MAX",
            "last_backfill_bitwise": 0,
            "purged_snaps": "[]",
            "history": {
                "epoch_created": 11,
                "last_epoch_started": 12,
                "last_epoch_clean": 13,
                "last_epoch_split": 0,
                "last_epoch_marked_full": 0,
                "same_up_since": 0,
                "same_interval_since": 0,
                "same_primary_since": 0,
                "last_scrub": "0'0",
                "last_scrub_stamp": "2016-03-16 21:13:08.883121",
                "last_deep_scrub": "0'0",
                "last_deep_scrub_stamp": "2016-03-16 21:13:08.883121",
                "last_clean_scrub_stamp": "0.000000"
            },
            "stats": {
                "version": "0'0",
                "reported_seq": "0",
                "reported_epoch": "0",
                "state": "inactive",
                "last_fresh": "0.000000",
                "last_change": "0.000000",
                "last_active": "0.000000",
                "last_peered": "0.000000",
                "last_clean": "0.000000",
                "last_became_active": "0.000000",
                "last_became_peered": "0.000000",
                "last_unstale": "0.000000",
                "last_undegraded": "0.000000",
                "last_fullsized": "0.000000",
                "mapping_epoch": 0,
                "log_start": "0'0",
                "ondisk_log_start": "0'0",
                "created": 0,
                "last_epoch_clean": 0,
                "parent": "0.0",
                "parent_split_bits": 0,
                "last_scrub": "0'0",
                "last_scrub_stamp": "0.000000",
                "last_deep_scrub": "0'0",
                "last_deep_scrub_stamp": "0.000000",
                "last_clean_scrub_stamp": "0.000000",
                "log_size": 0,
                "ondisk_log_size": 0,
                "stats_invalid": "0",
                "stat_sum": {
                    "num_bytes": 0,
                    "num_objects": 0,
                    "num_object_clones": 0,
                    "num_object_copies": 0,
                    "num_objects_missing_on_primary": 0,
                    "num_objects_degraded": 0,
                    "num_objects_misplaced": 0,
                    "num_objects_unfound": 0,
                    "num_objects_dirty": 0,
                    "num_whiteouts": 0,
                    "num_read": 0,
                    "num_read_kb": 0,
                    "num_write": 0,
                    "num_write_kb": 0,
                    "num_scrub_errors": 0,
                    "num_shallow_scrub_errors": 0,
                    "num_deep_scrub_errors": 0,
                    "num_objects_recovered": 0,
                    "num_bytes_recovered": 0,
                    "num_keys_recovered": 0,
                    "num_objects_omap": 0,
                    "num_objects_hit_set_archive": 0,
                    "num_bytes_hit_set_archive": 0,
                    "num_flush": 0,
                    "num_flush_kb": 0,
                    "num_evict": 0,
                    "num_evict_kb": 0,
                    "num_promote": 0,
                    "num_flush_mode_high": 0,
                    "num_flush_mode_low": 0,
                    "num_evict_mode_some": 0,
                    "num_evict_mode_full": 0
                },
                "up": [],
                "acting": [],
                "blocked_by": [],
                "up_primary": -1,
                "acting_primary": -1
            },
            "empty": 0,
            "dne": 0,
            "incomplete": 0,
            "last_epoch_started": 12,
            "hit_set_history": {
                "current_last_update": "0'0",
                "history": []
            }
        }
    ],
    "recovery_state": [
        {
            "name": "Started\/Primary\/Active",
            "enter_time": "2016-03-16 21:13:36.769083",
            "might_have_unfound": [],
            "recovery_progress": {
                "backfill_targets": [],
                "waiting_on_backfill": [],
                "last_backfill_started": "MIN",
                "backfill_info": {
                    "begin": "MIN",
                    "end": "MIN",
                    "objects": []
                },
                "peer_backfill_info": [],
                "backfills_in_flight": [],
                "recovering": [],
                "pg_backend": {
                    "pull_from_peer": [],
                    "pushing": []
                }
            },
            "scrub": {
                "scrubber.epoch_start": "0",
                "scrubber.active": 0,
                "scrubber.waiting_on": 0,
                "scrubber.waiting_on_whom": []
            }
        },
        {
            "name": "Started",
            "enter_time": "2016-03-16 21:13:09.216260"
        }
    ],
    "agent_state": {}
}
```

That\\'s all for now.

&#x20;Regards, Eduardo Gonzalez
