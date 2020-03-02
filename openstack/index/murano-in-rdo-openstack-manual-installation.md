# Murano in RDO OpenStack manual installation

Want to install and use Murano in a RDO OpenStack environment? Here are the steps to do it.

The first thing we need to do, is to know what is Murano: 

Murano is an application catalog who gives the users the capacity to launch pre-configured s/instances/jobs/g with apps in an OpenStack infrastructure.

As the final user just select an application from a catalog with a minimal configuration, and Murano will take the role to orchestrate the background jobs\(create instances, configure apps, connect networks, etc\) For more information about application catalog project refer to this site: [https://wiki.openstack.org/wiki/Murano/ApplicationCatalog](https://wiki.openstack.org/wiki/Murano/ApplicationCatalog)

At this tutorial, i will use the following s/configurations/versions/g:

* Centos 7.1
* RDO Liberty release
* Hosts installed with packstack/ML2 network

Let's start installing some pre requisites

```text
sudo yum install -y gcc python-setuptools python-devel git postgresql-devel libffi-devel openssl-devel
```

Install pip

```text
sudo easy_install pip
```

Install tox and upgrade six

```text
sudo pip install tox
sudo pip install --upgrade six
```

Create a database for murano

```text
mysql -u root -p
CREATE DATABASE murano;
```

Create murano user at MySQL

```text
GRANT ALL PRIVILEGES ON murano.* TO 'murano'@'localhost' IDENTIFIED BY 'MURANODB_PASS';
GRANT ALL PRIVILEGES ON murano.* TO 'murano'@'%' IDENTIFIED BY 'MURANODB_PASS';
```

Clone murano from liberty/stable branch

```text
git clone -b stable/liberty git://git.openstack.org/openstack/murano
```

Install all requirements

```text
cd ~/murano/
sudo  pip install -r requirements.txt
```

Install murano

```text
sudo python setup.py install
```

Create sample configuration file

```text
oslo-config-generator --config-file etc/oslo-config-generator/murano.conf
```

Create murano directory and copy the sample content on it

```text
mkdir /etc/murano
cp ~/murano/etc/murano/* /etc/murano/
```

Rename sample configuration to murano.conf

```text
mv /etc/murano/murano.conf.sample /etc/murano/murano.conf
```

\| Edit the configuration file like this, adjust the configuration as your environment needs. `vi /etc/murano/murano.conf`

```text
[oslo_messaging_rabbit]

rabbit_host=RABBITMQ_IP
rabbit_port=5672
rabbit_hosts=RABBITMQ_IP:5672
rabbit_use_ssl=False
rabbit_userid=guest
rabbit_password=guest
rabbit_virtual_host=/
rabbit_ha_queues=False
rabbit_notification_exchange=openstack
rabbit_notification_topic=notifications

[database]
connection = mysql://murano:MURANODB_PASS@MYSQL_IP/murano

[keystone_authtoken]
auth_uri=http://KEYSTONE_IP:5000/v2.0
identity_uri=http://KEYSTONE_IP:35357
admin_user=murano
admin_password=MURANO_PASS
admin_tenant_name=services

[murano]
url = http://MURANO_IP:8082

[rabbitmq]

host=RABBITMQ_IP
login=guest
password=guest
virtual_host=/
```

Create murano user

```text
openstack user create --password MURANO_PASS murano
```

Add murano user to services tenant with admin privileges

```text
openstack role add --project services --user murano admin
```

Create a service for application-catalog

```text
openstack service create --name muranoapi --description "Murano Project" application-catalog
```

Associate an endpoint to application-catalog service

```text
openstack endpoint create --region RegionOne --publicurl 'http://MURANO_IP:8082/' --adminurl 'http://MURANO_IP:8082/' --internalurl 'http://http://MURANO_IP:8082/' MURANO_SERVICE_ID
```

Sync the database

```text
murano-db-manage --config-file /etc/murano/murano.conf upgrade
```

Open a new terminal and start murano-api service

```text
murano-api --config-file /etc/murano/murano.conf
```

Import base murano package

```text
murano-manage --config-file /etc/murano/murano.conf import-package murano/meta/io.murano
```

In a new terminal, start murano-engine service

```text
murano-engine --config-file /etc/murano/murano.conf
```

Clone stable liberty module for horizon

```text
git clone -b stable/liberty git://git.openstack.org/openstack/murano-dashboard
```

Install base requirements

```text
cd ~/murano-dashboard
pip install -r requirements.txt
```

Install murano-dashboard module

```text
sudo python setup.py install
```

Enable murano-dashboard at horizon

```text
cp muranodashboard/local/_50_murano.py /usr/share/openstack-dashboard/openstack_dashboard/enabled/
```

Restart apache to apply changes

```text
systemctl restart httpd
```

Import ApacheHttpServer package

```text
murano --murano-repo-url="http://storage.apps.openstack.org/" package-import io.murano.apps.apache.ApacheHttpServer
```

You can find more packages at: [http://apps.openstack.org/\#tab=murano-apps](http://apps.openstack.org/#tab=murano-apps)

This will add a Debian image to glance image service, wait until the image is in active status

Create a file with the following content, modify the variables with your own needs`vi object_model_patch.json`

```text
[
    { "op": "add", "path": "/-", "value":
        {
            "instance": {
                "availabilityZone": "nova",
                "name": "APP_NAME",
                "image": "GLANCE_IMAGE_ID",
                "keyname": "KEY_PAIR",
                "flavor": "FLAVOR",
                "assignFloatingIp": false,
                "?": {
                    "type": "io.murano.resources.LinuxMuranoInstance",
                    "id": "===id1==="
                }
            },
            "name": "ApacheHttpServer",
            "enablePHP": true,
            "?": {
                "type": "io.murano.apps.apache.ApacheHttpServer",
                "id": "===id2==="
            }
        }
    }
]
```

Create an environment

```text
murano environment-create --join-subnet-id SUBNET_ID ENVIRONMENT_NAME

murano environment-create --join-subnet-id e2c5175a-d5bc-4eb7-91ba-67ac9120c64a test
+----------------------------------+------+---------------------+---------------------+
| ID                               | Name | Created             | Updated             |
+----------------------------------+------+---------------------+---------------------+
| 68a19d233d2d42459faf64d375d995e5 | test | 2015-12-11T13:09:57 | 2015-12-11T13:09:57 |
+----------------------------------+------+---------------------+---------------------+
```

Create a session for temporal working on the environment

```text
murano environment-session-create ENVIRONMENT_ID

murano environment-session-create 68a19d233d2d42459faf64d375d995e5
Created new session:
+----------+----------------------------------+
| Property | Value                            |
+----------+----------------------------------+
| id       | b0f5e39a9c4c419c9ee7fdb6c92c37a6 |
+----------+----------------------------------+
```

Add the file with the apps configuration

```text
murano environment-apps-edit --session-id SESSION_ID ENVIRONMENT_ID FILE_NAME

murano environment-apps-edit --session-id b0f5e39a9c4c419c9ee7fdb6c92c37a6 68a19d233d2d42459faf64d375d995e5 object_model_patch.json 
```

Deploy the environment

```text
murano environment-deploy ENVIRONMENT_ID --session-id SESSION_ID

murano environment-deploy 68a19d233d2d42459faf64d375d995e5 --session-id b0f5e39a9c4c419c9ee7fdb6c92c37a6
+-----------+-------------------------------------------------------------+
| Property  | Value                                                       |
+-----------+-------------------------------------------------------------+
| created   | 2015-12-11T13:09:57                                         |
| id        | 68a19d233d2d42459faf64d375d995e5                            |
| name      | test                                                        |
| services  | [                                                           |
|           |   {                                                         |
|           |     "instance": {                                           |
|           |       "availabilityZone": "nova",                           |
|           |       "name": "test",                                       |
|           |       "assignFloatingIp": false,                            |
|           |       "keyname": "",                                        |
|           |       "flavor": "twogb",                                    |
|           |       "image": "9049eb0c-081e-4d56-9413-72fdc6f8d8bf",      |
|           |       "?": {                                                |
|           |         "type": "io.murano.resources.LinuxMuranoInstance",  |
|           |         "id": "30f5a591a58a468fbf4d7ef4755e0512"            |
|           |       }                                                     |
|           |     },                                                      |
|           |     "name": "ApacheHttpServer",                             |
|           |     "enablePHP": true,                                      |
|           |     "?": {                                                  |
|           |       "status": "deploying",                                |
|           |       "type": "io.murano.apps.apache.ApacheHttpServer",     |
|           |       "id": "98b994565c634f7e97d5f365203ce222"              |
|           |     }                                                       |
|           |   }                                                         |
|           | ]                                                           |
| status    | deploying                                                   |
| tenant_id | 3a5d50fac9a3462fa4d76b8b84677c3f                            |
| updated   | 2015-12-11T13:09:57                                         |
| version   | 0                                                           |
+-----------+-------------------------------------------------------------+
```

Now, you can check at nova the building status of the instances

```text
nova list
+--------------------------------------+-----------------------------------------+--------+------------+-------------+----------+
| ID                                   | Name                                    | Status | Task State | Power State | Networks |
+--------------------------------------+-----------------------------------------+--------+------------+-------------+----------+
| a68cedfb-7b4c-47a6-96fb-6b64a85a8ca6 | murano-mmnpdii1ozz7r2-test-5np5cvfeoiyh | BUILD  | scheduling | NOSTATE     |          |
+--------------------------------------+-----------------------------------------+--------+------------+-------------+----------+
```

After a while, the instance is up and running

```text
nova list
+--------------------------------------+-----------------------------------------+--------+------------+-------------+------------------+
| ID                                   | Name                                    | Status | Task State | Power State | Networks         |
+--------------------------------------+-----------------------------------------+--------+------------+-------------+------------------+
| a68cedfb-7b4c-47a6-96fb-6b64a85a8ca6 | murano-mmnpdii1ozz7r2-test-5np5cvfeoiyh | ACTIVE | -          | Running     | private=10.0.0.8 |
+--------------------------------------+-----------------------------------------+--------+------------+-------------+------------------+
```

Once the instance is active, murano will configure the application inside, wait until the status is ready.

```text
murano environment-show f392de2004e24ff7b2a08f05df0599b8
+-----------+---------------------------------------------------------------+
| Property  | Value                                                         |
+-----------+---------------------------------------------------------------+
| created   | 2015-12-11T13:43:23                                           |
| id        | 68a19d233d2d42459faf64d375d995e5                              |
| name      | test                                                          |
| services  | [                                                             |
|           |   {                                                           |
|           |     "instance": {                                             |
|           |       "availabilityZone": "nova",                             |
|           |       "openstackId": "91615340-e1d3-428e-848f-38a762004d33",  |
|           |       "name": "test",                                         |
|           |       "securityGroupName": null,                              |
|           |       "image": "9049eb0c-081e-4d56-9413-72fdc6f8d8bf",        |
|           |       "assignFloatingIp": false,                              |
|           |       "floatingIpAddress": null,                              |
|           |       "keyname": "",                                          |
|           |       "?": {                                                  |
|           |         "classVersion": "0.0.0",                              |
|           |         "name": null,                                         |
|           |         "package": "io.murano",                               |
|           |         "type": "io.murano.resources.LinuxMuranoInstance",    |
|           |         "_actions": {},                                       |
|           |         "id": "30f5a591a58a468fbf4d7ef4755e0512"              |
|           |       },                                                      |
|           |       "ipAddresses": [                                        |
|           |         "10.0.0.8"                                            |
|           |       ],                                                      |
|           |       "flavor": "twogb",                                      |
|           |       "networks": {                                           |
|           |         "useFlatNetwork": false,                              |
|           |         "primaryNetwork": null,                               |
|           |         "useEnvironmentNetwork": true,                        |
|           |         "customNetworks": []                                  |
|           |       },                                                      |
|           |       "sharedIps": []                                         |
|           |     },                                                        |
|           |     "name": "ApacheHttpServer",                               |
|           |     "?": {                                                    |
|           |       "classVersion": "0.0.0",                                |
|           |       "status": "ready",                                      |
|           |       "name": null,                                           |
|           |       "package": "io.murano.apps.apache.ApacheHttpServer",    |
|           |       "type": "io.murano.apps.apache.ApacheHttpServer",       |
|           |       "_actions": {},                                         |
|           |       "id": "98b994565c634f7e97d5f365203ce222"                |
|           |     },                                                        |
|           |     "enablePHP": true                                         |
|           |   }                                                           |
|           | ]                                                             |
| status    | ready                                                         |
| tenant_id | 3a5d50fac9a3462fa4d76b8b84677c3f                              |
| updated   | 2015-12-11T13:47:35                                           |
| version   | 1                                                             |
+-----------+---------------------------------------------------------------+
```

That's all you need to have up and running a Murano application catalog, for now there is no rpm package to ease the installation, so you need to install from source like we done. 

A thing you can do, is create systemd files to manage murano services in a easier way.

Regards, Eduardo Gonzalez

