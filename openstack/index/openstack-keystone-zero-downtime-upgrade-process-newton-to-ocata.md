# OpenStack keystone zero downtime upgrade process newton to ocata

This blog post will show Keystone upgrade procedure from OpenStack Newton to Ocata release with zero-downtime.

In the case of doing this in production, please read release notes, ensure a proper configuration, do database backups and test the upgrade a thousand times.

Keystone upgrade will need to stop one node in order to use it as upgrade server. | In the case of a PoC this is not an issue, but in a production environment, Keystone loads may be intensive and stopping a node for a while may decrease other nodes performance more than expected. | For this reason I prefer orchestrate the upgrade from an external Docker container. With this method all nodes will be fully running almost all the time.

*   New container won\\'t start any service, just will sync the database

    schema with new Keystone version avoiding stop a node to orchestrate

    the upgrade.
*   The Docker image is provided by OpenStack Kolla project, if already

    using Kolla this upgrade won\\'t be needed as kolla-ansible already

    provide an upgrade method.
*   At the moment of writing of this blog, Ocata packages were not

    released into stable repositories. For this reason I use DLRN

    repositories.
*   If Ocata is released please do not use DLRN, use stable packages

    instead.
*   Use stable Ocata Docker image if available with tag 4.0.x and will

    avoid repository configuration and package upgrades.
*   NOTE: Upgrade may need more steps depending of your own

    configuration, i.e, if using fernet token more steps are necessary

    during the upgrade.
* All Keystone nodes are behind HAproxy.

### Prepare the upgrade

Start Keystone Docker container with host networking (needed to communicate with database nodes directly) and root user (needed to install packages).

```
(host)# docker run -ti --net host -u 0 kolla/centos-binary-keystone:3.0.2 bash
```

Download Delorean CentOS trunk repositories

```
(keystone-upgrade)# curl -Lo /etc/yum.repos.d/delorean.repo http://buildlogs.centos.org/centos/7/cloud/x86_64/rdo-trunk-master-tested/delorean.repo
(keystone-upgrade)# curl -Lo /etc/yum.repos.d/delorean-deps.repo http://trunk.rdoproject.org/centos7/delorean-deps.repo
```

Disable Newton repository

```
(keystone-upgrade)# yum-config-manager --disable centos-openstack-newton
```

Ensure Newton repository is not longer used by the system

```
(keystone-upgrade)# yum repolist | grep -i openstack
delorean                        delorean-openstack-glance-0bf9d805886c2  565+255
```

Update all packages in the Docker container to bump keystone version to Ocata.

```
(keystone-upgrade)# yum clean all && yum update -y
```

Configure keystone.conf file, this are my settings. Review you configuration and ensure all is correctly, otherwise may cause issues in the database. | An important option is default\_domain\_id, this value is for backward compatible with users created under default domain.

```
(keystone-upgrade)# egrep ^[^#] /etc/keystone/keystone.conf 
[DEFAULT]
debug = False
log_file = /var/log/keystone/keystone.log
secure_proxy_ssl_header = HTTP_X_FORWARDED_PROTO
[database]
connection = mysql+pymysql://keystone:ickvaHC9opkwbz8z8sy28aLiFNezc7Z6Fm34frcB@192.168.100.10:3306/keystone
max_retries = -1
[cache]
backend = oslo_cache.memcache_pool
enabled = True
memcache_servers = 192.168.100.215:11211,192.168.100.170:11211
[identity]
default_domain_id = default
[token]
provider = uuid
```

Check migrate version in the database. | As you will notice, contract/data\_migrate/expand are in the same version

```
(mariadb)# mysql -ukeystone -pickvaHC9opkwbz8z8sy28aLiFNezc7Z6Fm34frcB -h192.168.100.10 keystone -e "select * from migrate_version;" 
Warning: Using a password on the command line interface can be insecure.
+-----------------------+--------------------------------------------------------------------------+---------+
| repository_id         | repository_path                                                          | version |
+-----------------------+--------------------------------------------------------------------------+---------+
| keystone              | /usr/lib/python2.7/site-packages/keystone/common/sql/migrate_repo        |     109 |
| keystone_contract     | /usr/lib/python2.7/site-packages/keystone/common/sql/contract_repo       |       4 |
| keystone_data_migrate | /usr/lib/python2.7/site-packages/keystone/common/sql/data_migration_repo |       4 |
| keystone_expand       | /usr/lib/python2.7/site-packages/keystone/common/sql/expand_repo         |       4 |
+-----------------------+--------------------------------------------------------------------------+---------+
```

Before start upgrading the database schema, you will need add SUPER privileges in the database to keystone user or set log\_bin\_trust\_function\_creators to True. | In my opinion is safer set the value to True, I don\\'t want keystone with SUPER privileges.

```
(mariadb)# mysql -uroot -pnkLMrBibfMTRqOGBAP3UAxdO4kOFfEaPptGM5UDL -h192.168.100.10 keystone -e "set global log_bin_trust_function_creators=1;"
```

Now use Rally, tempest or some tool to test/benchmarch keystone service during upgrade. | If don\\'t want to use one of those tools, just use this for command.

```
(host)# for i in {1000..6000} ; do openstack user create --password $i $i; done
```

### Start Upgrade

Check database status before upgrade using Doctor, this may raise issues in the configuration. Some of them may be ignored(Please, ensure is not an issue before ignoring). As example, I'm not using fernet tokens and errors appear about missing folder.

```
(keystone-upgrade)# keystone-manage doctor
```

Remove obsoleted tokens

```
(keystone-upgrade)# keystone-manage token_flush
```

Now, expand the database schema to latest version, in keystone.log can see the status.&#x20;

Check in the logs if some error is raised before jump to the next step.

```
(keystone-upgrade)# keystone-manage db_sync --expand


2017-01-31 13:42:02.772 306 INFO migrate.versioning.api [-] 4 -> 5... 
2017-01-31 13:42:03.004 306 INFO migrate.versioning.api [-] done
2017-01-31 13:42:03.005 306 INFO migrate.versioning.api [-] 5 -> 6... 
2017-01-31 13:42:03.310 306 INFO migrate.versioning.api [-] done
2017-01-31 13:42:03.310 306 INFO migrate.versioning.api [-] 6 -> 7... 
2017-01-31 13:42:03.670 306 INFO migrate.versioning.api [-] done
2017-01-31 13:42:03.671 306 INFO migrate.versioning.api [-] 7 -> 8... 
2017-01-31 13:42:03.984 306 INFO migrate.versioning.api [-] done
2017-01-31 13:42:03.985 306 INFO migrate.versioning.api [-] 8 -> 9... 
2017-01-31 13:42:04.185 306 INFO migrate.versioning.api [-] done
2017-01-31 13:42:04.185 306 INFO migrate.versioning.api [-] 9 -> 10... 
2017-01-31 13:42:07.202 306 INFO migrate.versioning.api [-] done
2017-01-31 13:42:07.202 306 INFO migrate.versioning.api [-] 10 -> 11... 
2017-01-31 13:42:07.481 306 INFO migrate.versioning.api [-] done
2017-01-31 13:42:07.481 306 INFO migrate.versioning.api [-] 11 -> 12... 
2017-01-31 13:42:11.334 306 INFO migrate.versioning.api [-] done
2017-01-31 13:42:11.334 306 INFO migrate.versioning.api [-] 12 -> 13... 
2017-01-31 13:42:11.560 306 INFO migrate.versioning.api [-] done
```

After expand the database, migrate it to latest version.&#x20;

Ensure there are not errors in Keystone logs.

```
(keystone-upgrade)# keystone-manage db_sync --migrate

#keystone.log
2017-01-31 13:42:58.771 314 INFO migrate.versioning.api [-] 4 -> 5... 
2017-01-31 13:42:58.943 314 INFO migrate.versioning.api [-] done
2017-01-31 13:42:58.943 314 INFO migrate.versioning.api [-] 5 -> 6... 
2017-01-31 13:42:59.143 314 INFO migrate.versioning.api [-] done
2017-01-31 13:42:59.143 314 INFO migrate.versioning.api [-] 6 -> 7... 
2017-01-31 13:42:59.340 314 INFO migrate.versioning.api [-] done
2017-01-31 13:42:59.341 314 INFO migrate.versioning.api [-] 7 -> 8... 
2017-01-31 13:42:59.698 314 INFO migrate.versioning.api [-] done
2017-01-31 13:42:59.699 314 INFO migrate.versioning.api [-] 8 -> 9... 
2017-01-31 13:42:59.852 314 INFO migrate.versioning.api [-] done
2017-01-31 13:42:59.852 314 INFO migrate.versioning.api [-] 9 -> 10... 
2017-01-31 13:43:00.135 314 INFO migrate.versioning.api [-] done
2017-01-31 13:43:00.135 314 INFO migrate.versioning.api [-] 10 -> 11... 
2017-01-31 13:43:00.545 314 INFO migrate.versioning.api [-] done
2017-01-31 13:43:00.545 314 INFO migrate.versioning.api [-] 11 -> 12... 
2017-01-31 13:43:00.703 314 INFO migrate.versioning.api [-] done
2017-01-31 13:43:00.703 314 INFO migrate.versioning.api [-] 12 -> 13... 
2017-01-31 13:43:00.854 314 INFO migrate.versioning.api [-] done
```

Now, see migrate\_version table, you will notice that expand and data\_migrate are in the latest version, but contract still in the previous version.

```
(mariadb)# mysql -ukeystone -pickvaHC9opkwbz8z8sy28aLiFNezc7Z6Fm34frcB -h192.168.100.10 keystone -e "select * from migrate_version;"
+-----------------------+--------------------------------------------------------------------------+---------+
| repository_id         | repository_path                                                          | version |
+-----------------------+--------------------------------------------------------------------------+---------+
| keystone              | /usr/lib/python2.7/site-packages/keystone/common/sql/migrate_repo        |     109 |
| keystone_contract     | /usr/lib/python2.7/site-packages/keystone/common/sql/contract_repo       |       4 |
| keystone_data_migrate | /usr/lib/python2.7/site-packages/keystone/common/sql/data_migration_repo |      13 |
| keystone_expand       | /usr/lib/python2.7/site-packages/keystone/common/sql/expand_repo         |      13 |
+-----------------------+--------------------------------------------------------------------------+---------+
```

### Every Keystone node, one by one

Go to keystone nodes.&#x20;

Stop Keystone services, in my case using wsgi inside Apache

```
(keystone_nodes)# systemctl stop httpd
```

Configure Ocata repositories as made in the Docker container.

Update packages, if you have Keystone sharing the node with other OpenStack service, do not update all packages as it will break other services.&#x20;

Update only required packages.

```
(keystone_nodes)# yum clean all && yum update -y
```

Configure Keystone configuration file to the desired state. Your configuration may change.

```
(keystone_nodes)# egrep ^[^#] /etc/keystone/keystone.conf 
[DEFAULT]
debug = False
log_file = /var/log/keystone/keystone.log
secure_proxy_ssl_header = HTTP_X_FORWARDED_PROTO
[database]
connection = mysql+pymysql://keystone:ickvaHC9opkwbz8z8sy28aLiFNezc7Z6Fm34frcB@192.168.100.10:3306/keystone
max_retries = -1
[cache]
backend = oslo_cache.memcache_pool
enabled = True
memcache_servers = 192.168.100.215:11211,192.168.100.170:11211
[identity]
default_domain_id = default
[token]
provider = uuid
```

Start Keystone service.

```
(keystone_nodes)# systemctl start httpd
```

### Finish Upgrade

After all the nodes are updated to latest version (please ensure all nodes are using latest packages, if not will fail).&#x20;

Contract Keystone database schema.

&#x20;Look at keystone.log for errors.

```
(keystone-upgrade)# keystone-manage db_sync --contract

keystone.log

2017-01-31 13:57:52.164 322 INFO migrate.versioning.api [-] 4 -> 5... 
2017-01-31 13:57:52.379 322 INFO migrate.versioning.api [-] done
2017-01-31 13:57:52.379 322 INFO migrate.versioning.api [-] 5 -> 6... 
2017-01-31 13:57:52.969 322 INFO migrate.versioning.api [-] done
2017-01-31 13:57:52.969 322 INFO migrate.versioning.api [-] 6 -> 7... 
2017-01-31 13:57:53.462 322 INFO migrate.versioning.api [-] done
2017-01-31 13:57:53.462 322 INFO migrate.versioning.api [-] 7 -> 8... 
2017-01-31 13:57:53.793 322 INFO migrate.versioning.api [-] done
2017-01-31 13:57:53.793 322 INFO migrate.versioning.api [-] 8 -> 9... 
2017-01-31 13:57:53.957 322 INFO migrate.versioning.api [-] done
2017-01-31 13:57:53.957 322 INFO migrate.versioning.api [-] 9 -> 10... 
2017-01-31 13:57:54.111 322 INFO migrate.versioning.api [-] done
2017-01-31 13:57:54.112 322 INFO migrate.versioning.api [-] 10 -> 11... 
2017-01-31 13:57:54.853 322 INFO migrate.versioning.api [-] done
2017-01-31 13:57:54.853 322 INFO migrate.versioning.api [-] 11 -> 12... 
2017-01-31 13:57:56.727 322 INFO migrate.versioning.api [-] done
2017-01-31 13:57:56.728 322 INFO migrate.versioning.api [-] 12 -> 13... 
2017-01-31 13:57:59.529 322 INFO migrate.versioning.api [-] done
```

Now if we look at migrate\_version table, will see that contract version is latest and match with the other version (Ensure all are in the same version).&#x20;

This means the database upgrade has been successfully implemented.

```
(mariadb)# mysql -ukeystone -pickvaHC9opkwbz8z8sy28aLiFNezc7Z6Fm34frcB -h192.168.100.10 keystone -e "select * from migrate_version;"
+-----------------------+--------------------------------------------------------------------------+---------+
| repository_id         | repository_path                                                          | version |
+-----------------------+--------------------------------------------------------------------------+---------+
| keystone              | /usr/lib/python2.7/site-packages/keystone/common/sql/migrate_repo        |     109 |
| keystone_contract     | /usr/lib/python2.7/site-packages/keystone/common/sql/contract_repo       |      13 |
| keystone_data_migrate | /usr/lib/python2.7/site-packages/keystone/common/sql/data_migration_repo |      13 |
| keystone_expand       | /usr/lib/python2.7/site-packages/keystone/common/sql/expand_repo         |      13 |
+-----------------------+--------------------------------------------------------------------------+---------+
```

Remove log\_bin\_trust\_function\_creators value.

```
(mariadb)# mysql -uroot -pnkLMrBibfMTRqOGBAP3UAxdO4kOFfEaPptGM5UDL -h192.168.100.10 keystone -e "set global log_bin_trust_function_creators=0;"
```

After finish the upgrade, Rally tests should not have any error.

\*\*If using HAproxy for load balance Keystone service, some errors may happen due a connection drop while stopping Keystone service and re-balance to other Keystone node. This can be avoided putting the node to update in Maintenance Mode in HAproxy backend.

Have to thank Keystone team in #openstack-keystone IRC channel for the help provided with a couple of issues.

Regards, Eduardo Gonzalez
