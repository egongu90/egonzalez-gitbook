---
description: 'Here is a simple script to list all tenants belonging an user:'
---

# List all tenants belonging an user

```text
#!/bin/bash
   echo -n "Username : " ; read usercheck
   for userid in $(keystone user-list | grep -w $usercheck | awk '{print$2}')
       do
       for tenant in $(keystone tenant-list | awk 'NR>3 && /^|/ {print$2}')
       do
           for tenantid in $(keystone user-role-list --user $userid --tenant $tenant | awk 'NR>3 && /^|/ {print$8}')
           do
               keystone tenant-list | grep $tenantid | awk '{print$4}'
           done
       done
   done
```

Also you can run all in a simple cmd line

```text
echo -n "user name "; read usercheck; for userid in $(keystone user-list | grep $usercheck | awk '{print$2}'); do echo $userid | for tenant in $(keystone tenant-list | awk 'NR>3 && /^|/ {print$2}'); do echo $tenant | for tenantid in $(keystone user-role-list --user $userid --tenant $tenant | awk 'NR>3 && /^|/ {print$8}'); do keystone tenant-list | grep $tenantid | awk '{print$4}'; done ; done ; done
```

If you are a developer, probably you need to list all tenants in a HTTP request, for this purpose you can use the REST API to the port 5000 of keystone

```text
curl -i -X GET http://KEYSTONEIP:5000/v2.0/tenants -H "User-Agent: python-keystoneclient" -H "X-Auth-Token: USERTOKEN"
```

I have saved the user token in a OS\_VARIABLE called OS\_TOKEN, if you don't do that, you should input all the token in the HTTP request.

```text
curl -i -X GET http://192.168.1.11:5000/v2.0/tenants -H "User-Agent: python-keystoneclient" -H "X-Auth-Token: $OS_TOKEN"
```

