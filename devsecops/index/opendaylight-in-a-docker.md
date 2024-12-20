# OpenDaylight in a Docker

This is a quick guide to start a Docker container with OpenDaylight running on it.

Clone OpenDaylight integration repository

```
[egonzalez@localhost]$ git clone https://github.com/opendaylight/integration.git
```

Move to the directory where CentOS Dockerfile is saved.

```
[egonzalez@localhost]$ cd integration/packaging/docker/centos/
```

\| Build the new image, you can call it as your DockerHub name(in my case egonzalez90), so you can push it there later. | If you don\\'t want to create a new image, you can use my image. This step will download and start the new container: `docker run -d egonzalez90/opendaylight`

```
[egonzalez@localhost centos]$ docker build -t egonzalez90/opendaylight .

Sending build context to Docker daemon  7.68 kB
Step 1 : FROM centos:7
Trying to pull repository docker.io/library/centos ... 7: Pulling from library/centos
1544084fad81: Pull complete 
df0fc3863fbc: Pull complete 
a3d54b467fad: Pull complete 
a65193109361: Pull complete 
Digest: sha256:a9237ff42b09cc6f610bab60a36df913ef326178a92f3b61631331867178f982
Status: Downloaded newer image for docker.io/centos:7

 ---> a65193109361
Step 2 : MAINTAINER OpenDaylight Project <info@opendaylight.org>
 ---> Running in d3f98f949b11
 ---> 81a1bad2e3a7
Removing intermediate container d3f98f949b11
Step 3 : ADD opendaylight-3-candidate.repo /etc/yum.repos.d/
 ---> 069a9c60878e
Removing intermediate container b9afb18311f3
Step 4 : RUN yum update -y && yum install -y opendaylight
 ---> Running in 559b3970235d

[[[ PACKAGE INSTALLATION STUFF ]]]                                      

Complete!
 ---> 4003e5874b03
Removing intermediate container 559b3970235d
Step 5 : EXPOSE 162 179 1088 1790 1830 2400 2550 2551 2552 4189 4342 5005 5666 6633 6640 6653 7800 8000 8080 8101 8181 8383 12001
 ---> Running in 7defebe8b7e2
 ---> 9668a559bdac
Removing intermediate container 7defebe8b7e2
Step 6 : WORKDIR /opt/opendaylight
 ---> Running in 9298a116dd14
 ---> 5bf42f56e282
Removing intermediate container 9298a116dd14
Step 7 : CMD ./bin/karaf server
 ---> Running in e0a218941b15
 ---> c1a0db72dbbc
Removing intermediate container e0a218941b15
Successfully built c1a0db72dbbc
```

Once the image is built or downloaded, ensure you have it locally

```
[egonzalez@localhost]$ docker images | grep opendaylight
egonzalez90/opendaylight                              latest              c1a0db72dbbc        About a minute ago   740.6 MB
```

Start a new container in a detached mode.

```
[egonzalez@localhost]$ docker run -d egonzalez90/opendaylight
ae08898ba6adc30df012513dc6eac54943d9de8c8059e73ade185757fe684c6a
Usage of loopback devices is strongly discouraged for production use. Either use `--storage-opt dm.thinpooldev` or use `--storage-opt dm.no_warn_on_loop_devices=true` to suppress this warning.
```

Check if the container is running with:

```
[egonzalez@localhost]$ docker ps | grep opendaylight 
ae08898ba6ad        egonzalez90/opendaylight   "./bin/karaf server"     14 seconds ago      Up 11 seconds       162/tcp, 179/tcp, 1088/tcp, 1790/tcp, 1830/tcp, 2400/tcp, 2550-2552/tcp, 4189/tcp, 4342/tcp, 5005/tcp, 5666/tcp, 6633/tcp, 6640/tcp, 6653/tcp, 7800/tcp, 8000/tcp, 8080/tcp, 8101/tcp, 8181/tcp, 8383/tcp, 12001/tcp   awesome_khorana
```

Now, check container information with docker inspect, we search for the IP address

```
[egonzalez@localhost]$ docker inspect  ae08898ba6ad | grep -i IPAddress
        "SecondaryIPAddresses": null,
        "IPAddress": "172.17.0.3",
                "IPAddress": "172.17.0.3",
```

\| Now you know the container IP address, to login into karaf, first we need to download and install karaf client tool | Go to the following URL to download the package: [http://www.apache.org/dyn/closer.lua/karaf/4.0.5/apache-karaf-4.0.5.tar.gz](http://www.apache.org/dyn/closer.lua/karaf/4.0.5/apache-karaf-4.0.5.tar.gz)

Extract the files and move to the new directory

```
[egonzalez@localhost Downloads]$ tar -xzvf apache-karaf-4.0.5.tar.gz 
[egonzalez@localhost Downloads]$ cd apache-karaf-4.0.5/
```

Execute the client authenticating with the container IP

```
[egonzalez@localhost apache-karaf-4.0.5]$ ./bin/client -a 8101 -h 172.17.0.3 -u karaf -v
client: JAVA_HOME not set; results may vary
13 [main] INFO org.apache.sshd.common.util.SecurityUtils - BouncyCastle not registered, using the default JCE provider
Logging in as karaf
194 [sshd-SshClient[12bb4df8]-nio2-thread-1] INFO org.apache.sshd.client.session.ClientSessionImpl - Client session created
203 [main] INFO org.apache.sshd.client.session.ClientSessionImpl - Start flagging packets as pending until key exchange is done
204 [sshd-SshClient[12bb4df8]-nio2-thread-1] INFO org.apache.sshd.client.session.ClientSessionImpl - Server version string: SSH-2.0-SSHD-CORE-0.12.0
321 [sshd-SshClient[12bb4df8]-nio2-thread-3] WARN org.apache.sshd.client.keyverifier.AcceptAllServerKeyVerifier - Server at /172.17.0.3:8101 presented unverified DSA key: 09:a0:45:95:7a:dd:94:7c:6b:c3:f9:c0:23:88:1d:b0
324 [sshd-SshClient[12bb4df8]-nio2-thread-3] INFO org.apache.sshd.client.session.ClientSessionImpl - Dequeing pending packets
327 [sshd-SshClient[12bb4df8]-nio2-thread-4] INFO org.apache.sshd.client.session.ClientUserAuthServiceNew - Received SSH_MSG_USERAUTH_FAILURE
338 [sshd-SshClient[12bb4df8]-nio2-thread-5] INFO org.apache.sshd.client.session.ClientUserAuthServiceNew - Received SSH_MSG_USERAUTH_FAILURE
341 [sshd-SshClient[12bb4df8]-nio2-thread-6] INFO org.apache.sshd.client.auth.UserAuthKeyboardInteractive - Received Password authentication  en-US
344 [sshd-SshClient[12bb4df8]-nio2-thread-7] INFO org.apache.sshd.client.session.ClientUserAuthServiceNew - Received SSH_MSG_USERAUTH_SUCCESS

    ________                       ________                .__  .__       .__     __       
    \_____  \ ______   ____   ____ \______ \ _____  ___.__.|  | |__| ____ |  |___/  |_     
     /   |   \\____ \_/ __ \ /    \ |    |  \\__  \< | || | | |/ ___\| | \ __\ / | \ |_> >  ___/|   |  \|    `   \/ __ \\___  ||  |_|  / /_/  >   Y  \  |      
    \_______  /   __/ \___  >___|  /_______  (____  / ____||____/__\___  /|___|  /__|      
            \/|__|        \/     \/        \/     \/\/            /_____/      \/          


Hit '' for a list of available commands
and '[cmd] --help' for help on a specific command.
Hit '' or type 'system:shutdown' or 'logout' to shutdown OpenDaylight.
```

Once karaf login succeed, install a few features like DLUX

```
opendaylight-user@root>feature:install odl-restconf odl-l2switch-switch odl-mdsal-apidocs odl-dlux-core
```

Now you can login at the container IP with admin as username and password.

```
http://172.17.0.3:8181/index.html
```

Best regards
