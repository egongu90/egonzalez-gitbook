# Nova VNC flows under the hood

Most OpenStack deployments has a VNC console implemented with nova-novncproxy. This service gives the final user the ability to log into their instances in a web based method through a browser.

At this post i'm going to show how a vnc console request works under the hood while using the following command or lauching a vnc session through Horizon.

```text
# nova get-vnc-console INSTANCE novnc
```

First of all, a user connects to NOVA and issues a VNC console request for an instance. Nova API needs to validate the user issuing an authentication request to keystone.

The user receives a token with nova's endpoint URL in the catalog, with that endpoint and the token, the user makes a request against nova calling for a VNC session.

```text
GET http://192.168.200.208:5000/v2.0 -H "Accept: application/json" -H \
"User-Agent: python-keystoneclient"

GET http://192.168.200.208:8774/v2/ -H "User-Agent: python-novaclient" -H \
"Accept: application/json" -H "X-Auth-Token: {SHA1}3b6262df9eaba5da33c1004805187806322201f1"
```

If a name instead of an instance ID is used in the request, Nova need to check his database to match that name with his corresponding ID, as we can see in the following request.

```text
GET http://192.168.200.208:8774/v2/ee84411cdb8148d28674b129ef482f31/servers?name=test1 \
-H "User-Agent: python-novaclient" -H "Accept: application/json" \
-H "X-Auth-Token: {SHA1}3b6262df9eaba5da33c1004805187806322201f1"

RESP BODY: {"servers": [{"id": "9165dbda-f54e-4186-b2cb-e6ca05ac53ee", \
"links": [{"href": "http://192.168.200.208:8774/v2/ee84411cdb8148d28674b129ef482f31/servers/9165dbda-f54e-4186-b2cb-e6ca05ac53ee", "rel": "self"},\
 {"href": "http://192.168.200.208:8774/ee84411cdb8148d28674b129ef482f31/servers/9165dbda-f54e-4186-b2cb-e6ca05ac53ee", \
"rel": "bookmark"}], "name": "test1"}]}
```

Once the ID is matched with the name, Nova check information about the instance \(I thought it was to validate if is in ACTIVE status, but i realized that even when is in STOPPED status the request is made it anyway\).

```text
GET http://192.168.200.208:8774/v2/ee84411cdb8148d28674b129ef482f31/servers/9165dbda-f54e-4186-b2cb-e6ca05ac53ee\
 -H "User-Agent: python-novaclient" -H "Accept: application/json" \
 -H "X-Auth-Token: {SHA1}3b6262df9eaba5da33c1004805187806322201f1"

RESP BODY: {"server": {"status": "ACTIVE", "updated": "2016-03-02T17:28:45Z", "hostId": "ca3a874dcad9079fcc6a0b10b0e2efaa394bc66b5335197fdd9c2498", "OS-EXT-SRV-ATTR:host": "liberty", "addresses": {"private": [{"OS-EXT-IPS-MAC:mac_addr": "fa:16:3e:aa:1c:32", "version": 4, "addr": "10.0.0.6", "OS-EXT-IPS:type": "fixed"}]}, "links": [{"href": "http://192.168.200.208:8774/v2/ee84411cdb8148d28674b129ef482f31/servers/9165dbda-f54e-4186-b2cb-e6ca05ac53ee", "rel": "self"}, {"href": "http://192.168.200.208:8774/ee84411cdb8148d28674b129ef482f31/servers/9165dbda-f54e-4186-b2cb-e6ca05ac53ee", "rel": "bookmark"}], "key_name": null, "image": {"id": "bf31eadd-c5f4-40f8-9ddb-30f688ca5e5f", "links": [{"href": "http://192.168.200.208:8774/ee84411cdb8148d28674b129ef482f31/images/bf31eadd-c5f4-40f8-9ddb-30f688ca5e5f", "rel": "bookmark"}]}, "OS-EXT-STS:task_state": null, "OS-EXT-STS:vm_state": "active", "OS-EXT-SRV-ATTR:instance_name": "instance-0000000a", "OS-SRV-USG:launched_at": "2016-03-02T17:28:45.000000", "OS-EXT-SRV-ATTR:hypervisor_hostname": "liberty", "flavor": {"id": "1", "links": [{"href": "http://192.168.200.208:8774/ee84411cdb8148d28674b129ef482f31/flavors/1", "rel": "bookmark"}]}, "id": "9165dbda-f54e-4186-b2cb-e6ca05ac53ee", "security_groups": [{"name": "default"}], "OS-SRV-USG:terminated_at": null, "OS-EXT-AZ:availability_zone": "nova", "user_id": "d9164a323be649c0a8c5c80fdd5bd585", "name": "test1", "created": "2016-03-02T17:28:34Z", "tenant_id": "ee84411cdb8148d28674b129ef482f31", "OS-DCF:diskConfig": "MANUAL", "os-extended-volumes:volumes_attached": [], "accessIPv4": "", "accessIPv6": "", "progress": 0, "OS-EXT-STS:power_state": 1, "config_drive": "", "metadata": {}}}
```

When we get the information, nova-api POST a request to nova-consoleauth for a VNC console.

```text
POST http://192.168.200.208:8774/v2/ee84411cdb8148d28674b129ef482f31/servers/9165dbda-f54e-4186-b2cb-e6ca05ac53ee/action \
-H "User-Agent: python-novaclient" -H "Content-Type: application/json" \
-H "Accept: application/json" -H "X-Auth-Token: {SHA1}3b6262df9eaba5da33c1004805187806322201f1"\
-d '{"os-getVNCConsole": {"type": "novnc"}}'


DEBUG nova.api.openstack.wsgi [req-2201b9d6-5711-46d3-ac4d-669094f07527 \
d9164a323be649c0a8c5c80fdd5bd585 ee84411cdb8148d28674b129ef482f31 - - -] \
Action: 'action', calling method: , body: {"os-getVNCConsole": {"type": "novnc"}} \
_process_stack /usr/lib/python2.7/site-packages/nova/api/openstack/wsgi.py:789
```

Nova-consoleauth receives the console request and create an access URL while generates a temporary token for the vnc console.

```text
INFO nova.consoleauth.manager [req-d4def6f9-1ab9-4626-b6a8-d81643ea5eb4 d9164a323be649c0a8c5c80fdd5bd585 ee84411cdb8148d28674b129ef482f31 - - -] \
Received Token: 3dfcd011-28f1-4cf3-8f5c-8cd18de4560e, \
{'instance_uuid': u'9165dbda-f54e-4186-b2cb-e6ca05ac53ee', \
'access_url': u'http://192.168.200.208:6080/vnc_auto.html?token=3dfcd011-28f1-4cf3-8f5c-8cd18de4560e',\
 'token': u'3dfcd011-28f1-4cf3-8f5c-8cd18de4560e', 'last_activity_at': 1456940028.356214, \
'internal_access_path': None, 'console_type': u'novnc', 'host': u'liberty', 'port': u'5900'}
```

Nova-consoleauth answer to nova-api who also answers to the user with an access URL.

This URL got the following content on it:

* HTTP or HTTPS connection to nova-novncproxy IP
* Nova-novncproxy port
* A token to validate the VNC connection

```text
RESP BODY: {"console": {"url": "http://192.168.200.208:6080/vnc_auto.html?token=3dfcd011-28f1-4cf3-8f5c-8cd18de4560e", "type": "novnc"}}

+-------+--------------------------------------------------------------------------------------+
| Type  | Url                                                                                  |
+-------+--------------------------------------------------------------------------------------+
| novnc | http://192.168.200.208:6080/vnc_auto.html?token=3dfcd011-28f1-4cf3-8f5c-8cd18de4560e |
+-------+--------------------------------------------------------------------------------------+
```

Until now, nova-novncproxy service can be stopped or isn't used at all, is at this point the when proxy server enter into the game. The user connects through a web browser to the nova-novncproxy's URL provided by nova before.

```text
DEBUG nova.console.websocketproxy [-] 192.168.200.1: \
new handler Process vmsg /usr/lib/python2.7/site-packages/websockify/websocket.py:828
```

Nova-vncproxy validate the issued token with the URL against nova-consoleauth.

```text
nova.consoleauth.manager [req-399c7b58-700a-4779-b215-b12d10056813 - - - - -] \
Checking Token: 3dfcd011-28f1-4cf3-8f5c-8cd18de4560e, True
```

When the token is validated, nova-novncproxy maps compute's node private IP \(at this case port 5900\) with the nova-novncproxy public IP\(6080 port\).

```text
INFO nova.console.websocketproxy [req-399c7b58-700a-4779-b215-b12d10056813 - - - - -]\
   7: connect info: {u'instance_uuid': u'9165dbda-f54e-4186-b2cb-e6ca05ac53ee', u'\
internal_access_path': None, u'last_activity_at': 1456940028.356214, \
u'console_type': u'novnc', u'host': u'liberty', u'token': u'3dfcd011-28f1-4cf3-8f5c-8cd18de4560e', \
u'access_url': u'http://192.168.200.208:6080/vnc_auto.html?token=3dfcd011-28f1-4cf3-8f5c-8cd18de4560e'\
, u'port': u'5900'}
```

We can see how the python novncproxy process binds both IPs/port.

```text
# ps aux | grep vnc
nova     14840  1.2  0.7 362096 41000 ?        S    18:53   0:14 /usr/bin/python2 /usr/bin/nova-novncproxy --web /usr/share/novnc/

# netstat -putona | grep 14840
tcp        0      0 192.168.200.208:6080    192.168.200.1:59918     ESTABLISHED 14840/python2        keepalive (3,13/0/0)
tcp        0      0 192.168.122.73:57764    192.168.122.73:5900     ESTABLISHED 14840/python2        keepalive (3,13/0/0)
```

Nova-novncproxy starts the connection between the instance and user's browser session.

```text
INFO nova.console.websocketproxy [req-399c7b58-700a-4779-b215-b12d10056813 - - - - -]\
   7: connecting to: liberty:5900
```

Libvirt connects a vnc console into the instance, as we can see at the xml provided by virsh command. Also, port 5900 now is binded at qemu-kvm process.

```text
# virsh dumpxml 2
...
<graphics type='vnc' port='5900' autoport='yes' listen='0.0.0.0' keymap='en-us'>
     <listen type='address' address='0.0.0.0'/>
   </graphics>
...

# netstat -putona | grep 5900
tcp        0      0 0.0.0.0:5900            0.0.0.0:*               LISTEN      5910/qemu-kvm        off (0.00/0/0)
tcp        0      0 192.168.122.73:5900     192.168.122.73:57702    ESTABLISHED 5910/qemu-kvm        off (0.00/0/0)
tcp        0      0 192.168.122.73:57702    192.168.122.73:5900     ESTABLISHED 11118/python2        keepalive (1,92/0/0)
```

Nova-novncproxy keeps the connection alive until browser session ends.

```text
DEBUG nova.console.websocketproxy [-] \
Reaing zombies, active child count is 1 vmsg /usr/lib/python2.7/site-packages/websockify/websocket.py:828
```

When a token is not valid while authenticating against nova-consoleauth, we can see a message like the following.

```text
INFO nova.console.websocketproxy [req-9164b32d-3ce1-441b-82c7-6c23c9a354d0 - - - - -] \
handler exception: The token '3dfcd011-28f1-4cf3-8f5c-8cd18de4560e' is invalid or has expired
```

Regards. Eduardo Gonzalez

