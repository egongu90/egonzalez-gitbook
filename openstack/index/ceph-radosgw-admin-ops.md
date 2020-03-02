---
description: How to use it
---

# Ceph RadosGW admin Ops

Using RadosGW admin ops for the first time, can be a real headache , for this purpose i have made this post, where you will understand how to use this API.

Let's start:

For issue a request through admin ops, you need to have a signature, this signature is make it signing a header. The header must to be composed by the current date, the request type\(GET/PUT/POST/DELETE\) and the request itself. This header must be signed by SSL including the admin ops secret on this signature. 

Now , you can make a request. 

Sometimes, the time is not the same as the radosgw node expect, you can hack on it changing the date=$\(date\) value with: 

If your host has two hours more than the radosgw node, substract this two hours under $\(\( 10\#$i-2\)\) variable, where 2 is the two hours to substract.

```text
date=$(for i in $(date "+%H") ; do date "+%a, %d %b %Y $(( 10#$i-2 )):%M:%S +0000" ; done)
```

Examples:

Create a user named egonzalez

```text
#!/bin/bash
token=U2JCD4ZG4D1XJOI5XNF4 ## USER_TOKEN
secret=+IFgr7POzLWS0i3hQnC+dd3DOAZObHoY5NYm6m3b ## USER_SECRET
query=$1
name=$2
query3="&uid="
query2=admin/user
query4="&quota-type=user"
date=$(date)
header="PUT\n\n\n${date}\n/${query2}"
sig=$(echo -en ${header} | openssl sha1 -hmac ${secret} -binary | base64)
curl -v -H "Date: ${date}" -H "Authorization: AWS ${token}:${sig}" -L -X PUT "http://10.0.2.10/${query2}?format=json${query3}${query}&display-name=${name}" -H "Host: 10.0.2.10"
##Change IPs with your own IPs
```

See quotas

```text
   #!/bin/bash
   token=U2JCD4ZG4D1XJOI5XNF4 ## USER_TOKEN
   secret=+IFgr7POzLWS0i3hQnC+dd3DOAZObHoY5NYm6m3b ## USER_SECRET
   query=$1
   query3="&uid="
   query2=admin/user
   query4="&quota-type=user"
   date=$(date)
   header="GET\n\n\n${date}\n/${query2}"
   sig=$(echo -en ${header} | openssl sha1 -hmac ${secret} -binary | base64)
   curl -v -H "Date: ${date}" -H "Authorization: AWS ${token}:${sig}" -L -X GET "http://10.0.2.10/${query2}?quota${query3}${query}&quota-type=user" -H "Host: 10.0.2.10"
   ##Change IPs with your own IPs
```

See egonzalez user information

```text
#!/bin/bash
token=U2JCD4ZG4D1XJOI5XNF4 ## USER_TOKEN
secret=+IFgr7POzLWS0i3hQnC+dd3DOAZObHoY5NYm6m3b ## USER_SECRET
query=$1
query3="&uid="
query2=admin/user
date=$(date)
header="GET\n\n\n${date}\n/${query2}"
sig=$(echo -en ${header} | openssl sha1 -hmac ${secret} -binary | base64)
curl -v -H "Date: ${date}" -H "Authorization: AWS ${token}:${sig}" -L -X GET "http://10.0.2.10/${query2}?format=json${query3}${query}" -H "Host: 10.0.2.10"
##Change IPs with your own IPs
```

When you really understand how admin ops works, is not as difficult to use it, just search at the official documentation and modify the desired values.

I hope this helps:

Regards, Eduardo.

