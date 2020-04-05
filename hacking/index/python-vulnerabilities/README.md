# Python Vulnerabilities

This section will explain what vulnerabilities are common in python developments, how to exploit and fix them.

{% page-ref page="data-deserialization/" %}



## Vulnerable server

For the purpose of learning all this vulnerabilities I've create a vulnerable server that will allow practice and execution of the provided examples, but is better if try to break by yourself googling for information about the technologies or the bugs.

To run the service on port http://localhost:5000

```text
docker run --name vuln_server \
           --rm -ti -p 80:5000 \
           egonzalez90/vuln_python_server:latest
```

Once in the main page, browse through the menu and try to get the flag at `/root/flag` exploiting the different techniques.

Other command and files can also be executed, just be careful if mount other volumes in the container.

Server source code: [https://github.com/xNaaro/vulnerable\_python](https://github.com/xNaaro/vulnerable_python)

Happy hacking and fixing vulnerabilities ;\)



