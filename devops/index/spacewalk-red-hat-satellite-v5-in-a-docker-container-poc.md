# Spacewalk Red Hat Satellite v5 in a Docker container PoC

Spacewalk was the upstream project to provide a Linux systems management layer on which Red Hat Satellite was based, was based at least until RH Satellite version 5. Newer versions are not anymore based on Spacewalk, instead Satellite is a federation of several upstream open source projects, including Katello, Foreman, Pulp, and Candlepin.

Some weeks ago, a friend asked me if I knew a Docker container image for Satellite. \| I have not found any image. What I found was some Spacewalk images, but sadly none of them worked for me. \| I decided to create an image for this purpose.

While developing the image, I found serious troubles to make it run with systemd \(I\'m a fan of systemd, but not inside containers yet\). \| The result was a semi functional working image. I said semi functional because some Spacewalk features are not working \(probably an issue with systemd again\). \| The main problem was that spacewalk-setup script starts and uses systemd to configure the database and the other needed services, that\'s OK in a VM but not in a container. \| So i needed to hack into postgres setup and start the services with the typical `command --config-file file.conf` executed from supervisord as Docker entrypoint. Currently there is an issue with `osa-dispatcher`, on which I can\'t find a fix to make it run.

This image is primarily created just for test Spacewalk interface and be more comfortable with it aka testing/development purposes, or just to have fun hacking with Docker containers.

Now, I\'m going to make a short description of what the Dockerfile makes and then start the container. \| Have fun.

I used centos as image base for this PoC

```text
FROM centos:7 
```

Typical Maintainer line

```text
MAINTAINER Eduardo Gonzalez Gutierrez 
```

Add jpackage repo which provides Java packages for Linux

```text
COPY jpackage-generic.repo /etc/yum.repos.d/jpackage-generic.repo
```

Install EPEL and Spacewalk repositories, after install, clean all stored cache to minimize image size

```text
RUN yum install -y http://yum.spacewalkproject.org/2.5/RHEL/7/x86_64/spacewalk-repo-2.5-3.el7.noarch.rpm \
        epel-release && \
        yum clean all
```

Import Keys to allow installation from these repositories

```text
RUN rpm --import http://www.jpackage.org/jpackage.asc && \
    rpm --import https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-7 && \
    rpm --import http://yum.spacewalkproject.org/RPM-GPG-KEY-spacewalk-2015 && \
    yum clean all
```

Install spacewalk and supervisord packages

```text
RUN yum -y install \
        spacewalk-setup-postgresql \
        spacewalk-postgresql \
        supervisor  \
        yum clean all
```

Copy the example file used to sync spacewalk database in a later step

```text
COPY answerfile.txt /tmp/answerfile.txt
```

Open necessary ports

```text
EXPOSE 80 443 5222 68 69
```

Change to postgres user

```text
USER postgres
```

Initialize the database

```text
RUN /usr/bin/pg_ctl initdb  -D /var/lib/pgsql/data/
```

Create spacewalk database, user, role and create pltclu language

```text
RUN /usr/bin/pg_ctl start -D /var/lib/pgsql/data/  -w -t 300 && \
     psql -c 'CREATE DATABASE spaceschema' && \
     psql -c "CREATE USER spaceuser WITH PASSWORD 'spacepw'" && \
     psql -c 'ALTER ROLE spaceuser SUPERUSER' && \
     createlang pltclu spaceschema
```

Change to root user

```text
USER root
```

Start the database and execute spacewalk configuration script

```text
RUN su -c "/usr/bin/pg_ctl start -D /var/lib/pgsql/data/  -w -t 300" postgres && \
    su -c "spacewalk-setup --answer-file=/tmp/answerfile.txt --skip-db-diskspace-check --skip-db-install" root ; exit 0
```

Copy supervisord configuration

```text
ADD supervisord.conf /etc/supervisord.d/supervisord.conf
```

Use supervisord command to start all services at container launch time

```text
ENTRYPOINT supervisord -c /etc/supervisord.d/supervisord.conf
```

You can check or download the source code at GitHub [https://github.com/egonzalez90/docker-spacewalk](https://github.com/egonzalez90/docker-spacewalk)

I uploaded the image to DockerHub, which is auto-build from my GitHub repository, you can find it with the following command.

```text
[egonzalez@localhost ~]$ docker search spacewalk
INDEX       NAME                                       DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
docker.io   docker.io/ruo91/spacewalk                  Spacewalk is an open source Linux systems ...   3                    [OK]
docker.io   docker.io/jamesnetherton/spacewalk         Spacewalk running under Docker                  1                    
docker.io   docker.io/coffmant/spacewalk-docker        Spacewalk                                       0                    [OK]
docker.io   docker.io/csabakollar/spacewalk            Spacewalk 2.4 in a CentOS6 container            0                    
docker.io   docker.io/egonzalez90/spacewalk            Spacewalk docker image                          0                    [OK]
docker.io   docker.io/jdostal/spacewalk-clients        Repository containing spacewalk-clients         0                    
docker.io   docker.io/jhutar/spacewalk-client                                                          0                    
docker.io   docker.io/norus/spacewalk-reposync                                                         0                    
docker.io   docker.io/pajinek/spacewalk-client                                                         0                    [OK]
docker.io   docker.io/perfectweb/spacewalk             spacewalk                                       0                    [OK]
docker.io   docker.io/researchiteng/docker-spacewalk   spacewalk is the open source version of Re...   0                    [OK]
docker.io   docker.io/varhoo/spacewalk-proxy                                                           0                    [OK]
```

To start the container use the following command. If you don\'t have the image locally, it will download the image from DockerHub

```text
[egonzalez@localhost ~]$ docker run -d --privileged=True egonzalez90/spacewalk
Unable to find image 'egonzalez90/spacewalk:latest' locally
Trying to pull repository docker.io/egonzalez90/spacewalk ... 
latest: Pulling from docker.io/egonzalez90/spacewalk
a3ed95caeb02: Already exists 
da71393503ec: Already exists 
519093688e2c: Pull complete 
97bbffaa9fc9: Pull complete 
63bfb115f62d: Pull complete 
929bbb68aff9: Pull complete 
532bc4af8e1a: Pull complete 
3eb667dda9ee: Pull complete 
275894897aa4: Pull complete 
93bcddf9cedb: Pull complete 
266c3b70754f: Pull complete 
Digest: sha256:a4dd98548f9dbb405fb4c6bb4a2a07b83d5f2bf730f29f71913b72876b1a61ab
Status: Downloaded newer image for docker.io/egonzalez90/spacewalk:latest
ded4a8b7eb1ee61fecc8ddc2eb1b092917a361bc36f7f752b32d76e79501d70a
```

Now you have the container running, check if all the ports are properly exposed

```text
[egonzalez@localhost ~]$ docker ps --latest --format 'table {{.ID}}\t{{.Image}}\t{{.Ports}}'
CONTAINER ID        IMAGE                   PORTS
ded4a8b7eb1e        egonzalez90/spacewalk   68-69/tcp, 80/tcp, 443/tcp, 5222/tcp
```

Get the container IP address in order to enter from a Web Browser

```text
[egonzalez@localhost ~]$ docker inspect ded4a8b7eb1e | egrep IPAddress
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.3",
                    "IPAddress": "172.17.0.3",
```

\| Open A browser and go to the container IP address, if you use HTTP, by default it will redirect you to HTTPS. \| The container uses an auto-signed SSL certificate, you have to add an exception in the Browser you use to allow connections to Spacewalk. \| Once in the Welcome page, create an Organization.

\| Now you are in Spacewalk and can play/test some features.

\| There is an issue I was not able to fix, so osa-dispatcher and some other features will not work with this image. \| If someone can give me an input to fix the issue it will appreciated.

```text
[egonzalez@localhost ~]$ docker logs ded4a8b7eb1e | egrep FATAL
2016-07-12 18:13:32,220 INFO gave up: osa-dispatcher entered FATAL state, too many start retries too quickly
```

Thanks for your time and hopes this image at least serves you to learn and play with the interface.

Regards, Eduardo Gonzalez

