# Migrate keystone v2.0 to keystone v3 OpenStack

Migrate from keystone v2.0 to v3 isn't as easy like just changing the endpoints at the database, every service must be configured to authenticate against keystone v3.

I've been working on that the past few days looking for a method, with the purpose of facilitate operators life's who need this kind of migration.

I have to thank Adam Young work, i followed his blog to make a first configuration idea, after that, i configured all core services to make use of keystone v3. If you want to check Adam's blog, follow this link: [http://adam.younglogic.com/2015/05/rdo-v3-only/](http://adam.younglogic.com/2015/05/rdo-v3-only/)

I used OpenStack Liberty installed with RDO packstack over CentOS 7 servers. The example IP used is `192.168.200.168`, use your own according your needs. Password used for all services is `PASSWD1234`, use your own password, you can locate your passwords at the packstack answer file.

Horizon

First we configure Horizon with keystone v3 as below:

```text
vi /etc/openstack-dashboard/local_settings

OPENSTACK_API_VERSIONS = {
    "identity": 3
}

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
```

keystone

Check your current identity endpoints

```text
mysql  --user keystone_admin --password=PASSWD1234  keystone -e "select interface, url from endpoint where service_id =  (select id from service where service.type = 'identity');"
```

Change your public, admin and internal endpoints with v3 at the end, instead of v2.0

```text
mysql  --user keystone_admin --password=PASSWD1234   keystone -e "update endpoint set   url  = 'http://192.168.200.178:5000/v3' where  interface ='internal' and  service_id =  (select id from service where service.type = 'identity');"

mysql  --user keystone_admin --password=PASSWD1234   keystone -e "update endpoint set   url  = 'http://192.168.200.178:5000/v3' where  interface ='public' and  service_id =  (select id from service where service.type = 'identity');"

mysql  --user keystone_admin --password=PASSWD1234   keystone -e "update endpoint set   url  = 'http://192.168.200.178:35357/v3' where  interface ='admin' and  service_id =  (select id from service where service.type = 'identity');"
```

Ensure the endpoints are properly created

```text
mysql  --user keystone_admin --password=KEYSTONE_DB_PW   keystone -e "select interface, url from endpoint where service_id =  (select id from service where service.type = 'identity');"
```

Create a source file or edit keystonerc\_admin with the following data

```text
vi v3_keystone

unset OS_SERVICE_TOKEN
export OS_USERNAME=admin
export OS_PASSWORD=PASSWD1234
export OS_AUTH_URL=http://192.168.200.178:5000/v3
export OS_PROJECT_NAME=admin
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_REGION_NAME=RegionOne
export PS1='[\u@\h \W(keystone_admin)]\$ '
export OS_IDENTITY_API_VERSION=3
```

Comment both pipelines, in public\_api and admin\_api

```text
vi /usr/share/keystone/keystone-dist-paste.ini

[pipeline:public_api]
# The last item in this pipeline must be public_service or an equivalent
# application. It cannot be a filter.
#pipeline = sizelimit url_normalize request_id build_auth_context token_auth admin_token_auth json_body ec2_extension user_crud_extension public_service

[pipeline:admin_api]
# The last item in this pipeline must be admin_service or an equivalent
# application. It cannot be a filter.
#pipeline = sizelimit url_normalize request_id build_auth_context token_auth admin_token_auth json_body ec2_extension s3_extension crud_extension admin_service
```

Comment v2.0 entries in composite:main and admin sections.

```text
[composite:main]
use = egg:Paste#urlmap
#/v2.0 = public_api
/v3 = api_v3
/ = public_version_api

[composite:admin]
use = egg:Paste#urlmap
#/v2.0 = admin_api
/v3 = api_v3
/ = admin_version_api
```

Restart httpd to apply changes

```text
systemctl restart httpd
```

Check whether keystone and horizon are properly working

The command below should prompt an user list, if not, check configuration in previous steps

```text
openstack user list
```

Glance

Edit the following files, with the content below:

```text
vi /etc/glance/glance-api.conf 
vi /etc/glance/glance-registry.conf 
vi /etc/glance/glance-cache.conf 

[keystone_authtoken]

auth_plugin = password
auth_url = http://192.168.200.178:35357
username = glance
password = PASSWD1234
project_name = services
user_domain_name = Default
project_domain_name = Default
auth_uri=http://192.168.200.178:5000
```

Comment the following lines:

```text
#auth_host=127.0.0.1
#auth_port=35357
#auth_protocol=http
#identity_uri=http://192.168.200.178:35357
#admin_user=glance
#admin_password=PASSWD1234
#admin_tenant_name=services
```

Those lines, should be commented in all the other OpenStack core services at keystone\_authtoken section

Edit the files below and comment the lines inside keystone\_authtoken section.

```text
vi /usr/share/glance/glance-api-dist.conf 
vi /usr/share/glance/glance-registry-dist.conf 

[keystone_authtoken]
#admin_tenant_name = %SERVICE_TENANT_NAME%
#admin_user = %SERVICE_USER%
#admin_password = %SERVICE_PASSWORD%
#auth_host = 127.0.0.1
#auth_port = 35357
#auth_protocol = http
```

Restart glance services

```text
openstack-service restart glance
```

Ensure glance service is working

```text
openstack image list
```

Nova

Edit the file below and comment the lines inside keystone\_authtoken

```text
vi /usr/share/nova/nova-dist.conf

[keystone_authtoken]
#auth_host = 127.0.0.1
#auth_port = 35357
#auth_protocol = http
```

Edit nova.conf and add the auth content inside keystone\_authtoken, don't forget to comment the lines related to the last auth method, which were commented in glance section.

```text
vi /etc/nova/nova.conf

[keystone_authtoken]

auth_plugin = password
auth_url = http://192.168.200.178:35357
username = nova
password = PASSWD1234
project_name = services
user_domain_name = Default
project_domain_name = Default
auth_uri=http://192.168.200.178:5000
```

Configure nova authentication against neutron

```text
[neutron]

auth_plugin = password
auth_url = http://192.168.200.178:35357
username = neutron
password = PASSWD1234
project_name = services
user_domain_name = Default
project_domain_name = Default
auth_uri=http://192.168.200.178:5000
```

Restart nova services to apply changes

```text
openstack-service restart nova
```

Check if nova works

```text
openstack hypervisor list
```

Neutron

Comment or remove the following entries at api-paste.ini and add the new version auth lines

```text
vi /etc/neutron/api-paste.ini 

[filter:authtoken]
#identity_uri=http://192.168.200.178:35357
#admin_user=neutron
#admin_password=PASSWD1234
#auth_uri=http://192.168.200.178:5000/v2.0
#admin_tenant_name=services

auth_plugin = password
auth_url = http://192.168.200.178:35357
username = neutron
password = PASSWD1234
project_name = services
user_domain_name = Default
project_domain_name = Default
auth_uri=http://192.168.200.178:5000
```

Configure v3 authentication for metadata service, remember comment the old auth lines

```text
vi /etc/neutron/metadata_agent.ini

[DEFAULT]

auth_plugin = password
auth_url = http://192.168.200.178:35357
username = neutron
password = PASSWD1234
project_name = services
user_domain_name = Default
project_domain_name = Default
auth_uri=http://192.168.200.178:5000
```

Configure neutron server with v3 auth

```text
vi /etc/neutron/neutron.conf

nova_admin_auth_url = http://192.168.200.178:5000
# nova_admin_tenant_id =1fb93c84c6474c5ea92c0ed5f7d4a6a7
nova_admin_tenant_name = services


[keystone_authtoken]

auth_plugin = password
auth_url = http://192.168.200.178:35357
username = neutron
password = PASSWD1234
project_name = services
user_domain_name = Default
project_domain_name = Default
auth_uri=http://192.168.200.178:5000

#auth_uri = http://192.168.200.178:5000/v2.0
#identity_uri = http://192.168.200.178:35357
#admin_tenant_name = services
#admin_user = neutron
#admin_password = PASSWD1234
```

Configure neutron auth against nova services

```text
[nova]

auth_plugin = password
auth_url = http://192.168.200.178:35357
username = nova
password = PASSWD1234
project_name = services
user_domain_name = Default
project_domain_name = Default
auth_uri=http://192.168.200.178:5000
```

Restart neutron services to apply changes

```text
openstack-service restart neutron
```

Test correct neutron funtionality

```text
openstack network list
```

Cinder

Edit api-paste.ini with the following content

```text
vi /etc/cinder/api-paste.ini 

[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
auth_plugin = password
auth_url = http://192.168.200.178:35357
username = cinder
password = PASSWD1234
project_name = services
user_domain_name = Default
project_domain_name = Default
auth_uri=http://192.168.200.178:5000
#admin_tenant_name=services
#auth_uri=http://192.168.200.178:5000/v2.0
#admin_user=cinder
#identity_uri=http://192.168.200.178:35357
#admin_password=PASSWD1234
```

Restart cinder services to apply changes

```text
openstack-service restart cinder
```

Ensure cinder is properly running

```text
openstack volume create --size 1 testvolume
openstack volume list
```

Now, you can check if nova is working fine, create an instance and ensure it is in ACTIVE state.

```text
openstack server create --flavor m1.tiny --image cirros --nic net-id=a1aa6336-9ae2-4ffb-99f5-1b6d1130989c testinstance
openstack server list
```

If any error occurs, review configuration files

Swift

Configure proxy server auth agains keystone v3

```text
vi /etc/swift/proxy-server.conf

[filter:authtoken]
log_name = swift
signing_dir = /var/cache/swift
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
auth_plugin = password
auth_url = http://192.168.200.178:35357
username = swift
password = PASSWD1234
project_name = services
user_domain_name = Default
project_domain_name = Default
auth_uri=http://192.168.200.178:5000

#auth_uri = http://192.168.200.178:5000/v2.0
#identity_uri = http://192.168.200.178:35357
#admin_tenant_name = services
#admin_user = swift
#admin_password = PASSWD1234
delay_auth_decision = 1
cache = swift.cache
include_service_catalog = False
```

Restart swift services to apply changes

```text
openstack-service restart swift
```

\| Swift commands must be issued with python-openstackclient instead of swiftclient

If done with swiftclient a -V 3 option must be used in order to avoid issues

Check if swift works fine

```text
openstack container create testcontainer
```

Ceilometer

Configure ceilometer service in order to authenticate agains keystone v3

```text
[keystone_authtoken]

auth_plugin = password
auth_url = http://192.168.200.178:35357
username = ceilometer
password = PASSWD1234
project_name = services
user_domain_name = Default
project_domain_name = Default
auth_uri=http://192.168.200.178:5000

[service_credentials]

os_auth_url = http://controller:5000/v3
os_username = ceilometer
os_tenant_name = services
os_password = PASSWD1234
os_endpoint_type = internalURL
os_region_name = RegionOne
```

Restart ceilometer services

```text
openstack-service restart ceilometer
```

Check ceilometer funtionality

```text
ceilometer statistics -m memory
```

Heat

Configure Heat authentication, since trusts are not stable use password auth method

```text
vi /etc/heat/heat.conf

# Allowed values: password, trusts
#deferred_auth_method = trusts
deferred_auth_method = password
```

Configure auth\_uri and keystone\_authtoken section

```text
# From heat.common.config
#
# Unversioned keystone url in format like http://0.0.0.0:5000. (string value)
#auth_uri =
auth_uri = http://192.168.200.178:5000

[keystone_authtoken]

auth_plugin = password
auth_url = http://192.168.200.178:35357
username = heat
password = PASSWD1234
project_name = services
user_domain_name = Default
project_domain_name = Default
auth_uri=http://192.168.200.178:5000

#admin_user=heat
#admin_password=PASSWD1234
#admin_tenant_name=services
#identity_uri=http://192.168.200.178:35357
#auth_uri=http://192.168.200.178:5000/v2.0
```

Comment or remove heat-dist auth entries in order to avoid conflicts with your config files

```text
vi /usr/share/heat/heat-dist.conf 

[keystone_authtoken]
#auth_host = 127.0.0.1
#auth_port = 35357
#auth_protocol = http
#auth_uri = http://127.0.0.1:5000/v2.0
#signing_dir = /tmp/keystone-signing-heat
```

Restart heat services to apply changes

```text
openstack-service restart heat
```

Ensure heat authentication is properly configured with a simple heat template

```text
heat stack-create --template-file sample.yaml teststack
```

Most issues occurs in the authentication between nova and neutron services, if instances does not launch as expected, review \[nova\] and \[neutron\] sections.

Best regards, Eduardo Gonzalez

