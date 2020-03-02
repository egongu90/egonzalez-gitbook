# Rally OpenStack benchmarking with Docker

OpenStack Rally is a project under the Big Tent umbrella with the mission of verify OpenStack environments to ensure SLAs under high loads or fail over scenarios, and cloud services verification. Rally can also be used to continuous integration and delivery tasks.

Why use Rally inside a Docker container? Rally is a service that is not commonly used in most environments, is a tool that is used when new infrastructure changes are made or when a SLAs review must be done, not make any sense have a service consuming infrastructure resources or block a server only for use under specific situations. Also, if your OpenStack infrastructure is automated, with a container you can have a nice integration with CI/CD tools like Jenkins.

Main reasons to use Rally inside Docker containers:

* Quick tests/deployments of Rally tasks
* Automated testing
* Cost savings
* Operators can execute tasks with their own computers, freeing

  infrastructure resources

* Re-utilization of resources

Here you got my suggestions about how to use Rally inside Docker:

* Create a new container\(automatized or not by another tool\)
* Always use an external volume to store rally reports data
* Execute Rally tasks
* Export the reports to the volume shared with the Docker host
* Kill the container

\| Let\'s start with this quick guide: \| Clone the repo i created with the Dockerfile

```text
[egonzalez@localhost ~]$ git clone https://github.com/egonzalez90/docker-rally.git
```

Move to docker-rally directory

```text
[egonzalez@localhost ~]$ cd docker-rally/
```

Create the Docker image

```text
[egonzalez@localhost docker-rally]$ docker build -t egonzalez90/rally-mitaka .
Sending build context to Docker daemon  76.8 kB
Step 1 : FROM centos:7
 ---> 904d6c400333
Step 2 : MAINTAINER Eduardo Gonzalez Gutierrez <dabarren@gmail.com>
 ---> Using cache
 ---> ee93bc7747e1
Step 3 : RUN yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-mitaka/rdo-release-mitaka-3.noarch.rpm
 ---> Using cache
 ---> 8492ab9ee261
Step 4 : RUN yum update -y
 ---> Using cache
 ---> 1374340eb39a
Step 5 : RUN yum -y install         openstack-rally         gcc         libffi-devel         python-devel         openssl-devel         gmp-devel         libxml2-devel         libxslt-devel         postgresql-devel         redhat-rpm-config         wget         openstack-selinux         openstack-utils &&         yum clean all
 ---> Using cache
 ---> 9b65e4a281be
Step 6 : RUN rally-manage --config-file /etc/rally/rally.conf db recreate
 ---> Using cache
 ---> dc4f3dbc1505
Successfully built dc4f3dbc1505
```

Start rally container with a pseudo-tty and a volume to store rally execution data

```text
[egonzalez@localhost docker-rally]$ docker run -ti -v /opt/rally-data/:/rally-data:Z egonzalez90/rally-mitaka
[root@07766ba700e8 /]# 
```

Create a file called deploy.json with the admin info of your OpenStack environment

```text
[root@07766ba700e8 /]# vi deploy.json

{
    "type": "ExistingCloud",
    "auth_url": "http://controller:5000/v2.0",
    "region_name": "RegionOne",
    "admin": {
        "username": "admin",
        "password": "my_password",
        "tenant_name": "admin"
    }
}
```

Create a deployment with the json we previously created

```text
[root@07766ba700e8 /]# rally deployment create --file=deploy.json --name=existing
2016-06-15 09:42:25.428 25 INFO rally.deployment.engine [-] Deployment a5162111-02a5-458f-bb59-f822cab1aa93 | Starting:  OpenStack cloud deployment.
2016-06-15 09:42:25.478 25 INFO rally.deployment.engine [-] Deployment a5162111-02a5-458f-bb59-f822cab1aa93 | Completed: OpenStack cloud deployment.
+--------------------------------------+----------------------------+----------+------------------+--------+
| uuid                                 | created_at                 | name     | status           | active |
+--------------------------------------+----------------------------+----------+------------------+--------+
| a5162111-02a5-458f-bb59-f822cab1aa93 | 2016-06-15 09:42:25.391691 | existing | deploy->finished |        |
+--------------------------------------+----------------------------+----------+------------------+--------+
Using deployment: a5162111-02a5-458f-bb59-f822cab1aa93
~/.rally/openrc was updated

HINTS:
* To get your cloud resources, run:
        rally show [flavors|images|keypairs|networks|secgroups]

* To use standard OpenStack clients, set up your env by running:
        source ~/.rally/openrc
  OpenStack clients are now configured, e.g run:
        glance image-list
```

Source the openrc file rally has created with your user info and test if you can connect with glance

```text
[root@07766ba700e8 /]# source ~/.rally/openrc

[root@07766ba700e8 /]# glance  image-list
+--------------------------------------+--------+
| ID                                   | Name   |
+--------------------------------------+--------+
| 1c4fc8a6-3ea7-433c-8ece-a14bbaf861e2 | cirros |
+--------------------------------------+--------+
```

Check deployment status

```text
[root@07766ba700e8 /]# rally deployment check
keystone endpoints are valid and following services are available:
+-------------+----------------+-----------+
| services    | type           | status    |
+-------------+----------------+-----------+
| __unknown__ | volumev2       | Available |
| ceilometer  | metering       | Available |
| cinder      | volume         | Available |
| cloud       | cloudformation | Available |
| glance      | image          | Available |
| heat        | orchestration  | Available |
| keystone    | identity       | Available |
| neutron     | network        | Available |
| nova        | compute        | Available |
+-------------+----------------+-----------+
NOTE: '__unknown__' service name means that Keystone service catalog doesn't return name for this service and Rally can not identify service by its type. BUT you still can use such services with api_versions context, specifying type of service (execute `rally plugin show api_versions` for more details).
```

Create a test execution file, this test will check if nova can boot and delete some instances

```text
[root@07766ba700e8 /]# vi execution.json

{
  "NovaServers.boot_and_delete_server": [
    {
      "runner": {
        "type": "constant", 
        "concurrency": 2, 
        "times": 10
      }, 
      "args": {
        "force_delete": false, 
        "flavor": {
          "name": "m1.tiny"
        }, 
        "image": {
          "name": "cirros"
        }
      }, 
      "context": {
        "users": {
          "project_domain": "default", 
          "users_per_tenant": 2, 
          "tenants": 3, 
          "resource_management_workers": 30, 
          "user_domain": "default"
        }
      }
    }
  ]
}
```

Run the task with the following command

```text
[root@07766ba700e8 /]# rally task start execution.json
--------------------------------------------------------------------------------
 Preparing input task
--------------------------------------------------------------------------------

Input task is:
{
    "NovaServers.boot_and_delete_server": [
        {
            "args": {
                "flavor": {
                    "name": "m1.tiny"
                },
                "image": {
                    "name": "cirros"
                },
                "force_delete": false
            },
            "runner": {
                "type": "constant",
                "times": 10,
                "concurrency": 2
            },
            "context": {
                "users": {
                    "tenants": 3,
                    "users_per_tenant": 2
                }
            }
        }
    ]
}

Task syntax is correct :)
2016-06-15 09:48:11.556 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Starting:  Task validation.
2016-06-15 09:48:11.579 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Starting:  Task validation of scenarios names.
2016-06-15 09:48:11.581 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Completed: Task validation of scenarios names.
2016-06-15 09:48:11.581 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Starting:  Task validation of syntax.
2016-06-15 09:48:11.587 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Completed: Task validation of syntax.
2016-06-15 09:48:11.588 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Starting:  Task validation of semantic.
2016-06-15 09:48:11.588 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Starting:  Task validation check cloud.
2016-06-15 09:48:11.694 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Completed: Task validation check cloud.
2016-06-15 09:48:11.700 101 INFO rally.plugins.openstack.context.keystone.users [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Starting:  Enter context: `users`
2016-06-15 09:48:12.004 101 INFO rally.plugins.openstack.context.keystone.users [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Completed: Enter context: `users`
2016-06-15 09:48:12.106 101 WARNING rally.task.types [-] FlavorResourceType is deprecated in Rally v0.3.2; use the equivalent resource plugin name instead
2016-06-15 09:48:12.207 101 WARNING rally.task.types [-] ImageResourceType is deprecated in Rally v0.3.2; use the equivalent resource plugin name instead
2016-06-15 09:48:12.395 101 INFO rally.plugins.openstack.context.keystone.users [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Starting:  Exit context: `users`
2016-06-15 09:48:13.546 101 INFO rally.plugins.openstack.context.keystone.users [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Completed: Exit context: `users`
2016-06-15 09:48:13.546 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Completed: Task validation of semantic.
2016-06-15 09:48:13.547 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Completed: Task validation.
Task config is valid :)
--------------------------------------------------------------------------------
 Task  137eb997-d1f8-4d3f-918a-8aec3db7500f: started
--------------------------------------------------------------------------------

Benchmarking... This can take a while...

To track task status use:

        rally task status
        or
        rally task detailed

Using task: 137eb997-d1f8-4d3f-918a-8aec3db7500f
2016-06-15 09:48:13.555 101 INFO rally.api [-] Benchmark Task 137eb997-d1f8-4d3f-918a-8aec3db7500f on Deployment a5162111-02a5-458f-bb59-f822cab1aa93
2016-06-15 09:48:13.558 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Starting:  Benchmarking.
2016-06-15 09:48:13.586 101 INFO rally.task.engine [-] Running benchmark with key:
{
  "kw": {
    "runner": {
      "type": "constant",
      "concurrency": 2,
      "times": 10
    },
    "args": {
      "force_delete": false,
      "flavor": {
        "name": "m1.tiny"
      },
      "image": {
        "name": "cirros"
      }
    },
    "context": {
      "users": {
        "users_per_tenant": 2,
        "tenants": 3
      }
    }
  },
  "name": "NovaServers.boot_and_delete_server",
  "pos": 0
}
2016-06-15 09:48:13.592 101 INFO rally.plugins.openstack.context.keystone.users [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Starting:  Enter context: `users`
2016-06-15 09:48:14.994 101 INFO rally.plugins.openstack.context.keystone.users [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Completed: Enter context: `users`
2016-06-15 09:48:15.244 292 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 0 START
2016-06-15 09:48:15.245 293 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 1 START
2016-06-15 09:48:16.975 292 WARNING rally.common.logging [-] 'wait_for' is deprecated in Rally v0.1.2: Use wait_for_status instead.
2016-06-15 09:48:17.095 293 WARNING rally.common.logging [-] 'wait_for' is deprecated in Rally v0.1.2: Use wait_for_status instead.
2016-06-15 09:49:21.024 292 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 0 END: OK
2016-06-15 09:49:21.028 292 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 2 START
2016-06-15 09:49:32.109 293 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 1 END: OK
2016-06-15 09:49:32.112 293 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 3 START
2016-06-15 09:49:41.504 292 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 2 END: OK
2016-06-15 09:49:41.508 292 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 4 START
2016-06-15 09:49:52.455 293 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 3 END: OK
2016-06-15 09:49:52.462 293 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 5 START
2016-06-15 09:50:01.907 292 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 4 END: OK
2016-06-15 09:50:01.918 292 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 6 START
2016-06-15 09:50:12.692 293 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 5 END: OK
2016-06-15 09:50:12.694 293 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 7 START
2016-06-15 09:50:23.122 292 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 6 END: OK
2016-06-15 09:50:23.131 292 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 8 START
2016-06-15 09:50:33.322 293 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 7 END: OK
2016-06-15 09:50:33.332 293 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 9 START
2016-06-15 09:50:43.285 292 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 8 END: OK
2016-06-15 09:50:53.422 293 INFO rally.task.runner [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | ITER: 9 END: OK
2016-06-15 09:50:53.436 101 INFO rally.plugins.openstack.context.cleanup.user [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Starting:  user resources cleanup
2016-06-15 09:50:55.244 101 INFO rally.plugins.openstack.context.cleanup.user [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Completed: user resources cleanup
2016-06-15 09:50:55.245 101 INFO rally.plugins.openstack.context.keystone.users [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Starting:  Exit context: `users`
2016-06-15 09:50:57.438 101 INFO rally.plugins.openstack.context.keystone.users [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Completed: Exit context: `users`
2016-06-15 09:50:58.023 101 INFO rally.task.engine [-] Task 137eb997-d1f8-4d3f-918a-8aec3db7500f | Completed: Benchmarking.

--------------------------------------------------------------------------------
Task 137eb997-d1f8-4d3f-918a-8aec3db7500f: finished
--------------------------------------------------------------------------------

test scenario NovaServers.boot_and_delete_server
args position 0
args values:
{
  "runner": {
    "type": "constant",
    "concurrency": 2,
    "times": 10
  },
  "args": {
    "force_delete": false,
    "flavor": {
      "name": "m1.tiny"
    },
    "image": {
      "name": "cirros"
    }
  },
  "context": {
    "users": {
      "project_domain": "default",
      "users_per_tenant": 2,
      "tenants": 3,
      "resource_management_workers": 30,
      "user_domain": "default"
    }
  }
}

+-----------------------------------------------------------------------------------------------------------------------+
|                                                 Response Times (sec)                                                  |
+--------------------+-----------+--------------+--------------+--------------+-----------+-----------+---------+-------+
| Action             | Min (sec) | Median (sec) | 90%ile (sec) | 95%ile (sec) | Max (sec) | Avg (sec) | Success | Count |
+--------------------+-----------+--------------+--------------+--------------+-----------+-----------+---------+-------+
| nova.boot_server   | 17.84     | 18.158       | 64.433       | 69.419       | 74.405    | 28.299    | 100.0%  | 10    |
| nova.delete_server | 2.24      | 2.275        | 2.454        | 2.456        | 2.458     | 2.317     | 100.0%  | 10    |
| total              | 20.09     | 20.437       | 66.888       | 71.875       | 76.863    | 30.616    | 100.0%  | 10    |
+--------------------+-----------+--------------+--------------+--------------+-----------+-----------+---------+-------+

Load duration: 158.199862003
Full duration: 163.846753836

HINTS:
* To plot HTML graphics with this data, run:
        rally task report 137eb997-d1f8-4d3f-918a-8aec3db7500f --out output.html

* To generate a JUnit report, run:
        rally task report 137eb997-d1f8-4d3f-918a-8aec3db7500f --junit --out output.xml

* To get raw JSON output of task results, run:
        rally task results 137eb997-d1f8-4d3f-918a-8aec3db7500f
```

\| After a while, you will receive an output execution resume, you can export to a report file with the following command in a pretty style report. \| Use the volume we created with the Docker Host to save report files.

```text
[root@07766ba700e8 /]# rally task report 137eb997-d1f8-4d3f-918a-8aec3db7500f --html-static --out /rally-data/output.html
```

\| Open the output file form a Web browser and review the report.

\| Regards

